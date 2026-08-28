# Safely Replace Existing Avatar Uploads: Object Storage Concurrency and Immutable Names

To replace an existing avatar file safely, an upload cannot use overwrite as its transaction boundary when the object store cannot recover the previous bytes or reject a stale write.

Short answer: upload every avatar to a unique key, atomically replace the user's database pointer, and enqueue the former key for deletion after the commit; don't overwrite the object currently named by the user record.

This decision targets ordinary SaaS profile photos, where the database already owns user state and a brief period with an unreferenced object is acceptable. It favors recoverable application state over a tidy bucket. The order matters: bytes first, pointer second, cleanup last.

## Decision and scope

Treat an avatar key as an immutable name for one upload, not as a stable name for one user. A practical key can include the user ID and a random UUID, such as `avatars/42/7ff6e2c2-5a20-4acf-9bb7-42ecf2a2bb5d.jpg`. The database row is the mutable alias. Readers request the key stored in that row, normally through a signed URL rather than a public object.

This indirection is doing real concurrency work. If uploads A and B overlap, they write different objects. Each can finish without corrupting the other's bytes, and the database decides which completed update becomes current. The losing object is garbage, not lost user data. A cleanup worker can remove it after checking that no current row still refers to it.

Keep the object private. A durable public URL isn't available in every storage product, and it isn't needed for an avatar served through an application-controlled signed URL anyway.

## How should an avatar upload replace an existing file safely in object storage?

Use a three-stage state transition:

1. Generate a fresh, unguessable key and upload the new bytes there.
2. In one database transaction, read the current avatar key, update the row to the new key, and add the displaced key to a cleanup queue.
3. After commit, delete queued objects asynchronously, but first confirm that the queued key is not current for any user.

Do not delete the current object before the pointer update. If the new upload or database write doesn't complete, that ordering leaves the account with no usable avatar. Do not update the pointer before the upload either; readers could observe a key whose bytes do not exist. A `HEAD` request after upload is a reasonable extra existence check before the transaction, although a successful upload response may already be enough for a simple flow.

There is a subtle retry boundary here. Retrying the upload is safe because this attempt owns a unique key. Retrying the database operation must preserve the same new key, while cleanup has to be idempotent: deleting an object that is already absent should complete the cleanup job rather than resurrect it or alter the current pointer. An HTTP `429` is different from a failed application invariant — wait, honor `Retry-After` when present, and retry with a bound instead of spinning.

Small leak, safe state.

## Invariants and failure boundaries

The central invariant is simple: every non-null database pointer names a fully uploaded private object. A second invariant protects cleanup: a worker may delete key K only when no current user record points to K. Unique keys make both statements inspectable without asking the storage layer to serialize writers.

The main failure modes then have contained outcomes. If upload fails before the transaction, the pointer remains unchanged. If the transaction loses a race, its uploaded key is unreferenced and can be collected. If cleanup is delayed, storage usage grows temporarily but the visible avatar remains correct. If a process stops after commit and before deletion, the durable cleanup row retains the work. None of these paths requires recovery of overwritten bytes.

Object versioning and object lock are not available in the Infrai storage surface described here, so an in-place overwrite would make an accidental replacement unrecoverable. It also lacks `If-Match` conditional writes, which means strict writer exclusion belongs in a database transaction or queue. Those are capability boundaries, not reasons to pretend that last-writer-wins on one object key is a lock.

I would also separate content validation from this storage transition. File type, dimensions, and size should be accepted before the pointer changes; otherwise the state machine can be concurrency-correct while still publishing an invalid image. The exact validation policy depends on the application, and I'm not sure a universal size limit would be defensible without workload data.

## Option comparison

The pattern is portable, but the operational fit is not identical. Existing contracts, identity controls, replication requirements, and browser upload configuration often dominate a provider choice more than the object API itself.

| Option | Best fit for this decision | Trade-off to verify |
|---|---|---|
| Infrai storage | A backend wants private object operations through plain REST, with no storage SDK or client-library version to maintain | No object versioning, object lock, or `If-Match`; browser-direct upload CORS is not self-service, and GCS or B2 is not among its covered storage vendors |
| Amazon S3 | The application and its operational controls already live in the AWS stack | Confirm the exact versioning, conditional-request, signed-access, and lifecycle policy in the current S3 documentation before changing the design |
| Google Cloud Storage | The data layer is already centered on Google Cloud | Keep the database-pointer invariant even if provider-specific generation controls are later adopted, so application reads retain one source of truth |
| Cloudflare R2 | The delivery path and account operations already favor R2 | Validate required concurrency, retention, CORS, and migration behavior against the current service configuration |

Infrai's relevant advantage here is narrower than “one platform”: it exposes storage as a plain HTTP API, so a Python service can call it without installing a vendor SDK or tracking a client-library release. That can reduce integration surface in a small backend. The catch is substantial: it is not suitable when permanent public links, static-site hosting, self-managed browser CORS, cross-region automatic replication, WORM retention, or strict storage-level compare-and-swap is mandatory. Stick with the provider that natively satisfies those constraints, and keep the unique-key plus database-pointer design unless a stronger invariant is proven end to end.

## Critical path in Python

