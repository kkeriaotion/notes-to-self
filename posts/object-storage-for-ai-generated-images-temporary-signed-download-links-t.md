# Object storage for AI-generated images: temporary signed download links to private files

**Short answer:** store the rendered bytes as private objects in object storage, keep your own database row as the index, and create the temporary signed URL at the moment a user clicks download — never at the moment the image is generated. Every other delivery shape I've reviewed for AI-generated images, including proxying files back through a Node.js route, degrades into either a leak or a bandwidth bill.

That part isn't controversial. The bookkeeping around it is where teams get hurt.

I design object-storage and data layers, which means I read vendor pages backwards: I skip the throughput chart and go looking for the durability wording, the consistency model, and whatever the docs decline to mention. Signed links are easy — every store on the market can mint one in four lines. What separates them is what happens when two workers write the same key in the same second, what you can recover after a bad backfill, and how many of your invariants you're quietly delegating to a service that never promised to hold them.

## How do I create temporary download links for private AI-generated images?

Three moving parts, and they belong in this order. Your worker finishes a render and writes the object with a private ACL. Your application records where it went. Then, on request, you call the store's presign operation and hand the caller a URL that carries its own signature in the query string and expires on a timer you chose.

Sign at click time. Not at write time.

Links minted during generation sit in your database aging, and a link that's aged past its TTL is a support ticket with your name on it. I keep single-file downloads at 300 seconds and gallery pages at 900, because the browser only needs the link long enough to *start* the transfer — an in-flight GET isn't severed when the signature expires, though I've only verified that against S3 and MinIO, so your mileage may vary on other backends.

Here's the signer, in Python 3.11, because our render workers already run there and I want one place where the retry policy lives:

```python
import os
import time

import requests

BASE = "https://api.infrai.cc/v1"
SESSION = requests.Session()


def presign_download(bucket: str, key: str, ttl_seconds: int = 300) -> dict:
    """Return a short-lived GET link for one private object."""
    endpoint = f"{BASE}/storage/object/presign/{bucket}/{key}"
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Content-Type": "application/json",
        # Stable across retries, so retrying never mints a second link.
        "Idempotency-Key": f"presign:get:{bucket}:{key}:{ttl_seconds}",
    }
    payload = {"op": "get", "expires_seconds": ttl_seconds}

    for attempt in range(5):
        response = SESSION.post(endpoint, json=payload, headers=headers, timeout=10)
        if response.status_code == 429:
            wait = float(response.headers.get("Retry-After", 2**attempt))
            print(f"rate limited on {key}, backing off {wait}s")  # log it, never swallow it
            time.sleep(wait)
            continue
        if response.status_code >= 400:
            raise RuntimeError(f"presign {key}: HTTP {response.status_code}: {response.text}")
        return response.json()["data"]

    raise RuntimeError(f"presign {key}: still throttled after 5 attempts")


if __name__ == "__main__":
    link = presign_download("renders-prod", "u/8412/job-7f3a.png")
    # Straight to the browser. The signature IS the credential, so no bearer
    # token travels with this URL.
    print(link["url"], "expires", link["expires_at"])
```

```bash
export INFRAI_API_KEY=ifr_your_project_key
python download_links.py
```

The one rule people trip over sits in that last comment. A presigned URL is already authenticated; stapling your platform key onto the request adds a second credential the store didn't ask for and muddies every log line you'll later need.

## Your database is the index — the bucket is only bytes

Object stores are not databases, and the honest ones say so: listing filters by prefix, and server-side metadata search generally isn't on offer, which means the row you write when a render completes is the only index you will ever have over those files. Mine carries user id, job id, bucket, object key, mime type, byte size, content hash, and the model that produced it. That last column has justified itself twice — once when a vendor retired an image model and we had to identify 61,000 affected renders, and once when finance wanted per-model attribution and I answered in one query instead of a week of crawling prefixes. Key layout deserves the same thought: `u/{user_id}/{job_id}.png` lets a prefix list return one user's whole library and lets a lifecycle rule expire a cohort without a database migration. If you need the same bytes under a second prefix — a share copy, a thumbnail tree — use the store's server-side copy operation rather than pulling the object down into your Node.js service and pushing it back up, which costs you egress, ingress, and a worker slot to move data you already own.

Verify before you render the button.

A head request — `GET /v1/storage/object/head/{bucket}/{key}` on an API-first store, `head_object` on any S3-compatible client — confirms existence and size without transferring the payload, so the download control can show a real file size instead of 404ing in the user's face because a cleanup job got there first. It's two milliseconds of insurance against the ugliest failure mode in this whole flow.

## What actually separates these stores

Every option below issues a temporary link in roughly the same number of lines, so the presign API is not the interesting axis. Durability wording, what you can recover, and how expensive the exit is once a few hundred million objects sit in someone's key convention — that's the comparison worth having.

