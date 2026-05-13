# Design Updates

## Update 1: Per-user seat hold limit

**Triggered by:** Panel Question 3 - one user trying to hold 200 seats across many tabs.

**What changed:**
Added a Redis counter keyed by user ID, `holds:{user_id}:count`, with a TTL of 15 minutes. The API checks this counter before seat lock acquisition and returns `429 Too Many Requests` when the count reaches 8 concurrent holds. The counter increments when a hold succeeds and decrements when the hold is released or confirmed.

**Why this is necessary:**
Without this guard, a single user can monopolize a large chunk of inventory during the sale window even if the lock itself is correct. That is an abuse problem, not a locking problem.

**What it costs:**
One extra Redis read on the hold path, plus a small amount of state management on release and confirmation.

**What it still doesn't solve:**
It limits abuse from one account or session, but a coordinated attacker with many accounts can still spread the pressure. Stronger prevention would need product-level identity checks.

## Update 2: SQS publish circuit breaker with booking status polling

**Triggered by:** Panel Question 4 - SQS outage or prolonged publish failures during an active sale.

**What changed:**
The API now tracks SQS publish failures and trips a circuit breaker after 60 seconds of continuous failure. When that happens, the API falls back to synchronous payment processing at reduced throughput instead of returning a fake `202` forever. The booking API also exposes `GET /bookings/:id` so the client can poll and show `pending`, `confirmed`, or `failed` while the worker is still processing.

**Why this is necessary:**
If the queue publish path is unavailable, users need a truthful answer about whether the booking is still in progress. The polling endpoint and fallback mode prevent silent black holes.

**What it costs:**
Additional operational logic in the API tier, one more status lookup for the client, and slower throughput during fallback mode.

**What it still doesn't solve:**
The synchronous fallback cannot match the throughput of the normal queue-based path. It is a degraded safety mode, not a permanent replacement.

## Update 3: Peak-sale hold expiry shortened

**Triggered by:** Panel Question 1 and the overall concurrency roast - keeping seats unavailable too long makes abuse and stale holds worse.

**What changed:**
During active sale events, the seat hold expiry is reduced from 10 minutes to 3 minutes while `held_until` still remains the source of truth in PostgreSQL.

**Why this is necessary:**
Shorter holds release abandoned carts faster and reduce the window in which inventory is artificially locked.

**What it costs:**
Users have less time to complete checkout, so the UX is stricter during the peak sale period.

**What it still doesn't solve:**
It does not eliminate abuse or guarantee payment completion. It only reduces the time seats stay unavailable after a user abandons the flow.