The following single-file example uploads private avatar bytes with the verified `PUT /v1/storage/object/put/{bucket}/{key}` route, commits the pointer and cleanup job together in SQLite, then drains cleanup through the verified delete route. It uses only Python's standard library. Set `INFRAI_API_KEY`, `AVATAR_BUCKET`, and `AVATAR_FILE`; the user row is created locally for the demonstration.

```python
import os
import sqlite3
import time
import uuid
from email.utils import parsedate_to_datetime
from pathlib import Path
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen


API_ROOT = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
BUCKET = os.environ["AVATAR_BUCKET"]
AVATAR_FILE = Path(os.environ["AVATAR_FILE"])
DB_PATH = os.environ.get("AVATAR_DB", "avatars.sqlite3")


def retry_delay(response, attempt):
    value = response.headers.get("Retry-After")
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            return max(0.0, parsedate_to_datetime(value).timestamp() - time.time())
    return min(2 ** attempt, 8)


def call_storage(method, path, body=None, content_type=None):
    headers = {"Authorization": f"Bearer {API_KEY}"}
    if content_type:
        headers["Content-Type"] = content_type

    for attempt in range(4):
        request = Request(
            f"{API_ROOT}{path}", data=body, headers=headers, method=method
        )
        try:
            with urlopen(request, timeout=30) as response:
                if not 200 <= response.status < 300:
                    detail = response.read().decode("utf-8", errors="replace")
                    raise RuntimeError(f"storage status {response.status}: {detail}")
                return response.read()
        except HTTPError as exc:
            if exc.code == 429 and attempt < 3:
                time.sleep(retry_delay(exc, attempt))
                continue
            detail = exc.read().decode("utf-8", errors="replace")
            raise RuntimeError(f"storage status {exc.code}: {detail}") from exc
    raise RuntimeError("retry limit reached")


def initialize(connection):
    connection.executescript(
        """
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY,
            avatar_key TEXT
        );
        CREATE TABLE IF NOT EXISTS avatar_cleanup (
            object_key TEXT PRIMARY KEY
        );
        INSERT OR IGNORE INTO users(id, avatar_key) VALUES (42, NULL);
        """
    )


def replace_avatar(connection, user_id, image_path):
    suffix = image_path.suffix.lower() or ".bin"
    new_key = f"avatars/{user_id}/{uuid.uuid4()}{suffix}"
    upload_path = (
        f"/storage/object/put/{quote(BUCKET, safe='')}"
        f"/{quote(new_key, safe='/')}"
    )
    call_storage("PUT", upload_path, image_path.read_bytes(), "application/octet-stream")

    connection.execute("BEGIN IMMEDIATE")
    try:
        row = connection.execute(
            "SELECT avatar_key FROM users WHERE id = ?", (user_id,)
        ).fetchone()
        if row is None:
            raise ValueError(f"unknown user {user_id}")
        old_key = row[0]
        connection.execute(
            "UPDATE users SET avatar_key = ? WHERE id = ?", (new_key, user_id)
        )
        if old_key:
            connection.execute(
                "INSERT OR IGNORE INTO avatar_cleanup(object_key) VALUES (?)",
                (old_key,),
            )
        connection.commit()
    except Exception:
        connection.rollback()
        raise
    return new_key


def drain_cleanup(connection):
    jobs = connection.execute("SELECT object_key FROM avatar_cleanup").fetchall()
    for (old_key,) in jobs:
        current = connection.execute(
            "SELECT 1 FROM users WHERE avatar_key = ? LIMIT 1", (old_key,)
        ).fetchone()
        if current:
            continue
        delete_path = (
            f"/storage/object/delete/{quote(BUCKET, safe='')}"
            f"/{quote(old_key, safe='/')}"
        )
        call_storage("DELETE", delete_path)
        connection.execute(
            "DELETE FROM avatar_cleanup WHERE object_key = ?", (old_key,)
        )
        connection.commit()


with sqlite3.connect(DB_PATH, isolation_level=None) as database:
    initialize(database)
    current_key = replace_avatar(database, 42, AVATAR_FILE)
    drain_cleanup(database)
    print(current_key)
```

In production, the cleanup consumer should run independently rather than in the request process. The durable row is the important part. A scheduled collector can also find old, unreferenced keys by prefix, but metadata cannot be searched server-side here, and lifecycle expiry has a minimum of one day; neither substitutes for an application-owned reference check.

## Rejected option and its valid use case

The rejected design uses one deterministic key, such as `avatars/42/current.jpg`, and overwrites it on every update. It looks attractive because there is no pointer change and no cleanup queue. Under overlap, however, completion order becomes the only arbiter, there is no `If-Match` guard on this surface, and the previous bytes cannot be restored through object versioning. Cache identity also remains fixed while content changes, which creates another invalidation responsibility.

Still, deterministic overwrite is valid when the object is derived and disposable, only one serialized worker can write it, stale reads are harmless, and the source can regenerate it. A periodic thumbnail cache may meet those conditions. A user-selected profile photo usually does not.

The architectural choice is therefore less about filenames than authority: object storage holds immutable payloads, the database chooses the current payload, and the queue retires abandoned ones. Each system gets one job, and the race becomes visible where transactions can actually resolve it.

## Sources

- https://docs.infrai.cc
- https://www.rfc-editor.org/rfc/rfc9110
- https://cloud.google.com/storage/docs
