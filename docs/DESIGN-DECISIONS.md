# Design Decisions

## Decision: Redis SETNX for seat locking, not PostgreSQL FOR UPDATE

**Context:**
The sale can spike to 500,000 concurrent users, while the API must stay under 500ms and the monthly budget is capped at $2,000.

**Options considered:**
1. PostgreSQL `SELECT ... FOR UPDATE` only - simplest correctness story, but the database connection pool becomes the bottleneck before the sale traffic does.
2. Optimistic locking only - cheap, but under a concentrated sale it produces too many retries and pushes contention back onto PostgreSQL.
3. Redis `SETNX` distributed locking - chosen because it serializes the hot path without tying up a database connection for each hold attempt.

**Why chosen:**
Redis gives sub-millisecond lock acquisition and keeps PostgreSQL focused on durable writes instead of contention management. The DB still stores `held_until`, `held_by`, and `version`, so correctness does not depend on Redis being permanently available.

**Tradeoffs accepted:**
Redis adds operational complexity and an extra failure mode. If Redis is unavailable, the system must fail closed or fall back to a slower safe path; it cannot silently continue without a lock.

**Revision trigger:**
If sustained traffic drops far below the current peak, or if the database tier is upgraded enough to absorb the lock contention comfortably, PostgreSQL row locking becomes the simpler design.

## Decision: Event-driven cache invalidation with TTL as a safety net

**Context:**
The system caches event metadata and availability counts, but stale availability can mislead users during a sale.

**Options considered:**
1. TTL-only invalidation - easiest to implement, but stale seat counts can remain wrong for too long during a flash sale.
2. Event-driven invalidation plus TTL fallback - chosen because writes delete the affected cache key immediately, while the TTL guarantees eventual recovery if an invalidation is missed.
3. Write-through cache updates on every seat change - considered, but it turns every seat update into a cache mutation path that is harder to reason about under partial failures.

**Why chosen:**
Targeted deletion keeps availability counts fresh after holds, releases, and bookings, while the TTL stops a missing invalidation from freezing a bad value indefinitely. This is a better fit than TTL-only because the visible sale window is short and freshness matters more than pure cache simplicity.

**Tradeoffs accepted:**
Availability counts remain approximate between invalidation and the next read repopulation. That is acceptable because the booking commit still rechecks the primary database under lock.

**Revision trigger:**
If the business required strict real-time browse counts on every screen, we would need a stronger invalidation or a different data model, not just shorter TTLs.

## Decision: UUIDs for booking IDs instead of SERIAL

**Context:**
Bookings must be idempotent across retries, and the system is distributed across API servers and workers.

**Options considered:**
1. SERIAL - simple, but predictable identifiers are easy to enumerate and are awkward for client-side idempotency.
2. UUID - chosen because the client or API can generate the identifier before the booking finishes, which makes retries safe.
3. Snowflake-style numeric IDs - scalable, but they add a second coordination mechanism that is unnecessary for this workload.

**Why chosen:**
UUIDs support idempotent create flows and avoid sequential leakage of booking volume. They also work cleanly across the API tier and the asynchronous worker tier without a central ID allocator.

**Tradeoffs accepted:**
UUIDs are larger indexes and slightly less cache-friendly than sequential integers. That cost is acceptable because correctness and retry safety matter more than compactness here.

**Revision trigger:**
If booking volume grows enough that UUID index size becomes a measurable storage or cache problem, we would revisit the ID strategy.

## Decision: SQS visibility timeout of 60 seconds

**Context:**
Payments are asynchronous and can involve gateway latency, retries, and transient network errors.

**Options considered:**
1. Short timeout such as 20 seconds - too aggressive for gateway slowness and likely to cause duplicate work.
2. 60 seconds - chosen because it covers the normal payment round trip plus a retry window without holding messages invisible for too long.
3. Very long timeout such as several minutes - reduces duplicate delivery but makes recovery from a dead worker too slow.

**Why chosen:**
60 seconds is long enough for a worker to finish a payment attempt and short enough that failed or crashed workers do not strand messages for too long. The DLQ after 3 receives gives the system a clean path for investigation.

**Tradeoffs accepted:**
Some slow payment attempts will be retried instead of waiting forever. That is an acceptable cost because the worker and payment gateway both use idempotency checks.

**Revision trigger:**
If the gateway latency profile changes materially, this timeout should be revisited alongside worker retry policy and DLQ thresholds.

## Decision: Read replicas for browse traffic, primary for booking correctness

**Context:**
The system needs to scale read-heavy browse traffic without making the actual booking path stale or unsafe.

**Options considered:**
1. Read everything from the primary - simplest, but wastes the primary on cacheable reads.
2. Route browse reads to replicas and keep booking writes on the primary - chosen because it scales better and preserves correctness where it matters.
3. Synchronous replication - considered, but the write latency penalty is too high for this sale workload.

**Why chosen:**
Event pages, seat maps, and booking history can tolerate small staleness. Booking confirmation cannot, so it always uses the primary and the locking path.

**Tradeoffs accepted:**
Replica lag can show slightly stale availability counts. That is a UX issue, not a double-booking issue, because the commit path rechecks the primary.

**Revision trigger:**
If browse reads need to become strongly consistent, the caching and database access patterns would need to change together.