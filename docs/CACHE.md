# CACHE.md

## Goals
- Reduce DB read load during peak sale.
- Keep user-facing latency < 500ms.
- Avoid caching data that would introduce correctness holes (no caching of per-seat availability state).

## Cache entries (design)

1) Event details (static metadata)
- Key format: `event:{event_id}:meta`
- Value: JSON { id, name, venue_id, starts_at, status, artist, total_seats }
- TTL: 3600s (1 hour)
- Invalidation: on `events` UPDATE/patch (update or delete key). For event lifecycle changes (status change to `on_sale`, `cancelled`), invalidate immediately.
- Rationale: rarely changing and safe to serve cached copy.

2) Seat availability count per event+category
- Key format: `availability:{event_id}:{category}`
- Value: integer count
- TTL: 30s
- Invalidation: targeted delete on any seat status change for that event+category (seat held/booked/released). The update flow deletes the key; next read repopulates via DB.
- Rationale: counts can be slightly stale but should converge fast. 30s keeps read/write traffic reasonable while providing near-real-time UX.

3) Static seat map layout
- Key format: `seatmap:{event_id}`
- Value: JSON structure describing sections, rows, seat numbers, categories, and base prices
- TTL: 86400s (24 hours)
- Invalidation: on event cancellation or venue layout change only.
- Rationale: layout is effectively static for an event and safe to cache long-term.

## What we DO NOT cache
- Individual seat `status` (A-12 available/booked): NOT cached, except as derived from DB or Redis lock state at request time.
- Reason: caching individual seat availability risks stale reads and windowed double-bookings. Concurrency must be solved with locks, not cache coherence.

## Invalidation strategy
- Use cache-aside (read-through is optional): on reads, check Redis; on miss, query DB and set key with TTL.
- On writes (seat status change), perform DB write first (source-of-truth), then delete relevant cache keys. Do NOT perform blind cache writes for seat counts — let the next reader repopulate.

Pseudocode: seat status change (hold/book/release)

async function updateSeatStatus(seatId, eventId, category, newStatus) {
  const client = await db.connect();
  try {
    await client.query('BEGIN');
    await client.query('UPDATE seats SET status=$1, held_until=$2, held_by=$3, version=version+1 WHERE id=$4',
      [newStatus, heldUntilOrNull, heldByOrNull, seatId]);

    // targeted invalidation
    await redis.del(`availability:${eventId}:${category}`);

    await client.query('COMMIT');
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}

## TTL justification
- `availability` 30s: balances freshness and Redis write cost. We also delete key immediately on any status change so TTL is a safety net for missed invalidations.
- `event meta` 1h: event metadata changes infrequently; longer TTL reduces DB reads.
- `seatmap` 24h: static layout; very low churn.

## Operational considerations
- Use targeted invalidation to avoid large-scale cache flushes during the sale.
- Track cache hit ratios and tail latencies. If availability count misses create heavy DB load during peaks, increase pre-warming rate shortly before sale.
- For super-hot events, consider using an in-memory precomputed availability snapshot sharded per API instance (but ensure it is eventually consistent via invalidation). Prefer Redis for centralized caching.
