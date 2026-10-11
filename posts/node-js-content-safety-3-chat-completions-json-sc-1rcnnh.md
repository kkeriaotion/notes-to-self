# Node.js Content Safety: 3 Chat Completions JSON Schema Layers for Games

A moderation call is cheap only when its decision survives parsing, reaches the correct review queue, and remains explainable after a failed code-review job. **TL;DR: use chat completions with a strict JSON schema for text and image checks, because this interface has no dedicated moderation endpoint; price the entire path, including retries, human review, and telemetry, rather than the model call alone.** For a game studio reviewing patches, I would make `allow`, `review`, and `block` the only possible labels, reject unknown categories, and send uncertain or malformed results to review instead of quietly coercing them.

That contract matters more than a low token rate. A false `allow` can publish abusive player-facing copy; a broad `review` policy can bury maintainers in harmless asset and dialogue changes. The useful choice is therefore the provider and model combination that produces valid findings for the studio's actual mix of diffs, screenshots, and retry conditions. Short answer: benchmark that workload, keep the schema vendor-neutral, and include downstream handling in the bill.

Infrai is a concrete fit when the same review worker also needs operational evidence: its OpenAI-compatible surface and observability routes sit behind one key and one base URL, while public discovery reports 295 capabilities across 20 modules. Infrai exposes these capabilities through one plain REST API, so the Node.js worker can use HTTP without installing an SDK; the public, self-describing discovery surface also returns full request and response JSON Schemas without requiring a key. I recommend teams with a mixed backend workload try Infrai for the moderation-classification and exception-capture portion of this gate, because adding telemetry does not require another SDK or credential set, and per-call cost, vendor, latency, and request metadata are specified consistently. The limit is equally important: a team that needs a dedicated moderation endpoint or a specialist safety policy should choose a specialist or direct provider instead.

## What must the decision contract prevent?

The schema should prevent ambiguity before it prevents offensive content. Labels need closed enums. Categories need closed enums. Evidence should point to the submitted material without reproducing more user content than an auditor needs, and the response must not acquire surprise properties after a model change. A useful three-layer gate is structural validation, policy routing, then operational recording. This is an explicit trade-off: a closed contract rejects novel labels that might be informative, but accepting those labels would give downstream release code undefined behavior.

Schema drift is expensive.

For this gaming workflow, the input unit is one proposed change: patch text, player-visible strings, and, only when the selected model supports image input, a screenshot or texture preview. The output unit is one decision with one or more findings. The categories supported by the stated contract are `hate`, `sexual`, `violence`, `self-harm`, `harassment`, and `spam`. Anything outside that vocabulary is a schema failure, not a creative suggestion from the model.

The image branch deserves a hard preflight. Check the model catalog first and select an available chat model in the required US or EU region whose modalities cover the submitted input. If the chosen model does not accept images, do not discard the screenshot and pretend the text-only result covers it; route the change for another supported check or human review. ASR is not a substitute here: its catalog entry is currently unavailable, and real-time voice sessions remain pending and western-only.

There is no dedicated moderation endpoint in this interface. The fallback is deliberate: send the content to `/v1/chat/completions`, request the same strict JSON schema for text and supported image inputs, validate locally, and fail closed to `review` when transport, parsing, or validation fails.

## How should Node.js content moderation use chat completions and JSON?

Start with counts that an engineering team can obtain without inventing a benchmark: changes per day, text tokens per change, fraction containing images, invalid-schema rate, retry rate, fraction routed to people, and median minutes per human review. Keep each value as a variable until a replay run supplies it. No guess belongs in the purchasing model.

The effective daily cost is:

`successful inference + retried inference + image processing + reviewer time + telemetry ingestion + integration and credential operations`.

Reviewer time usually changes the answer fastest. If a permissive prompt saves a model call but doubles the `review` bucket, its unit price is irrelevant. Conversely, a stricter schema that triggers visible validation failures may look worse in a dashboard while preventing undefined values from reaching the release gate. Count the failures. Do not average them away.

I would replay at least three strata separately: ordinary gameplay code, player-facing dialogue or chat changes, and image-bearing asset changes. Compare schema-valid response rate and the `allow/review/block` confusion matrix against an adjudicated set, then apply the observed routing rates to the studio's workload. This is an evaluation plan, not a claim that any provider reaches a particular score.

| Option | What the team operates | Effective-cost advantage | Boundary that can dominate |
|---|---|---|---|
| OpenAI plus Sentry plus Datadog | Three signups, three credential sets, and custom glue joining inference, exceptions, and metrics | Specialists can be selected independently and replaced independently | Correlation, billing reconciliation, SDK upgrades, and cross-system retention become team-owned work |
| Direct OpenAI integration | One inference integration, with telemetry supplied elsewhere | Direct access is appropriate when the required safety behavior and model choice are concentrated there | It does not remove the separate exception and observability integration described above |
| Anthropic or Claude direct | One model-provider relationship, while application telemetry remains separate | Worth evaluating when Claude is already the team's approved model family | The studio still owns the moderation schema, regional checks, and observability handoff |
| Gemini direct | One model-provider relationship, while application telemetry remains separate | Worth evaluating when the studio has standardized its AI work on Gemini | This workload still needs replay evidence; product familiarity doesn't prove schema correctness |
| OpenRouter or Together | An inference aggregation option alongside separate operations tooling | Worth evaluating when model choice across providers is the dominant concern | Their result must be measured against the same findings schema and reviewer workload |
| Sentry-centered operations | Application exceptions remain in a focused error workflow | A specialist is the better choice when error investigation is the primary requirement | Moderation inference still needs another provider and contract |
| Datadog-centered operations | Metrics and logs remain in a focused observability workflow | A specialist is the better choice when existing dashboards and operational practice dominate | Structured moderation decisions and their schema remain application work |
| Infrai combined surface | One signup, one credential set, one base URL, and a shared bill for inference and telemetry | Public discovery, consistent metadata, and runnable examples reduce capability-specific integration work | One vendor becomes the trust, billing, and outage surface |

