# Architecture

This diagram is the end-to-end system view for the BookMyShow sale flow. Every arrow is labeled with the data that moves across it and the reason it exists.

```text
Mobile / Browser
      │
      ▼
CloudFront CDN  ── serves static assets + cached event pages
      │
      ├── cache hit: HTML/CSS/JS, event landing pages
      └── cache miss / API pass-through: /api/* requests
      │
      ▼
Application Load Balancer (ALB)
  - SSL termination
  - Health checks every 10s
  - Rate limit: 200 req/IP/min
      │
      ▼
Node.js API Auto-Scaling Group (4-20 instances)
  - reads Redis availability counts + event metadata
  - acquires seat locks with SETNX
  - writes durable seat/booking state to PostgreSQL primary
  - publishes payment jobs to SQS
  - checks per-user hold counter before lock acquisition
      │
      ├── READ / LOCK ───────────────────────────────────────────────► Redis Cluster (3 nodes)
      │                                                              - availability:{event_id}:{category} TTL 30s
      │                                                              - event:{event_id}:meta TTL 3600s
      │                                                              - seat_lock:{event_id}:{seat_id} SETNX, TTL 50ms
      │                                                              - holds:{user_id}:count TTL 15m, max 8 concurrent holds
      │                                                              ← Redis SETNX lock + cache TTL strategy (see docs/CONCURRENCY.md, docs/CACHE.md)
      │
      ├── WRITE ────────────────────────────────────────────────────► PostgreSQL Primary (RDS)
      │                                                              - booking rows, held_until, held_by, seat version checks
      │                                                              - all correctness-sensitive writes land here
      │                                                              ← Durable source of truth for booking state (see docs/SCHEMA.md)
      │
      │                                                              ┌──────────────────────────────┐
      │                                                              │ Replication stream           │
      │                                                              └──────────────┬───────────────┘
      │                                                                             ▼
      │                                                              Read Replica 1      Read Replica 2
      │                                                              - event details      - seat map browse reads
      │                                                              - user booking hist. - non-transactional availability reads
      │                                                              ← Read scaling for browse traffic (see docs/CACHE.md, docs/SCHEMA.md)
      │
      └── PUBLISH ──────────────────────────────────────────────────► SQS Payment Queue
                                                                     message JSON:
                                                                     bookingId, userId, eventId, seatIds,
                                                                     totalAmount, paymentToken, idempotencyKey, createdAt
                                                                     visibility timeout: 60s
                                                                     max receive count: 3
                                                                     DLQ: payment-dlq
                                                                     ← Async payment path to avoid holding DB connections open (see docs/QUEUE.md)
                                                                              │
                                                                              ▼
                                                                    Payment Worker (ECS Fargate) ×10
                                                                    1. long-poll SQS and validate message
                                                                    2. call external payment gateway
                                                                    3. confirm or fail booking in PostgreSQL primary
                                                                    4. publish booking-confirmed to SNS
                                                                    5. delete SQS message on success or terminal failure
                                                                    ← Idempotent worker flow and final booking confirmation (see docs/QUEUE.md, docs/SCHEMA.md)
                                                                              │
                                                                              ▼
                                                                            AWS SNS
                                                                           ├──► SES email
                                                                           └──► SMS via Twilio/SNS SMS
                                                                                trigger: worker publishes after confirmed booking
                                                                                ← Confirmation fan-out after payment success (see docs/QUEUE.md)
```

## How the data flows

1. Browser requests first hit CloudFront, which serves static assets and cached event pages.
2. Dynamic API traffic passes through the ALB to the Node.js API tier.
3. The API tier reads hot availability data from Redis, then acquires a seat lock with SETNX before touching the database.
4. The API writes the durable hold or booking state to PostgreSQL primary and publishes payment work to SQS.
5. Payment workers consume SQS messages, call the gateway, update PostgreSQL primary, and then publish SNS notifications.
6. Read replicas are used only for non-transactional browse traffic, never for the actual booking commit path.

## Part A references

- Redis SETNX locking and fallback behavior come from docs/CONCURRENCY.md.
- TTL-based cache-aside for event metadata and availability counts comes from docs/CACHE.md.
- The booking, seat, and booking_seats schema comes from docs/SCHEMA.md.
- The async payment queue and worker idempotency flow come from docs/QUEUE.md.

## Post-roast updates reflected here

- Per-user seat hold counter in Redis: `holds:{user_id}:count` with a max of 8 concurrent holds.
- SQS publish circuit breaker: if the queue publish path stays broken long enough, the API degrades to a synchronous fallback instead of silently returning 202 forever.