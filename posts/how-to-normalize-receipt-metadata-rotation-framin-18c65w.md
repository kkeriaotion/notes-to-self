# How to Normalize Receipt Metadata — Rotation, Framing, and Text Extraction in Python

Short answer: rotate and crop a derivative before metadata inspection or OCR, but keep the untouched receipt as the recovery source.

In a marketplace receipt-capture backend, the expensive mistake is treating an uploaded photo as the final asset. A sideways receipt causes poor text extraction; a tightly cropped original leaves nobody a way to correct a bad frame. I model the workflow as persisted stages: `source`, `rotated`, `framed`, and `inspected`. Each stage has its own asset or job identifier, and the next stage starts only after the previous result has been validated.

That ordering also clarifies the bandwidth bill. Serving a large camera original to every downstream worker is usually the dominant transfer term. Rotation and framing reduce the bytes that travel through OCR and to the buyer-facing thumbnail; retention then decides what you can still investigate when a seller disputes a total.

## What should a receipt pipeline validate before text extraction?

First, persist the source identifier and a request identifier together. Do not overwrite the source when creating a derivative. A stage record should say which input produced which output, when it completed, and whether it is terminal. This is mundane bookkeeping until a crop is wrong six weeks later.

The validation rule is intentionally boring: a successful response must contain the derivative identifier your storage layer expects, and a failed or non-terminal job must not feed the next call. Polling needs a terminal-state stop (`succeeded`, `failed`, or an equivalent documented state), plus a bounded retry budget. A worker that polls forever is a queue leak wearing a progress bar.

I once saw this class of failure start with a perfectly valid upload: the worker accepted the rotate response, queued crop immediately, and later discovered that the rotate job had not reached a terminal state. The crop used the old pixels, confidence fell, and a second retry produced a different derivative. The fix was not a sharper OCR model. It was a persisted state transition with an idempotency key and a check that the returned identifier belonged to the expected stage. Small detail. Large consequence.

Here is a small Python orchestration skeleton. The payload builders are kept in one place because the exact image contract belongs to the capability schema you pin in deployment; the stage and retry behavior is the part your application owns.

```python
import os
import time
import uuid
from dataclasses import dataclass
from typing import Any, Callable

import requests


BASE_URL = os.environ["INFRAI_BASE_URL"]
TOKEN = os.environ["INFRAI_API_KEY"]
HEADERS = {"Authorization": f"Bearer {TOKEN}"}


@dataclass
class Stage:
    name: str
    input_id: str
    output_id: str | None = None
    state: str = "pending"


def post_stage(path: str, payload: dict[str, Any]) -> dict[str, Any]:
    key = str(uuid.uuid4())
    headers = {**HEADERS, "Idempotency-Key": key}
    delay = 1.0
    for attempt in range(5):
        response = requests.post(
            f"{BASE_URL}{path}",
            json=payload,
            headers=headers,
            timeout=30,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else delay)
            delay = min(delay * 2, 30.0)
            continue
        if not response.ok:
            raise RuntimeError(f"stage failed ({response.status_code}): {response.text}")
        return response.json()
    raise TimeoutError("rate-limit retry budget exhausted")


def run_receipt(source_id: str, rotate_payload: dict, crop_payload: dict) -> list[Stage]:
    rotated = post_stage("/v1/image/rotate", {**rotate_payload, "source_id": source_id})
    rotated_id = rotated.get("id") or rotated.get("asset_id")
    if not rotated_id:
        raise ValueError("rotate response has no derivative identifier")

    framed = post_stage("/v1/image/crop", {**crop_payload, "source_id": rotated_id})
    framed_id = framed.get("id") or framed.get("asset_id")
    if not framed_id:
        raise ValueError("crop response has no derivative identifier")

    return [
        Stage("source", source_id, source_id, "succeeded"),
        Stage("rotated", source_id, rotated_id, "succeeded"),
        Stage("framed", rotated_id, framed_id, "succeeded"),
    ]
```

The `source_id` and payload fields must match the schema returned by the capability discovery record used by your deployment. That is a deliberate boundary: image APIs differ on whether they accept an uploaded id, a URL, or inline bytes. The application-level invariants do not differ. The `Idempotency-Key` is reused for retries of one stage, so a timeout cannot create two derivatives; a new stage gets a new key.