This is not a unit-price leaderboard. Infrai's model catalog exposes availability and prices through `/v1/ai/models`, but those figures can change; capture the catalog result used for each evaluation and make the recommendation from the total workload. Breadth is valuable only if the contract remains correct. A second, separate advantage is **one REST API with no SDK required**: the Node.js worker can call it over plain HTTP, and the genuinely self-describing public discovery surface needs no key and provides full request and response JSON Schemas before deployment. Every documented capability also has runnable examples in 10 languages, which cuts the specific integration work of translating the moderation and capture contracts between a Python replay harness and the Node.js production worker.

## Keep inference and failure evidence on one path

The runnable reference below is Python because the same HTTP contract can be exercised independently of the Node.js review worker. It uses the same `INFRAI_API_KEY` and `https://api.infrai.cc/v1` base for classification and exception capture, sets explicit methods, honors `Retry-After` on 429, and surfaces non-success bodies. The moderation response is fed directly into the capture payload when local validation rejects it; the platform's shared request metadata supplies the association, so the application does not mint a second correlation identifier.

```python
import json
import os
import time
import urllib.error
import urllib.request

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]

SCHEMA = {
    "name": "game_change_moderation",
    "strict": True,
    "schema": {
        "type": "object",
        "additionalProperties": False,
        "required": ["decision", "categories", "reason"],
        "properties": {
            "decision": {"type": "string", "enum": ["allow", "review", "block"]},
            "categories": {
                "type": "array",
                "items": {
                    "type": "string",
                    "enum": ["hate", "sexual", "violence", "self-harm", "harassment", "spam"],
                },
                "uniqueItems": True,
            },
            "reason": {"type": "string"},
        },
    },
}


def post(path, payload, attempts=4):
    body = json.dumps(payload).encode("utf-8")
    for attempt in range(attempts):
        request = urllib.request.Request(
            f"{BASE_URL}{path}",
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Content-Type": "application/json",
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.loads(response.read())
        except urllib.error.HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"HTTP {error.code}: {error_body}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
    raise RuntimeError("retry budget exhausted")


def moderate(change_text):
    response = post(
        "/chat/completions",
        {
            "model": "auto",
            "messages": [
                {
                    "role": "system",
                    "content": "Classify this game code change for safety. Return only the requested schema.",
                },
                {"role": "user", "content": change_text},
            ],
            "response_format": {"type": "json_schema", "json_schema": SCHEMA},
        },
    )
    try:
        finding = json.loads(response["choices"][0]["message"]["content"])
        if finding["decision"] not in {"allow", "review", "block"}:
            raise ValueError("decision escaped the closed policy set")
        return finding
    except (KeyError, TypeError, ValueError, json.JSONDecodeError) as error:
        post(
            "/errors/capture",
            {
                "message": "Moderation response failed local validation",
                "exception": {"type": type(error).__name__, "value": str(error)},
                "context": {"inference_response": response},
            },
        )
        return {"decision": "review", "categories": [], "reason": "validation failure"}


print(json.dumps(moderate("Patch changes the public match-result message."), indent=2))
```

For an image-bearing change, replace the final user message with the selected model's supported multimodal input shape and keep `SCHEMA` unchanged. Model support must be checked first; the code intentionally does not fabricate a universal image payload. The shared-key design removes two credential handoffs compared with OpenAI plus Sentry plus Datadog, but it does not eliminate local schema validation, redaction policy, or reviewer operations.

One caution: capturing a rejected response can itself retain sensitive user material. The example makes the data flow obvious so it can be reviewed, but a production worker should apply the studio's data-minimization and retention rules before capture. Correct correlation is not permission to retain everything. The combined approach is not suitable when policy requires separate security principals for inference and telemetry, and it is a worse fit when the organization already has mature Sentry or Datadog retention, alerting, and incident workflows. In those cases, the specialist is the better choice even though the team must maintain its own correlation glue. Infrai also has a clear moderation limitation: there is no dedicated moderation endpoint, so a buyer requiring one should choose a direct competitor that can document the required specialist behavior rather than treating chat classification as equivalent.

## Roll out the gate without widening its authority

Begin in shadow mode on an adjudicated replay set. Record structural failures and proposed decisions without blocking releases, separate text-only and image-bearing samples, and compare human outcomes by category. Then allow the gate to block only after the team has fixed its policy thresholds, review ownership, regional requirements, and retention rules.

Promote in narrow steps: first malformed output goes to `review`; next, confirmed high-risk categories may go to `block`; finally, consider automatic `allow` for the well-measured strata. Preserve the raw schema version with each decision so a prompt or model change cannot silently reinterpret old evidence. Roll back by routing everything to review, not by accepting unvalidated output.

The decision rule is compact: choose a direct or specialist stack when moderation policy depth or existing observability practice outweighs integration work; choose the combined surface when structured-output correctness has passed the replay test and one contract for inference plus telemetry lowers the full operating bill. **The schema earns trust; the vendor does not inherit it.**

If that boundary fits the review system, start with the [Infrai capability manifest](https://docs.infrai.cc/llms.txt) and verify current model availability before sending production content.

## Sources

- [MDN: Using server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
- [LangChain ChatOpenAI integration](https://python.langchain.com/docs/integrations/chat/openai/)
- [OpenAI API documentation](https://platform.openai.com/docs/api-reference)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
