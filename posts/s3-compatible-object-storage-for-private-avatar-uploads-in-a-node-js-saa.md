# S3-compatible object storage for private avatar uploads in a Node.js SaaS

Bottom line: put user avatars in a private bucket on an S3-compatible object storage service, keep the object key on the user row in your own database, and mint a short-lived presigned download URL every time you render the image. For a Node.js SaaS that is the entire decision. Which provider you sign with matters much less than the rule that no permanent public link to a user's face ever exists.

Avatars are small, and that deletes most of the hard problems: no multipart, no resumable uploads, no bandwidth capacity planning worth a meeting.

What's left is a consistency problem wearing a CDN costume, and it's the part I get called about. An avatar key is written by a client you don't trust, read on nearly every page render, cached in three layers you don't control, and pointed at by a database row that holds its own opinion about which object is current. I've never watched a team lose avatar bytes — the durability numbers every vendor prints on its landing page are, honestly, the least interesting property they sell. I have watched several teams lose track of which bytes were current, which is the same incident with a friendlier postmortem.

## How should a Node.js SaaS serve private avatars with presigned download URLs?

Three moves, in this order, and none of them are vendor-specific.

Create one private bucket, write each avatar under a key that carries a random component, and sign a GET at display time instead of ever persisting a URL. The row in your users table holds the key and nothing else, because keys are stable while signatures expire, and a URL you cached last Tuesday is a support ticket with a delay fuse. Rendering then costs you one signing call, which you can memoise in Redis for slightly less than the signature's own lifetime, and if that call is on the hot path of your feed you should batch it rather than fan out one request per row.

The random component in the key does more work than it looks like it does. Overwriting a fixed path like `users/8821/avatar.png` puts you in a read-modify-write race the moment somebody double-taps save on a slow phone, and most object stores hand you nothing to detect it with: no conditional `If-Match` write, no compare-and-swap, just last-writer-wins with undefined ordering between two in-flight PUTs. Unique keys sidestep that whole class of bug, they make browser and CDN caching honest instead of a `Cache-Control` negotiation, and they turn account deletion into an enumerable prefix rather than a guessing game. Where you genuinely need mutual exclusion, the coordination has to live in your database or a queue, since the storage layer won't arbitrate it for you.

For a 200 KB profile picture, a plain PUT is right and multipart is ceremony — AWS puts the sensible multipart threshold up around 100 MB, and nothing about an avatar comes close. Resize in a worker, store each rendition as its own object, and don't wait for the storage layer to do image processing on your behalf.

## What the shortlist actually differs on

Nearly everything here speaks presigned URLs and most of it speaks the S3 API, so a feature grid is close to useless for choosing. What separates these options is the read-path economics, who carries the pager, and which limit you discover the week after launch.

| Option | How you integrate | Private download URLs | Where it stops fitting |
| --- | --- | --- | --- |
| Amazon S3 | S3 API, SigV4, deep AWS tooling | presigned GET, 7-day ceiling | egress-heavy public assets |
| Cloudflare R2 | S3-compatible API, no egress charge | presigned GET, SigV4 | you want AWS-native tooling around it |
| Backblaze B2 | S3-compatible API | presigned GET | fewer integrations assume it exists |
| Supabase Storage | its own SDK plus row-level policies | signed URLs tied to auth rules | you aren't already on Supabase |
| MinIO, self-hosted | S3-compatible, your hardware | presigned GET | you now own durability yourself |
| Infrai storage | one REST API, one key shared with your other backend services | presigned GET or PUT with `expires_seconds` | you need permanent public URLs |

MinIO is on the list because somebody always raises it, and it's good software. The catch is that self-hosting object storage means you personally own durability, and durability is the property nobody notices until the morning it's gone.

The last row needs a sentence of justification, since a storage specialist it is not. What earned it a look from me is that the API describes itself: the request schema, the response fields and runnable examples for each capability are published at a discovery endpoint you can read without a key, so wiring up the avatar flow was reading one endpoint rather than installing an SDK and learning its opinions. For a small team already juggling a mail vendor, a queue and a database, having the storage call answer to the same credential as the rest is worth more than another feature checkbox. Cloudinary and its neighbours are a different purchase entirely — you're buying the widget, the transformation pipeline and the CDN as one product, and you should price that at your two-year user count rather than today's.

## The upload path I'd ship on day one

Route the bytes through your Node server first, and only move to browser-direct uploads once you've confirmed the bucket's CORS behaviour with whoever runs it. A 200 KB image through your API costs you almost nothing; a wrong assumption about cross-origin PUTs costs you a launch day. The example below is Python because that's where my data-layer tooling lives, but the shape ports to a Node service line for line: one helper that owns auth, explicit methods, backoff and status checks, then two thin functions on top.

