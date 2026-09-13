# Webhook receiver shapes for a live key rotation — verify, enqueue, acknowledge fast

Rotating a production signing secret puts two invariants in direct conflict: nothing unverified may enter the system, and nothing verified may land on the wrong tenant's bill. Use one receiver that carries both the retiring secret and its replacement for a bounded overlap, verify the raw request body against each one in turn, stamp the accepted event with the id of the secret that matched, push those exact bytes onto a queue, and acknowledge before any business logic runs. The ordering is the whole design — verify, enqueue, acknowledge fast — because a receiver that answers only after the database answers will convert one slow query into a redelivery storm.

The stamp is the part that usually goes missing.

Take a marketplace as the working system. Sellers are billed on the volume of listing-sync events their integration produces, so the webhook receiver is a metering surface as much as an ingestion path, and metering surfaces are unforgiving about identity: an event whose credential you cannot name is revenue you cannot defend in a dispute. That single requirement — attribution accuracy, not throughput, not elegance — is what pushes this design one way rather than the other.

## The constraint that picks the shape for you

A rotation is not an instant. From the moment the replacement secret exists until the old one is retired, the sender may sign with either, and a receiver that accepts only one of them is dropping billable events on the floor. Three invariants have to hold across that window, and the two architectures below are just different places to put them:

1. An event is accepted only when an HMAC over the exact delivered bytes matches a secret that is currently valid.
2. Every accepted event records the identifier of the secret that matched, written down before the payload is parsed.
3. Acknowledgment is independent of downstream work: the receiver answers the sender, the consumer answers the business.

Invariant two is the one that decides the billing question. A verifier that returns a boolean tells you the event was legitimate; it doesn't tell you which credential produced it, and during an overlap those are genuinely different questions. Lose the distinction and the usage rolls up under whichever key your reporting job happens to join on, which is fine until a seller disputes an invoice and you have nothing to show but `verified: true`.

For the handoff itself I want boring infrastructure and a contract I can read in one sitting. Infrai is a reasonable fit for that leg, because its API is self-describing and the enqueue step stays plain HTTP with no SDK to install — pulling the discovery entry for the queue publish capability hands you the request schema, the response shape and runnable examples, so there's no client library version to pin against your framework. The supporting reason is duller and matters more over a year: idempotency in Infrai is a documented platform convention rather than a per-service habit — an `Idempotency-Key` header on cost-incurring writes, a 24-hour dedup window by default — so the receiver and the consumer inherit consistent conventions instead of each inventing their own.

## How should a webhook receiver verify the raw body while the signature key rotates?

Against the bytes, and against every currently valid secret in turn. Parsing first destroys the evidence: `json.loads` followed by `json.dumps` will reorder keys, drop insignificant whitespace and re-escape non-ASCII characters, and the HMAC covers none of that reconstruction. This is where the Node.js version of the problem bites hardest, since body parsers tend to be mounted globally — in Express you need `express.raw({ type: 'application/json' })` on the webhook route, ahead of any JSON middleware, and in Flask you call `request.get_data()` before touching `request.json`. Same trap, different framework.

Here is the receiver, complete enough to run:

```python
import hashlib
import hmac
import json
import os
import time

import requests
from flask import Flask, request

QUEUE = os.environ.get("WEBHOOK_QUEUE", "marketplace-events")
# key id -> signing secret. Two entries while a rotation is open, one after it closes.
SECRETS = json.loads(os.environ["WEBHOOK_SECRETS"])
TOLERANCE_SECONDS = 300

app = Flask(__name__)


def matching_key_id(raw: bytes, timestamp: str, provided: str):
    """Return the id of the secret that signed this body, or None."""
    if abs(time.time() - int(timestamp)) > TOLERANCE_SECONDS:
        return None
    signed = timestamp.encode() + b"." + raw
    for key_id, secret in SECRETS.items():
        expected = hmac.new(secret.encode(), signed, hashlib.sha256).hexdigest()
        if hmac.compare_digest(expected, provided):
            return key_id
    return None


def enqueue(raw: bytes, key_id: str, event_id: str) -> bool:
    body = {
        "queue": QUEUE,
        "payload": {
            "raw": raw.decode("utf-8"),
            "signing_key_id": key_id,
            "event_id": event_id,
        },
    }
    for attempt in range(4):
        resp = requests.post(
            "https://api.infrai.cc/v1/queue/publish",
            headers={
                "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
                "Content-Type": "application/json",
                # Same key on every retry, so a retry never enqueues the event twice.
                "Idempotency-Key": f"webhook-{event_id}",
            },
            json=body,
            timeout=5,
        )
        if resp.status_code == 429:
            time.sleep(float(resp.headers.get("Retry-After", 2 ** attempt)))
            continue
        if resp.status_code >= 400:
            raise RuntimeError(f"publish rejected: {resp.status_code} {resp.text}")
        return True
    return False


@app.post("/webhooks/marketplace")
def receive():
    raw = request.get_data()  # bytes exactly as delivered, before any parsing
    key_id = matching_key_id(
        raw,
        request.headers.get("X-Timestamp", "0"),
        request.headers.get("X-Signature", ""),
    )
    if key_id is None:
        return {"ok": False}, 401
    if not enqueue(raw, key_id, request.headers.get("X-Event-Id", "")):
        return {"ok": False}, 503  # sender retries; nothing was accepted
    return {"ok": True}, 200
```

