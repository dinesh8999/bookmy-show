# QUEUE.md

## Why async payments?

Synchronous payment calls block DB connections and application threads for the payment round-trip (200–2000ms). At scale this exhausts the DB pool and collapses throughput.

Re-using the connection pool formula from CONCURRENCY.md shows Postgres exhaustion at ~2.8k RPS for common timings; synchronous payment calls magnify this and are therefore unacceptable.

We therefore push the payment to an async queue (SQS). API servers do the minimal durable work (hold seats + create `pending` booking) and return quickly.

---

## SQS message format (JSON)

{
  "bookingId": "<uuid>",
  "userId": "<uuid>",
  "eventId": <int>,
  "seatIds": [<int>, ...],
  "totalAmount": 0.00,
  "paymentToken": "<payment-provider-token>",
  "idempotencyKey": "<uuid-or-client-generated>",
  "createdAt": "<ISO8601>",
  "callbackUrl": "<optional-callback>"
}

Field reasons:
- `bookingId`: primary key for idempotency and final DB updates.
- `userId`: for notifications and secondary checks.
- `eventId`, `seatIds`: worker updates seat state and posts notifications without extra DB lookups if possible.
- `totalAmount`: allow basic validation and dispute checks.
- `paymentToken`: tokenized payment info from client (or payment gateway reference).
- `idempotencyKey`: protect from double-processing due to retries.
- `createdAt`: for observability and metrics.

---

## Worker logic (numbered steps)

1. Worker pulls message from SQS (long polling).
2. Worker validates message schema and checks idempotency (e.g., `SELECT status FROM bookings WHERE id = :bookingId`).
  - If `status == 'confirmed'` -> delete message (idempotent success).
  - If `status == 'failed'` -> treat according to retry policy or move to DLQ after max attempts.
  - If `status == 'pending'` -> proceed.
3. Call payment gateway with `paymentToken` and `totalAmount`.
  - Use provider SDK with per-request idempotency key derived from `idempotencyKey`.
4a. On SUCCESS response from gateway:
  - Begin DB transaction.
  - UPDATE bookings SET status='confirmed', payment_ref = <provider_txn>, confirmed_at = NOW() WHERE id = :bookingId AND status = 'pending';
  - UPDATE seats SET status='booked', held_until = NULL, held_by = NULL WHERE id IN (:seatIds) AND status IN ('held','available');
    - Use optimistic checks to ensure seat wasn't double-booked by a concurrent process; if conflict, mark booking as 'failed' and release seats accordingly.
  - Commit transaction.
  - Send confirmation notifications (SMS/email).
  - Delete SQS message.
4b. On FAILURE (gateway responds with definitive failure):
  - Begin DB transaction.
  - UPDATE bookings SET status='failed' WHERE id = :bookingId AND status = 'pending';
  - UPDATE seats SET status='available', held_until = NULL, held_by = NULL WHERE id IN (:seatIds);
  - Commit.
  - Send failure notification.
  - Delete SQS message.
4c. On TIMEOUT or ambiguous responses (network error, gateway timeout):
  - Do not delete the SQS message; allow it to become visible again after VisibilityTimeout.
  - Implement an exponential backoff retry policy (SQS redrive policy + worker-local throttling).

---

## SQS configuration
- VisibilityTimeout: **60 seconds**. Rationale: 2× expected worst-case payment handling to allow retries by same worker without duplication, while not holding messages indefinitely.
- MaxReceiveCount (before DLQ): **3**. Rationale: three retries is a pragmatic balance between transient gateway issues and manual investigation.
- Dead Letter Queue (DLQ): monitored by CloudWatch/alerting. When a message lands in DLQ, create a PagerDuty/Slack alert and surface the booking for manual inspection.

---

## Edge cases

1) API server crashes after publishing to SQS but before responding to user
- Outcome: The booking and SQS message exist. User may not receive the 202 response, but the worker will process the payment and final notification will be delivered. API is idempotent thanks to client-generated `bookingId`.

2) Payment gateway times out (neither success nor failure)
- Worker treats as transient error and does not delete message; message becomes visible after VisibilityTimeout for retry.
- Use a combination of SQS retries (up to `MaxReceiveCount`) and provider-side idempotency keys to ensure the gateway does not charge the user twice.

3) Worker processes message but fails after DB write (e.g., crash before deleting SQS message)
- If worker crashed after marking booking `confirmed` but before deleting the SQS message, a subsequent worker reading the message will see `status=='confirmed'` and safely delete the message (idempotent).

---

## Observability and monitoring
- Track SQS queue length, age of oldest message, DLQ counts.
- Track payment worker error rates and payment gateway latency percentiles.
- Alert when >1% of messages start failing or when DLQ receives messages.

