# A Failure-Boundary Guide to Startup Log Ingestion APIs and Centralized Search

Use a structured, asynchronous ingestion API behind an application-owned logging adapter, then choose the search backend by testing failure behavior rather than dashboard polish. The deciding constraint is control: a startup needs one event contract, bounded delivery semantics, and a credible way to leave before centralized application logs become part of every request path.

Short answer: the easiest backend logging feature is a small adapter that accepts structured events, batches them off the request path, and sends them to a centralized ingestion endpoint; the startup dashboard should search that stable schema, while crash artifacts travel through a separate path.

This is an architecture decision record, not a product ranking. The attractive demo is usually “send a string, see a chart.” The durable choice starts one layer lower, with the facts that must survive retries, deploys, schema changes, and a vendor change.

## What API should a startup use for centralized application logs ingestion and search?

Choose an HTTPS ingestion API that accepts structured JSON batches, but hide its exact payload and authentication scheme behind code owned by the application. Don't let dozens of call sites know an external field name. The application emits an internal `LogEvent`; one adapter maps that event to the selected destination, and a different interface handles search for the dashboard.

That split matters because ingestion and search have different failure boundaries. Ingestion runs near production traffic, where blocking, unbounded buffers, and retry storms can amplify an otherwise small telemetry problem. Search runs on the operator path, where a stale result is irritating but usually less dangerous than slowing every customer request. Combining both concerns in a single “logging client” makes those boundaries hard to see.

The event contract should include a timestamp, severity, service, environment, event name, message, and correlation identifier. Add fields only when their ownership is clear. A request identifier helps connect events from one request; it is not proof that two side effects committed atomically. A message is useful to a human; it is a poor substitute for a stable event name. Treat the contract like a database schema — reviewed, versioned when necessary, and tested at its producer boundary.

Keep secrets and unrestricted personal data out of the event at creation time. Redaction performed only in the destination is late: the sensitive value has already crossed the network and may have entered a retry buffer. The safer boundary is the application-owned adapter, where fields can be allowlisted before serialization.

Fast is good. Bounded is better.

## Invariants and failure boundaries

The first invariant is that logging must not decide whether a user request succeeds. Emit into a bounded in-process queue or a local collector, return control to the application, and make the overflow policy explicit. “Never lose a log” sounds responsible until the log destination is unavailable and the application exhausts memory. For ordinary application events, dropping a measured number of low-severity records can be the less damaging choice; audit records, financial state changes, and other evidence with retention obligations belong on a separately designed durable path.

The second invariant is that retries cannot silently create analytical lies. Give each event an identifier, keep retry windows bounded, and assume duplicates are possible unless the chosen system documents and demonstrates stronger behavior. Search queries and dashboard counts should tolerate duplicate delivery or deduplicate on that identifier. Exactly-once language deserves skepticism because the important question is not the label; it is where identity is assigned, how long deduplication state lives, and what happens after that window closes.

Ambiguity is normal.

The third invariant is that time is data, not decoration. Preserve the producer timestamp and distinguish it from ingestion time. A laptop waking from sleep, a mobile client reconnecting, or a queue draining after congestion can create late arrivals. If a dashboard groups only by ingestion time, an old incident can appear to be happening now. If it groups only by producer time, clock skew can move events across the investigation window. Store both when the architecture crosses unreliable clients or buffers.

Then test the ugly cases. Consider a batch of 200 events where the connection closes after bytes leave the process but before an acknowledgement returns. The sender cannot know whether the batch was accepted, so a retry can duplicate it; refusing to retry may lose the same batch, which means neither choice can be correct without an explicit event identity and loss policy. The useful drill records the batch identifier, interrupts the connection at several points, repeats delivery, and then searches for every event identifier. It checks two separate claims: that accepted events become searchable within the freshness budget, and that a repeated batch does not corrupt the dashboard's operational meaning. Now consider a queue capped at 10,000 events during a noisy deploy: the policy must say which records are discarded, expose a dropped-event counter, and avoid recursively logging the failure back into the same full queue. Finally, consider a schema deployment in which `duration_ms` changes from an integer to a string: reject it before release or route the incompatible record deliberately, rather than discovering the change when the dashboard query breaks. One exercise covers transport uncertainty, pressure, and producer compatibility without relying on a polished happy-path demo.

These aren't exotic scenarios. They are the API contract under pressure.

Make loss visible.

The operational signals for the logging path are small but non-negotiable: queue depth, oldest queued event age, batches attempted, batches accepted, retries, discarded events, serialization failures, and end-to-end freshness measured from producer time to searchable time. Put those measurements somewhere that does not depend entirely on the path being measured. Otherwise a broken telemetry pipeline reports its own health as silence.

## Compare the architectural options

No option wins every workload. The table records the decision criteria a small backend team should validate in a load test and a failure drill; it does not assign vendor scores.

| Option | Request-path risk | Operational burden | Portability | Appropriate use | Principal limitation |
|---|---|---|---|---|---|
| Direct HTTPS batches through an application adapter | Low when queues and timeouts are bounded | Low to moderate | Moderate to high if the internal event contract stays independent | A startup that wants a short path to centralized search | Each process owns buffering and backpressure unless a shared collector is added |
| Local or sidecar collector | Low after local handoff | Moderate | High when the handoff and event schema remain generic | Multiple runtimes that need consistent batching, redaction, and routing | Another component must be deployed, observed, and capacity-planned |
| Durable queue before indexing | Low when publishing is isolated correctly | High | High at the event boundary | Regulated or high-volume flows where replay is an explicit requirement | More latency, storage, failure modes, and ownership than many early teams need |
| Synchronous per-event calls | High | Superficially low | Low if scattered through business code | Tiny development tools where loss and latency do not matter | Network latency and destination failures leak into application behavior |

