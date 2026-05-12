# Constraints and Pre-design Notes

This file captures the constraint analysis required before writing architecture docs.

## Hard constraints
- Concurrent users: 500,000 hitting "Book Now" simultaneously.
- Double-bookings: zero tolerance.
- Monthly AWS budget: $2,000.
- Max API latency: 500ms.

## Peak RPS calculations
- Worst-case instantaneous spike: 500,000 requests in 1 second => 500,000 RPS.
- More realistic spreading over first minute: 500,000 / 60 ≈ 8,333 RPS.

## Which component hits limits first?
- Database connection pool (Postgres) is the most constrained resource if synchronous payment or DB-level locking is used for every request.
- A 500-connection pool with typical query times (~20ms) supports only a few thousand RPS before saturation.

## Summary impact on design choices
- Use Redis to offload hot-path locking and reduce DB contention.
- Use async payment processing (SQS + workers) to avoid long-lived DB connections during payment.
- Cache event metadata and availability counts to limit DB reads.

