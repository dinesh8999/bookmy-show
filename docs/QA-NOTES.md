# QA Notes

These notes capture the five panel questions and the answers I would give live. They are written as concise defense notes rather than a transcript.

## 1. What happens if Redis crashes during a seat hold?

**Answer:**
I would restate that Redis is the hot-path lock, not the source of truth. If the cluster is partially degraded, the request fails closed instead of proceeding without a lock. If Redis is fully unavailable, the API does not silently skip locking; it either returns a temporary failure or falls back to the slower PostgreSQL safety path. The durable seat state still lives in PostgreSQL with `held_until` and `version`, so the system does not double-book even if Redis is down.

**Assessment:**
Complete enough for the panel. The only limitation is throughput, not correctness.

## 2. How do you prevent a user from holding 200 seats?

**Answer:**
I would say this is abuse control, not just concurrency control. The API checks a Redis counter keyed by user ID before acquiring any seat lock. If the user already has too many active holds, the API returns 429 and does not even touch PostgreSQL. I would also shorten active-sale hold windows so abandoned seats are released faster, and tie payment completion to the same session token that acquired the hold.

**Assessment:**
Complete. It does not stop a determined attacker with many accounts, but it does stop the single-user multi-tab abuse case.

## 3. What if read replicas lag by 500ms and show the wrong availability?

**Answer:**
I would separate browse correctness from booking correctness. Replica lag can make a page show slightly stale seat counts, but the real booking commit never trusts a replica. When the user clicks Book, the API rechecks the primary under the lock path and either confirms the hold or returns an already-taken error. So stale availability is a UX issue, not a consistency failure.

**Assessment:**
Complete. The design accepts approximate browse counts, not approximate booking commits.

## 4. Why use Redis SETNX instead of PostgreSQL FOR UPDATE?

**Answer:**
SETNX is there to keep the hot path off the database connection pool. PostgreSQL row locks are correct, but at peak sale traffic they put too much pressure on the primary and the pool. Redis serializes the contention cheaply, while PostgreSQL still stores the durable `held_until`, `held_by`, and booking rows. I would also mention that the database still has optimistic version checks as a correctness backstop.

**Assessment:**
Complete. The tradeoff is complexity in exchange for lower contention and better peak handling.

## 5. Where is your CDN in this diagram?

**Answer:**
CloudFront sits in front of the ALB and serves static assets plus cached event pages. It does not own booking correctness. It only absorbs repeat browser traffic and passes dynamic `/api/*` calls through to the application tier. That keeps the origin focused on the sale logic while reducing load on the ALB and API servers.

**Assessment:**
Complete. It is a performance layer, not a consistency layer.