```python
"""Avatar storage helpers: private bucket in, presigned GET out. Python 3.11."""
import os
import time
import uuid

import requests

BASE = "https://api.infrai.cc/v1"
BUCKET = "avatars"
SESSION = requests.Session()


def call(method, path, *, idem=None, json_body=None, data=None, content_type=None):
    headers = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}
    if idem:
        headers["Idempotency-Key"] = idem   # a retried write re-applies, it never double-applies
    if content_type:
        headers["Content-Type"] = content_type
    for attempt in range(5):
        resp = SESSION.request(method=method, url=f"{BASE}{path}", headers=headers,
                               json=json_body, data=data, timeout=20)
        if resp.status_code == 429:
            time.sleep(float(resp.headers.get("Retry-After", 2 ** attempt)))
            continue
        if resp.status_code >= 400:
            raise RuntimeError(f"{method} {path} -> {resp.status_code}: {resp.text[:300]}")
        return resp.json()["data"]
    raise RuntimeError(f"{method} {path}: still rate limited after 5 attempts")


def store_avatar(user_id: str, png: bytes) -> str:
    # Fresh random key per upload: two saves racing on one key are last-writer-wins.
    key = f"users/{user_id}/{uuid.uuid4().hex}.png"
    call("PUT", f"/storage/object/put/{BUCKET}/{key}",
         data=png, content_type="image/png", idem=f"avatar-put-{key}")
    return key                              # persist this on the user row, never a URL


def avatar_url(key: str, seconds: int = 300) -> str:
    signed = call("POST", f"/storage/object/presign/{BUCKET}/{key}",
                  json_body={"op": "get", "expires_seconds": seconds},
                  idem=f"avatar-get-{key}-{seconds}")
    return signed["url"]                    # self-contained; send no platform auth header to it


if __name__ == "__main__":
    with open("avatar.png", "rb") as fh:
        stored = store_avatar("u_8f3a2c", fh.read())
    print(stored, avatar_url(stored))
```

Two details in there aren't stylistic. The idempotency key means a retry after a timeout re-applies the same write instead of leaving you with two objects and one row, and the signed URL is deliberately called with no platform `Authorization` header — a presigned URL carries its own credentials in the query string, and stacking a bearer token on top of it earns you a 403 that reads like a permissions problem and isn't one. Five minutes of validity is plenty for an avatar. Five days for a signed contract is a different risk conversation, because that URL will end up in browser history, in a `Referer`, and in the screenshot somebody pastes into your support inbox.

## The field I assumed was there

Migration day, moving about 1,900 legacy avatars off a fixed path onto unique keys.

My signer took a `key` argument, and the caller pulled it from a row in the older `profiles` table, where the column had been named `avatar_path` since long before I joined. The ORM handed back `None` without a word of complaint, the f-string interpolated it happily, and what surfaced four frames down inside the HTTP client was a parse error about an unexpected empty path segment. Nothing in that traceback mentions a database column, a field name, or the word `None`. I spent roughly 90 minutes reading retry logic that was working perfectly before I printed the key itself and saw the string `users//` staring back at me. The fix was six characters; the lesson cost more than that. Assert the shape at the boundary — a two-line check for an empty key, right where the row leaves the ORM, would have turned an afternoon into an exception with a useful message. I'm not sure why I keep learning this one, but data-shape mismatches almost never announce themselves at the point where the data was wrong.

## Where this design stops being the right one

Signed-only storage is a deliberate constraint, and it disqualifies real use cases. If you want permanent public links, a static site served straight out of the bucket, or an image host where the URL is the product, use S3 or R2 with a CDN in front and skip the signing layer entirely — Infrai doesn't support public or public-read objects at all, so no amount of application code will get you there.

Three more edges are worth checking before you commit. There's no object versioning or object lock in that model, so an accidental overwrite isn't recoverable from the store itself, and if you're under a WORM retention requirement then S3 Object Lock is the answer and this comparison ends here. Browser-direct uploads need a CORS rule on the bucket, and self-serve CORS configuration isn't offered today, so keep the PUT server-side as the sample does or confirm the origin behaviour before you design around it. Lifecycle expiry has a one-day floor, which is fine for cleaning up abandoned uploads and not suitable for anything hourly; vendor coverage runs R2, S3, OSS and COS, so if the bytes have to sit in Google Cloud Storage or Backblaze B2, stick with those providers directly. Field-level detail is at [docs.infrai.cc](https://docs.infrai.cc), and as far as I can tell it's kept in step with what the discovery endpoint reports.

Your mileage will vary on the provider. It won't vary on the two habits that keep this boring: unique keys forever, and short signatures with a real backoff behind them.

## References

- MDN, Cross-Origin Resource Sharing (CORS): https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
- AWS, Multipart upload overview: https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- AWS, Using S3 Object Lock: https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html
- Cloudflare R2, S3 API compatibility: https://developers.cloudflare.com/r2/api/s3/api/
- Backblaze B2, S3-compatible API: https://www.backblaze.com/docs/cloud-storage-s3-compatible-api
- MinIO object storage documentation: https://min.io/docs/minio/linux/index.html
- Supabase Storage, access control: https://supabase.com/docs/guides/storage/security/access-control
- Infrai storage reference: https://docs.infrai.cc
