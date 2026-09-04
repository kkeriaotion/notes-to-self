# 6 Checks for Large User-Uploaded Files — Database Blobs or Object Storage

**Short answer:** For a logistics backend that must retain signed user-uploaded documents until an explicit deletion deadline, keep searchable metadata and deletion state in the database, put large file bytes in object storage, and treat deletion as a verified workflow. A database blob can still be the cheaper operational choice for small volume, small files, and a team that values one transactional system more than independent upload throughput.

This decision is less about a storage invoice than about which subsystem should absorb multi-megabyte writes. A signed proof-of-delivery document has a business identity, a retention deadline, and a byte stream; forcing all three through one database transaction couples concerns that fail and scale differently. Don't choose from a price table before measuring upload size, concurrency, backup growth, restore time, and deletion lag.

## 1. Should a backend store large user-uploaded files as database blobs or object storage?

Use object storage for the file body when large-file throughput is the dominant constraint, while keeping a database row as the control record. The row should contain a stable document ID, tenant or shipment ID, object key, content length, checksum, state, and `delete_after` timestamp. The object key is a locator, not the source of business truth.

A database blob is reasonable when documents are small enough that they don't distort transaction latency, backups, replication, or restores, and when atomic creation with the surrounding record matters more than scaling the byte path separately. The catch is coupling: every upload now competes with ordinary queries and every retained byte follows the database through backup and recovery. Object storage separates that traffic, but it introduces a cross-system state machine because a database commit and an object write aren't one transaction.

That state machine matters.

For a signed logistics document, use explicit states such as `pending`, `available`, `deleting`, and `deleted`. Create the metadata record first, stream the body, verify the recorded byte count and digest, then mark it available. Readers must reject anything except `available`; cleanup must be retryable; reconciliation must find stale pending records and unreferenced objects. This is longer than saying "upload it to a bucket," but it names the actual failure modes: an object without a row, a row without an object, a partial stream, two cleanup workers claiming the same record, or a download served after its deadline.

## 2. Where should a throughput benchmark look for the byte-path bottleneck?

Measure it.

The useful benchmark crosses file size with concurrency. Test representative signed documents through the full request path, including authentication, checksum work, metadata writes, the byte sink, and any proxy that might buffer the request. Record p50 and p95 upload duration, failed and retried uploads, database transaction time, memory use, and bytes waiting in application processes. A 64 MB test object is a test fixture — not a universal threshold. Your mileage may vary once scanners, mobile links, or reverse proxies enter the path.

Stream bytes instead of reading the entire document into application memory. Apply backpressure, cap accepted size, and calculate the checksum during the stream. If the object interface supports multipart transfer, benchmark it with the same fixture set; don't assume more parts mean more useful throughput. The constraint could sit in the client uplink, application CPU, a proxy buffer, or the storage service, and a benchmark that bypasses one of those components answers a different question.

## 3. Make retention failure observable

A retention policy needs a machine-readable timestamp and an auditable transition. GDPR Article 17 defines a right to erasure and also lists circumstances and exceptions, so the effective deadline must come from the applicable policy rather than a developer's guess. Store the policy basis separately from `delete_after`; legal holds or contractual retention can then change eligibility without silently rewriting what the original rule was. At the deadline, a worker should claim eligible rows, mark them `deleting`, remove the byte object, confirm the removal according to the chosen storage interface, and then mark the metadata `deleted` or retain a minimal audit record as policy permits. Retries must use the stable document ID and object key. A second worker seeing `deleting` should resume or defer according to lease state, not create a new key.

Prove deletion.

Run a periodic reconciliation in both directions: metadata that points to no object, and objects that have no live metadata. Track oldest overdue deletion, counts in each state, age of pending uploads, cleanup retry count, and reconciliation mismatches. Those signals expose drift before an audit request does. They also make a migration measurable because the old and new byte stores can be compared against the same control table. A correct object placement can still leak a signed document through a careless delivery path, so authorize every download against the metadata record and its state, then apply a response caching policy appropriate to confidential documents. MDN documents `Cache-Control` response directives and their effects; use that vocabulary deliberately at the application or gateway boundary rather than relying on a browser default. The exact directive depends on the threat model and delivery design. For sensitive signed documents, evaluate a restrictive policy such as `private, no-store`, and verify actual response headers at every hop (application, gateway, and content delivery layer). I'm not sure a generic rule can settle every regulated logistics deployment because contractual retention and permitted caching differ; a security review and an end-to-end header test resolve that uncertainty. Also separate authorization lifetime from retention lifetime: a document may need to remain stored until Friday while a particular download grant should expire in minutes. Deleting access metadata is not deletion of the stored body, and deleting the body should invalidate every delivery path.