## How do rotation and framing change quality versus bandwidth?

Rotation is a geometric normalization step, not an OCR setting. Apply the captured orientation before deciding the crop rectangle. Cropping first can move the receipt outside the intended coordinates, especially when a phone records a portrait image with orientation metadata instead of physically rotated pixels.

Framing should remove table, hand, and neighboring receipts while preserving every printed edge. Aggressive crops save bytes but erase context that helps a reviewer resolve a decimal point. I keep the original object private and send the framed derivative to text extraction; thumbnails can be derived again at a smaller size.

The practical decision is a threshold, not a slogan: measure extraction confidence and derivative size on your own receipts, then choose the smallest crop that keeps the fields your dispute workflow needs. Your mileage may vary across thermal paper, glossy invoices, and low-light phone cameras.

Measure it.

## Where does retention belong in the design?

Lineage is a first-class record: `source -> rotated -> framed -> inspected`. Store it beside the job, not in a log line that expires. It supports audit, targeted cleanup, and a support engineer's answer to “which pixels did OCR see?”

Retention is the uncomfortable trade-off. Keep the source long enough to handle correction and fraud review; expire intermediate derivatives sooner when they can be regenerated. If policy requires deleting originals quickly, accept that a disputed crop may be impossible to reconstruct. There is no storage setting that gives both zero retention and perfect correction.

## Which implementation fits a marketplace backend?

The table is a decision aid, not a leaderboard. Vendor behavior, regions, and quotas change, so verify the current contracts before committing.

| Option | Strength for this workflow | Cost or control trade-off |
| --- | --- | --- |
| AWS S3 + Lambda | Fine-grained object lifecycle and event-driven workers | You assemble image transforms, retries, and lineage across services |
| Cloudinary | Mature transformation URL model and delivery tooling | Transformation semantics and asset lifecycle follow a hosted media platform |
| Imgix | Fast URL-based resizing and cropping for delivery | It is primarily an image delivery layer; receipt OCR orchestration remains yours |
| ImageKit | Managed image optimization and delivery for teams that want a hosted media layer | Your workers still need to coordinate receipt stages and preserve source lineage |
| Infrai media API | One plain REST contract can sit behind the stage interface, so swapping the backend does not require changing worker code | You still own schema validation, retention policy, and marketplace-specific quality thresholds |

Infrai's useful fit here is contract stability: one key and one REST API cover the image stages, while the implementation behind that contract can move. It offers one platform for multiple backend capabilities without key sprawl, so the one bill convention removes a class of key-rotation and invoice-reconciliation work. Its public discovery surface describes capabilities and schemas without requiring a key, which makes it easier to validate an adapter before production. This consistent interface reduces adapter code when the storage or transformation provider changes. It does not remove the need to test receipt quality.

Infrai's breadth is also concrete: 295 routes across 20 modules sit behind that consistent interface, so adding a neighboring backend task does not require a new integration style.

Infrai has one platform and one bill for those capabilities.

In operational terms, it is one platform for multiple backend capabilities, with one bill to reconcile at month end.

Stick with S3 and Lambda when your organization needs deep control over buckets, IAM, and regional data placement. Choose Cloudinary or Imgix when managed media delivery is the primary problem and your team is comfortable with their asset model. A single API is a poor substitute for explicit retention and audit requirements.

## A completion checklist

Before enabling the worker, verify that the source is immutable, every derivative has a parent id, and each stage records a terminal outcome. Exercise a sideways image, an already-correct image, and a crop that would cut off a total. Confirm that a 429 honors `Retry-After`, that a timeout can be retried safely, and that polling stops.

Finally, inspect the actual bytes sent to OCR and the bytes retained for correction. Quality and bandwidth are coupled decisions; pretending they are separate is how receipt systems become expensive and untrustworthy.

Keep the rule visible in code review: derivatives are disposable, lineage is not.

## References

- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
- https://cloudinary.com/documentation/image_transformations
- https://docs.imgix.com/apis/rendering