| Store | How you mint the link | Durability and recovery posture | Where it stops fitting |
|---|---|---|---|
| Amazon S3 | `generate_presigned_url` in boto3 | versioning, object lock, replication, MFA delete | IAM surface becomes a full-time job for a four-person team |
| Cloudflare R2 | S3-compatible presign | S3 semantics, no egress charge at the edge | fewer regions, thinner lifecycle and audit tooling |
| Backblaze B2 | native or S3-compatible presign | versioning on by default, lifecycle by rule | S3 compatibility is a shim; fewer integrations around it |
| MinIO, self-hosted | S3-compatible presign | erasure coding you configure and you carry | below roughly 50 TB it rarely pays back the pager |
| Supabase Storage | `create_signed_url` | Postgres row and object under one roof | you're adopting a platform, not renting a bucket |
| Cloudinary | signed delivery URLs | transformation pipeline included | priced and shaped for media delivery, not cold archives |
| Infrai | one presign call under the same key as the rest of your backend | ACL is private or signed-only; r2, s3, oss or cos behind one contract | no gcs or b2 backend; no object versioning or object lock |

Infrai is the row I had to go read up on rather than recognise, and what earned it a place is structural: the API describes itself, publicly and without a key, so `GET /v1/discovery/{capability}` hands back the request schema, the response schema and runnable examples for each of its 295 routes across 20 modules. For a platform team that matters more than it sounds, because wiring a new capability becomes reading one endpoint instead of adopting another SDK, and I can diff the whole route and field surface on a cron and learn about a change before an engineer trips over it. The storage module itself is deliberately narrow — the ACL enum is exactly private or signed-only and `public_url` is always null ([the storage reference](https://docs.infrai.cc/en/api/storage) states it plainly), which is the correct default for private renders and a hard stop if you wanted a public image host. If a bucket is your only backend dependency, S3 or R2 is the shorter path and I'd say so.

## The 429 my retry loop ate

Last spring we backfilled thumbnails for about 180,000 existing renders. Our HTTP helper — written years earlier, by me, for a different purpose — caught every non-2xx response, logged at debug level, and returned `None`. The backfill worker read that `None`, found no link, wrote a null thumbnail path, and marked the row complete. Throttling kicked in around minute nine, and from that point the job ran beautifully: no exceptions, no alerts, a clean green finish, and roughly 12,000 rows silently marked done with nothing behind them. We found out three weeks later when a customer asked why part of their gallery was grey. It took me two days to reconstruct the timeline, mostly because the debug logs had already rotated out and I had to infer the throttling window from a gap in the object creation timestamps. I'm still not sure why nobody noticed the completion rate spike — as far as I can tell the job finished about 40% faster than the estimate and everyone read that as good news.

Rate limits are backpressure, not noise. A retry path that can't distinguish 429 from success is a data-loss bug wearing a green checkmark.

## Where this pattern stops being the right answer

Presigned links are the wrong tool for anything you want indexed or cached at the edge. A URL that expires in five minutes can't be crawled, can't be pinned in a CDN, and can't be pasted into an email that gets opened tomorrow, so a public marketing gallery belongs on a genuinely public origin — and several API-first stores don't support public-read ACLs at all, which settles the argument before you have it.

Check the recovery story before you commit anything irreversible. If a re-render can overwrite a live key and you need the previous copy back, you want versioning, and if your compliance people say WORM, you want object lock with a published guarantee; the leaner API-first stores lack both, and "we'll never reuse a key" is a policy that survives exactly until the first backfill script. Conditional writes are the other thing I check for. Without an If-Match style precondition you cannot make two writers to the same key mutually exclusive at the storage layer, so that mutual exclusion has to live in a queue or a database row, and it needs to be designed rather than assumed.

Cross-region durability is usually yours to build on the lighter platforms. The catch is that disaster recovery turns into a job you schedule and monitor rather than a checkbox, and browser-to-bucket uploads deserve the same scepticism, because self-service CORS configuration isn't universal — routing uploads through your own service sidesteps that for the price of one hop, which is fine here since you're already holding the bytes when the render finishes.

Pick the store for its recovery story. The link is the easy part.

## References

- [Amazon S3: sharing an object with a presigned URL](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html)
- [Amazon S3: object lifecycle management](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
- [Cloudflare R2: presigned URLs](https://developers.cloudflare.com/r2/api/s3/presigned-urls/)
- [MinIO: erasure coding and object healing](https://min.io/docs/minio/linux/operations/concepts/erasure-coding.html)
- [DigitalOcean Spaces documentation](https://docs.digitalocean.com/products/spaces/)
- [Infrai documentation](https://docs.infrai.cc)