## 4. Compare recovery, consistency, and cost boundaries

There is no honest "cheapest backend" answer without workload dimensions. Compare monthly stored bytes, write and read operations, outbound transfer, database and replica growth, backup retention, restore exercises, engineering time for reconciliation, and the cost of missed deletion deadlines. Price belongs in the model, but architecture should follow the bottleneck and operational boundary.

| Check | Database blob tends to fit when | Object storage tends to fit when | Failure to test |
|---|---|---|---|
| Write path | Files and concurrency stay modest | Large streams should bypass database transactions | Upload latency harms unrelated queries |
| Recovery | One restore boundary is valuable | Byte recovery can be separate from metadata recovery | Restore time grows unnoticed |
| Consistency | Atomic row changes dominate | A small explicit state machine is acceptable | Orphans or missing objects |
| Retention | One system can prove deletion promptly | Lifecycle work is coordinated and verified | Metadata disappears before bytes |
| Delivery | Application-mediated reads are sufficient | Direct or proxied object reads reduce app traffic | Confidential responses become cacheable |

## 5. Implement a storage-neutral document boundary

The application layer should speak in document operations, not vendor operations. A narrow interface needs `put`, `open`, `delete`, and an existence or metadata check; the database owns identity, authorization, retention, and state transitions. It's enough abstraction to swap a blob column for an object service without pretending the two systems have identical consistency or recovery behavior.

The following Python sketch shows the important boundary. The 64 MB value is deliberately a configurable example, and the storage adapters are injected so deployment-specific APIs stay outside the workflow.

```python
from dataclasses import dataclass
from datetime import datetime
from hashlib import sha256
from typing import BinaryIO, Protocol

CHUNK_BYTES = 1024 * 1024


class ByteSink(Protocol):
    def put(self, key: str, chunks) -> None: ...
    def delete(self, key: str) -> None: ...
    def exists(self, key: str) -> bool: ...


@dataclass(frozen=True)
class DocumentPlan:
    document_id: str
    object_key: str
    delete_after: datetime


def checked_chunks(source: BinaryIO, digest, counter: list[int]):
    while chunk := source.read(CHUNK_BYTES):
        digest.update(chunk)
        counter[0] += len(chunk)
        yield chunk


def store_document(source: BinaryIO, plan: DocumentPlan, sink: ByteSink, records) -> None:
    records.create_pending(plan.document_id, plan.object_key, plan.delete_after)
    digest = sha256()
    byte_count = [0]

    try:
        sink.put(plan.object_key, checked_chunks(source, digest, byte_count))
        records.mark_available(plan.document_id, byte_count[0], digest.hexdigest())
    except Exception:
        records.mark_upload_failed(plan.document_id)
        raise


def delete_due_document(plan: DocumentPlan, sink: ByteSink, records) -> None:
    records.mark_deleting(plan.document_id)
    sink.delete(plan.object_key)
    if sink.exists(plan.object_key):
        raise RuntimeError("deletion could not be verified")
    records.mark_deleted(plan.document_id)
```

Production adapters still need bounded retries, request timeouts, authentication, access control, and durable worker leases. The point of the example is narrower: bytes flow as chunks, metadata records the digest and count only after the sink accepts the stream, and deletion remains observable until verification succeeds. The exception does not claim a particular service behavior; it prevents the workflow from declaring success without evidence.

## 6. Roll out by moving the byte path last

Begin with the control record and state machine while the current byte store remains authoritative. Add checksums and byte counts, run reconciliation, and establish deletion-lag alerts. Next, dual-write a bounded sample only if the team can compare both copies without serving ambiguous results; otherwise copy historical objects in batches, verify each digest, and switch reads per document through an explicit location field.

Keep rollback boring — a location flag should return reads to the old store while copied bytes remain intact. Stop the rollout if mismatch count rises, upload latency breaches the service objective, or overdue deletions accumulate. Only retire old blobs after every eligible record has a verified destination, restore procedures have been exercised, and the retention worker operates correctly against the new adapter.

Stick with database blobs when measured volume remains modest, one-system recovery is materially simpler, and the database workload has headroom. Choose an object-backed byte path when large-file throughput, independent scaling, or backup isolation justifies the extra consistency machinery. Neither choice removes the obligation to prove what was stored, who may read it, and when it was actually deleted.

## Further reading

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control
- https://gdpr-info.eu/art-17-gdpr/
