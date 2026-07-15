# review-code — conditional domain modules

Loaded per Procedure step 1: run only the modules the diff actually touches. Same contract as
[`checks.md`](checks.md) — every finding names the canonical concept and a named fix; attribution in
parentheses is the primary source (published canon). Glossed entries (marked *gloss*) are concepts
models know unevenly — carry the one-line definition into the finding.

## Contents
- [Concurrency & async](#concurrency--async)
- [Database & persistence](#database--persistence)
- [Distributed systems & messaging](#distributed-systems--messaging)
- [API & contracts](#api--contracts)
- [Observability & operations](#observability--operations)

## Concurrency & async
*Select when the diff touches threads, locks, shared state, async/await, promises, or event loops.*

| Check | Flags | Named fix |
|---|---|---|
| **Check-then-act race** (atomicity violation; TOCTOU — CWE-367) | read-decide-write on shared state without atomicity (`if not exists: create`, balance checks, file checks before use) | atomic compare-and-swap / upsert / unique constraint; do the operation and handle the conflict |
| **Shared mutable state** (Goetz, *Java Concurrency in Practice*) | new shared collections/fields mutated from multiple threads/tasks without confinement | immutability; confinement to one owner; message passing; otherwise a documented lock |
| **Lock discipline** (Goetz, *JCiP*) | inconsistent lock acquisition order (deadlock); locks held across I/O; double-checked locking without memory barriers | one documented lock order; shrink critical sections to exclude I/O; use the language's safe lazy-init idiom |
| **Async pitfalls** (Cleary, *Async/Await — Best Practices in Asynchronous Programming*) | fire-and-forget tasks with no error handler; unawaited promises/futures; sync-over-async blocking (`.Result`, `run_until_complete` in a handler); CPU/blocking work on the event loop | await or supervise every task; async all the way down; move blocking work to a worker pool |
| **Thread-safety as contract** (Goetz, *JCiP* — document thread safety) | a published type used concurrently with no statement of its thread-safety | document (and test) the type's concurrency contract; make it immutable if possible |

## Database & persistence
*Select when the diff touches queries, ORM code, transactions, schema, or migrations.*

| Check | Flags | Named fix |
|---|---|---|
| **N+1 Query** (Fowler PoEAA lazy-load hazard) | a per-row query inside a loop over a result set; ORM lazy relations iterated | eager-load/JOIN/batch (`IN` list); PoEAA Identity Map/Unit of Work idioms via the ORM |
| **Unbounded query** (Nygard Unbounded Result Sets) | `SELECT` without LIMIT/pagination on a growing table; `SELECT *` in production code | paginate (prefer keyset/cursor over offset); name the columns |
| **Missing index on a new path** (Winand, *SQL Performance Explained*) | a new query filtering/ordering by columns with no index, on a table expected to grow | add the index in the same change, or record why not |
| **Transaction scope** (Nygard, *Release It!*; Kleppmann for the isolation anomalies) | network/file I/O inside a DB transaction; transactions spanning user interaction; long-running transactions holding locks | shrink the transaction to the writes; do I/O outside; note the isolation anomaly risked (dirty/non-repeatable/phantom read, lost update; *gloss:* write skew — two txns each validate against state the other invalidates) |
| **Optimistic vs pessimistic locking** (Fowler PoEAA Offline Lock patterns) | read-modify-write across requests with no version field or lock — lost updates | Optimistic Offline Lock (version column + conflict error); pessimistic only when conflicts are the norm |
| **Expand–contract migrations** (Sato, *ParallelChange*; Ambler & Sadalage, *Refactoring Databases*) | a destructive migration (drop/rename column) in one step while old code still reads it; data backfill mixed into the schema step | expand → migrate readers → contract, as separate deploys; backfill as its own reversible step; state the rollback plan |

## Distributed systems & messaging
*Select when the diff touches service-to-service calls, queues/streams, caches, retries, or locks
across processes.*

| Check | Flags | Named fix |
|---|---|---|
| **Timeout on every I/O call** (Nygard Integration Points) | a new network call with no timeout; timeouts longer than the caller's own deadline | explicit timeout budget per hop; caller deadline > sum of callee retries (avoid retry storms) |
| **Retry with idempotency** (Nygard; Hohpe & Woolf Idempotent Receiver) | retrying a non-idempotent operation; "exactly-once" claimed anywhere | idempotency keys / Inbox (Idempotent Receiver); design consumers for at-least-once; treat exactly-once claims as a red flag |
| **Outbox Pattern** (Richardson, *Microservices Patterns*) | a DB write and a message publish as two non-atomic steps (dual-write) | Outbox — write the event in the same transaction, publish from the outbox |
| **Poison messages & DLQ** (Hohpe & Woolf, *EIP* Dead Letter Channel) | a new consumer with no dead-letter/parking path — one bad message halts the queue | DLQ + bounded retries with exponential backoff and jitter |
| **Cache stampede** (Vattani et al., *Optimal Probabilistic Cache Stampede Prevention*, VLDB 2015; a.k.a. dogpile) | cache-aside on a hot key with a fixed TTL and no stampede protection | TTL jitter; single-flight/request coalescing; soft-TTL refresh |
| **Distributed lock honesty** (Kleppmann, *How to do distributed locking*; *DDIA* ch.8 fencing tokens) | a lock/lease used across processes with no expiry or fencing reasoning (*gloss:* fencing token — a monotonically increasing number the resource checks so a stale lock-holder's writes are rejected — Kleppmann) | lease with expiry + fencing token; or restructure so the invariant is enforced by the datastore |
| **Ordering & backpressure** (Hohpe & Woolf Correlation Identifier; Nygard load shedding) | cross-service ordering assumed but not guaranteed; an unbounded in-memory queue between producers and consumers | Correlation Identifier + per-key ordering where the broker guarantees it; bounded queues + load shedding |

## API & contracts
*Select when the diff changes a published interface: HTTP/RPC endpoints, events, library exports.*

| Check | Flags | Named fix |
|---|---|---|
| **Backward compatibility** (semantic versioning; Hyrum's Law) | a breaking change — removed/renamed field, changed type/semantics — to a published contract without a major version/deprecation path; remember Hyrum's Law: *all* observable behaviour is contract | additive-only evolution; deprecate-then-remove; version the endpoint/event |
| **Pagination mandatory** (Nygard Unbounded Result Sets, applied at the contract) | a new collection endpoint returning the full set | cursor/keyset pagination from day one (offset breaks under mutation) |
| **Idempotency keys for mutations** (Hohpe & Woolf, *EIP* Idempotent Receiver; Stripe's idempotency-key convention) | an unsafe retriable mutation (payment, order) with no idempotency key | accept an idempotency key; return the original result on replay |
| **Concurrency control** (HTTP conditional requests — RFC 7232; Fowler Optimistic Offline Lock) | update endpoints with last-write-wins semantics | ETag/If-Match or a version field (optimistic concurrency) |
| **Error contract consistency** (RFC 7807 Problem Details) | ad-hoc error shapes per endpoint; internals leaked in error bodies | one machine-readable error shape (e.g. RFC 7807 Problem Details — *gloss:* a standard JSON error body with type/title/status) |
| **Tolerant Reader** (Fowler — *gloss:* consumers ignore unknown fields rather than failing) | a consumer that hard-fails on any unrecognized field of a contract it doesn't own | read only what you need; ignore unknowns; validate what you use |

## Observability & operations
*Select when the diff adds failure paths, external calls, background work, or user-facing errors.*

| Check | Flags | Named fix |
|---|---|---|
| **New failure path, no signal** (Beyer et al., *Site Reliability Engineering* — Golden Signals; Wilkie — RED method) | a new error branch or external call with no log/metric/trace — invisible in production | structured log with context + a metric on the failure path; propagate the trace/correlation ID |
| **Actionable errors** (Nygard, *Release It!* — Transparency) | error messages that don't say what failed, why, and what to do next | rewrite the message with those three parts; include identifiers, not payloads |
| **Log hygiene** (Beyer et al., *Site Reliability Engineering*; OWASP logging guidance for PII) | logging in a hot loop; whole payloads logged; PII/secrets in logs or traces; wrong levels (errors as info) | log once per operation at the boundary; log IDs and sizes, not bodies; redact; level by actionability |
| **Cardinality control** (Beyer et al., *Site Reliability Engineering*; Prometheus instrumentation guidance — avoid high-cardinality labels) | a metric labeled by user ID/URL/request ID — unbounded label values explode the metrics store | label by bounded dimensions (route, status, tenant tier); IDs go in traces/logs |
| **Config & flags hygiene** (12-Factor; feature-flag debt) | config/credentials hardcoded rather than environment-injected; a new feature flag with no cleanup plan; nested flags | config in the environment; flag with an owner + removal condition recorded |