For most startup backends, the first option is the narrowest decision that preserves an exit. A local collector becomes attractive when several services have copied queue and retry logic or when policy must be enforced consistently across languages. A durable queue is justified when replay has a stated business purpose and an owner. It shouldn't be installed as a ceremonial guarantee.

Cost still belongs in the review, though not as a headline number. Model bytes ingested, index expansion, retention, query scanning, and the labor required to operate the path. Cardinality is a design concern too: unconstrained identifiers used as indexed dimensions can make the search model difficult to predict. I'm not sure any static estimate remains useful through a launch; the uncertainty is resolved by measuring representative event volume and representative investigations during a trial.

## Critical path in Python

The following code shows the boundary, not a vendor endpoint. Business code creates a typed event; a background worker owns batching and transport; overflow and final delivery failure become explicit counters. Production code would also need a shutdown drain policy, bounded retry timing with jitter, persistent buffering if the loss budget requires it, and a real metrics sink.

```python
from __future__ import annotations

from dataclasses import asdict, dataclass
from datetime import datetime, timezone
from queue import Empty, Full, Queue
from threading import Event, Thread
from typing import Callable
from uuid import uuid4


@dataclass(frozen=True)
class LogEvent:
    event_id: str
    occurred_at: str
    severity: str
    service: str
    environment: str
    event_name: str
    message: str
    correlation_id: str | None


class LogIngestor:
    def __init__(
        self,
        send_batch: Callable[[list[dict[str, object]]], None],
        capacity: int = 10_000,
        batch_size: int = 200,
    ) -> None:
        self._send_batch = send_batch
        self._batch_size = batch_size
        self._queue: Queue[LogEvent] = Queue(maxsize=capacity)
        self._stopping = Event()
        self._worker = Thread(target=self._run, daemon=True)
        self.discarded_events = 0
        self.failed_batches = 0

    def start(self) -> None:
        self._worker.start()

    def emit(
        self,
        severity: str,
        service: str,
        environment: str,
        event_name: str,
        message: str,
        correlation_id: str | None = None,
    ) -> bool:
        event = LogEvent(
            event_id=str(uuid4()),
            occurred_at=datetime.now(timezone.utc).isoformat(),
            severity=severity,
            service=service,
            environment=environment,
            event_name=event_name,
            message=message,
            correlation_id=correlation_id,
        )
        try:
            self._queue.put_nowait(event)
            return True
        except Full:
            self.discarded_events += 1
            return False

    def _run(self) -> None:
        while not self._stopping.is_set():
            batch: list[LogEvent] = []
            try:
                batch.append(self._queue.get(timeout=0.25))
            except Empty:
                continue

            while len(batch) < self._batch_size:
                try:
                    batch.append(self._queue.get_nowait())
                except Empty:
                    break

            try:
                self._send_batch([asdict(event) for event in batch])
            except Exception:
                self.failed_batches += 1
```

Notice what the example refuses to pretend. A failed batch is not automatically requeued forever, because an endless retry loop without a loss budget is an outage multiplier. It also doesn't claim that a returned acknowledgement proves immediate search visibility. The adapter's concrete transport should define connection and read timeouts, authentication, response validation, maximum request size, retryable outcomes, and whether a partial batch can be accepted. Those details should be verified against the selected API and captured in contract tests.

The search side needs its own acceptance test: emit a uniquely identified event, wait within the agreed freshness budget, query by that identifier, and verify the structured fields. Repeat with a duplicate identifier, a late timestamp, a rejected schema, and a redacted sensitive field. A dashboard that looks polished but cannot answer “show errors for this correlation ID during this deployment” is decoration, not an operational tool.

## Rejected option and where it still fits

The rejected default is synchronous, one-event-per-request logging directly from business handlers. The catch is coupling: DNS delay, connection exhaustion, throttling, or slow acknowledgement can consume the same resources serving customers, while scattered transport calls make redaction and migration difficult to audit. It is not suitable when logs are frequent, latency matters, or losing the destination must not affect the application.

Stick with synchronous calls for a disposable internal script or a development-only utility when throughput is tiny, the behavior is visible, and simplicity has more value than isolation. Even there, use a short timeout and keep the call behind an interface. A good rejected option is not forbidden; it has a bounded use case.

Native crashes need a different lane. A normal application logger may never execute after a process-level crash, while a minidump is not an ordinary JSON event. Electron documents `crashReporter` specifically for native crashes and minidumps, so route those artifacts through the crash-reporting mechanism and place only a correlation reference in centralized application logs. This distinction keeps the log schema searchable without pretending that text events can replace crash evidence.

The final selection should therefore be conditional. Use direct asynchronous HTTPS batching behind an owned adapter for the common startup case; add a collector when shared policy and multi-runtime consistency outweigh its operational cost; add durable queuing only when replay and retention are named requirements. Preserve separate ingestion and search contracts, test ambiguous delivery, and require observable loss behavior. The dashboard comes last.

## References

- https://www.electronjs.org/docs/latest/api/crash-reporter