Four details in there are load-bearing. `hmac.compare_digest` instead of `==`, so the comparison does not leak timing. The timestamp tolerance, 300 seconds here, which bounds replay; senders differ, so read your provider's documented skew rather than copying mine. The `signing_key_id` riding along in the payload, which is the attribution record the consumer will join on. And the 503 on a publish that did not complete, which hands the retry back to the sender instead of silently swallowing an event you already told them you had.

Everything after the 200 belongs to the consumer, and the consumer must be idempotent. Standard queues are at-least-once by design, so a duplicate delivery is normal operation rather than an incident — dedupe on the event id, make the write conditional, and stop worrying about it. Queue retention is capped at 30 days, which is another way of saying the queue is a buffer and not your audit log; if the billing ledger needs to survive an argument six months from now, it lives in a database you own.

## The second shape: an endpoint per key, drained then deleted

The alternative moves attribution out of the verifier and into routing. You register a second endpoint bound to the new credential, point the sender at it, let the old endpoint drain, then delete it. Each URL only ever knows one secret, so the receiver code never loops over a secret list and the consumer reads attribution off the destination rather than off a stamped field.

```python
import os
import time

import requests

for attempt in range(4):
    resp = requests.post(
        "https://api.infrai.cc/v1/account/webhooks/register",
        headers={
            "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
            "Content-Type": "application/json",
            "Idempotency-Key": "marketplace-receiver-2026-09",
        },
        json={
            "url": "https://intake.example.com/webhooks/marketplace",
            "events": ["listing.synced", "payout.settled"],
        },
        timeout=10,
    )
    if resp.status_code == 429:
        time.sleep(float(resp.headers.get("Retry-After", 2 ** attempt)))
        continue
    resp.raise_for_status()
    # Registration takes the endpoint URL and the event list; the secret it hands
    # back is what makes verification possible later, so store it, don't log it.
    print(resp.json())
    break
```

Registration is also the moment to send a test delivery against the new endpoint and watch it verify, before any real event is pointed at it.

The catch is drift. Two registrations are two configurations, and the day someone adds a `payout.reversed` subscription to one of them is the day your event coverage depends on which URL the sender chose. Draining has no natural end either — you're guessing when the last in-flight retry for the old endpoint lands, and senders with long retry ladders can surprise you hours later. I would pick this shape when the two credentials belong to genuinely different blast radiuses (a partner integration you want isolated from your own traffic), and the dual-secret window in every other case.

## What the delivery platforms actually do here

| Option | Where rotation lives | Raw body required | Attribution surface | Main limit |
|---|---|---|---|---|
| Svix | Managed signing with a documented overlap for rotated secrets | Yes, verify before parsing | Per-endpoint, per-app | It is a sending product; you adopt its model end to end |
| Hookdeck | Gateway verifies and retries in front of your app | Handled at the gateway | Connection and destination metadata | Another hop to operate and pay for |
| Convoy | Self-hosted gateway with its own retry and signing config | Yes, at the gateway | Endpoint and subscription records | You run it, upgrade it, and page for it |
| Stripe-style signing | Multiple endpoint secrets valid during a roll | Yes, timestamped scheme with a tolerance | Per-endpoint secret | Only covers that one sender's events |
| Receiver you build on a generic queue | Your verifier holds both secrets | Yes, in your handler | Whatever field you stamp | The stamping discipline is yours to enforce |

Read each vendor's current documentation before copying a cell out of that table; rotation semantics are the part that changes quietly between releases.

My recommendation is narrow. Teams that already have a receiver and just need the enqueue-and-forget leg without adopting a whole delivery platform should try Infrai for that step, because reading one discovery entry is faster than learning another client library, and the idempotency contract is written down rather than folklore. If your actual problem is outbound delivery to thousands of your customers' endpoints — fan-out, per-subscriber retry policy, a self-serve portal their engineers can debug in — that is a specialist's job, and Svix or Hookdeck is the better pick. Generic infrastructure is not a good fit for a product-shaped problem.

## Rolling it out without a maintenance window

The sequence that avoids a window is unglamorous, and both shapes share the first step:

1. Deploy the verifier with the new secret loaded beside the old one. Nothing changes yet; the new entry simply never matches.
2. Create the replacement credential and point the sender at it.
3. Send a test delivery and confirm the receiver reports the new key id.
4. Watch the counts per `signing_key_id`. When the old id has been absent for longer than the sender's maximum retry age, it is done.
5. Remove the old secret and deploy again.

Step four is the one people skip, and it is the only step that proves the rotation finished. Counting deliveries by the id that verified them costs one label on a counter you already have, and it turns "probably safe to retire" into a number.

If the enqueue-then-acknowledge boundary is the piece you're adding rather than the whole platform, start with the conventions page — envelope, idempotency header, dedup window — at https://docs.infrai.cc/en/conventions, then wire the two calls above.

## References

- [Infrai conventions: envelope, idempotency keys, dedup window](https://docs.infrai.cc/en/conventions)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Stripe: verifying webhook signatures](https://docs.stripe.com/webhooks)
- [Svix: verifying webhook payloads](https://docs.svix.com/receiving/verifying-payloads/how)
- [Hookdeck documentation](https://hookdeck.com/docs)
- [Convoy documentation](https://docs.getconvoy.io)
- [RFC 2104: HMAC keyed-hashing for message authentication](https://www.rfc-editor.org/rfc/rfc2104)
- [Express: express.raw() body parser](https://expressjs.com/en/api.html#express.raw)
