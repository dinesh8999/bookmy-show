# CONCURRENCY.md

## Goal
Guarantee zero double-bookings while keeping API latency < 500ms and staying within a $2,000/mo AWS budget.

We analyze two real options and then state a defensible choice.

---

## Option A — PostgreSQL row-level locking (SELECT ... FOR UPDATE)

How it works (single-seat example):

BEGIN;
-- 1. lock seat row
SELECT id, status FROM seats
 WHERE id = ANY($1::int[])
 AND status = 'available'
 FOR UPDATE;

-- 2. mark held
UPDATE seats SET status = 'held',
  held_until = NOW() + INTERVAL '10 minutes',
  held_by = $2
 WHERE id = ANY($1::int[]);

-- 3. create booking record
INSERT INTO bookings (id, user_id, event_id, total_amount)
 VALUES ($3, $4, $5);

COMMIT;

Pros:
- No extra infra, ACID provided by Postgres
- Simple to reason about correctness

Cons / Hard limits:
- DB connection pool becomes the bottleneck under very high RPS
- Deadlock risk for multi-seat bookings if locking order varies

Pool exhaustion math (worked example):
- Let max_connections = 500 (PgBouncer)
- Assume two request classes: 80% quick (read+hold path) at avg DB time = 20ms (0.02s), 20% payment-related or heavier ops at avg DB hold = 800ms (0.8s).

ConnectionsHeldPerSecond = RPS * (0.8 * 0.02 + 0.2 * 0.8)
= RPS * (0.016 + 0.16) = RPS * 0.176

Set ConnectionsHeldPerSecond = 500 => RPS_threshold = 500 / 0.176 ≈ 2,841 RPS

Interpretation: with these conservative timings, Postgres-backed locking exhausts the DB pool at ≈2.8k RPS. The expected peak (8k–500k RPS) massively exceeds this. Thus pure Postgres locking cannot handle the hot-path spike.

Deadlock mitigation for multi-seat bookings:
- Acquire locks in a canonical global order (e.g., seat_id ascending) to avoid circular waits.
- Use short transactions and optimistic retries where possible.

---

## Option B — Redis SETNX distributed locks

How it works (single-seat):
- Lock key: `seat_lock:{event_id}:{seat_id}`
- Lock value: `{instance-id}:{timestamp}`
- Acquire with `SET key value NX PX <lock-ms>` (atomic)
- On success: read DB seat status, mark held in DB, publish booking pending
- Release via Lua script that checks value before DEL to avoid deleting someone else's lock

Lua release (pseudocode):
if redis.call('get', KEYS[1]) == ARGV[1] then
  return redis.call('del', KEYS[1])
else
  return 0
end

Pros:
- Extremely fast acquire (sub-ms), scales to 10k+ lock ops/sec per node
- Offloads lock pressure from DB connection pool

Cons:
- Requires Redis cluster (infrastructure + cost)
- Complexity: careful TTL choice and correct release semantics
- Redis failure modes require fallback

TTL selection rationale:
- The lock protects only the critical window where we check DB and write a `held` state. That window should be short — targeted to the DB update latency (∼10–50ms under load). We'll use a **50ms** TTL for SETNX acquisition (fast path) and immediately extend or perform DB write which sets long-lived `held_until` (10 minutes) in DB.
- If the DB write takes longer than TTL, we either extend the Redis lock before expiry (via safe renew with check) or treat the lock-acquire attempt as failed and retry.

Redis failure cases:
- If Redis is unavailable during the sale, the SETNX hot path fails. To preserve correctness we fall back to Postgres `SELECT FOR UPDATE` for that seat as a slower but safe alternative.
- If Redis acknowledges lock but the app crashes before writing DB, the lock TTL ensures it expires quickly, allowing retryers to attempt again. Seat `held` must be the source-of-truth in DB — the Redis lock only serializes attempts.

---

## Hybrid choice (recommended)

We choose a **Hybrid strategy**: Redis SETNX for the hot path (fast acquire), with DB `seats` written to immediately (held state with `held_until`) and Postgres optimistic/transactional confirmation for final booking.

Why hybrid?
- Requirement: 0 double-bookings at 500k concurrency. Pure Postgres locks can't scale to spikes (see math) under $2k budget.
- Redis lets us cheaply serialize the hot-path contention without allocating dozens of DB connections.
- The DB `held` state is the durable source-of-truth. Redis lock is ephemeral and used only to serialize racing attempts.
- If Redis fails or lock cannot be acquired, we fallback to Postgres `SELECT FOR UPDATE` as a correctness-first degraded mode.

What can't this strategy handle?
- Unbounded global spikes beyond the capacity of the network and the chosen EC2/Redis instances. We dimension Redis/EC2 for a 1-minute rolling peak and rely on CloudFront/ALB rate-limits and queuing to smooth traffic.

Switch conditions to pure Postgres (or different infra):
- If budget increases and we can afford a high-throughput RDS cluster with horizontally-scaled write capacity and higher connection limits, the design could shift.
- If Redis cannot be provisioned, system falls back to Postgres `FOR UPDATE` with stricter admission control at ALB to avoid DB meltdown.

---

## Operational notes
- Lock key format: `seat_lock:{event_id}:{seat_id}` to avoid cross-event interference and to enable sharding by event namespace.
- Lock TTL: short (50ms) for acquisition; DB `held_until` controls user charge window (10 minutes).
- Always use an atomic release (Lua script) and an owner value to prevent accidental delete.
- Monitor Redis latency, lock failures, and fallbacks to Postgres. Add alarms for lock acquisition failures during sale.
