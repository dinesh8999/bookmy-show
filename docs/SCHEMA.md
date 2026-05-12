# SCHEMA.md

## Complete PostgreSQL schema (DDL)

-- VENUES
CREATE TABLE venues (
  id            SERIAL PRIMARY KEY,
  name          VARCHAR(255) NOT NULL,
  city          VARCHAR(100) NOT NULL,
  capacity      INT NOT NULL CHECK (capacity > 0),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_venues_city ON venues(city);

-- EVENTS
CREATE TABLE events (
  id            SERIAL PRIMARY KEY,
  name          VARCHAR(255) NOT NULL,
  venue_id      INT NOT NULL REFERENCES venues(id) ON DELETE RESTRICT,
  starts_at     TIMESTAMPTZ NOT NULL,
  total_seats   INT NOT NULL CHECK (total_seats > 0),
  status        VARCHAR(20) NOT NULL DEFAULT 'up_sale'
                  CHECK (status IN ('upcoming','on_sale','sold_out','cancelled')),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_events_starts ON events(starts_at);
CREATE INDEX idx_events_status ON events(status);

-- SEATS
CREATE TABLE seats (
  id            SERIAL PRIMARY KEY,
  event_id      INT NOT NULL REFERENCES events(id) ON DELETE CASCADE,
  section       VARCHAR(50) NOT NULL,
  row_label     VARCHAR(10) NOT NULL,
  number        INT NOT NULL,
  category      VARCHAR(30) NOT NULL,
  price         DECIMAL(10,2) NOT NULL CHECK (price > 0),
  status        VARCHAR(20) NOT NULL DEFAULT 'available'
                  CHECK (status IN ('available','held','booked')),
  held_until    TIMESTAMPTZ,
  held_by       UUID,
  version       INT NOT NULL DEFAULT 0,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (event_id, section, row_label, number)
);
CREATE INDEX idx_seats_event_status ON seats(event_id, status);
CREATE INDEX idx_seats_event_category ON seats(event_id, category);

-- USERS
CREATE TABLE users (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email         VARCHAR(255) UNIQUE NOT NULL,
  phone         VARCHAR(20) UNIQUE,
  name          VARCHAR(100) NOT NULL,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- BOOKINGS
CREATE TABLE bookings (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
  event_id      INT NOT NULL REFERENCES events(id) ON DELETE RESTRICT,
  status        VARCHAR(20) NOT NULL DEFAULT 'pending'
                  CHECK (status IN ('pending','confirmed','failed','refunded')),
  total_amount  DECIMAL(10,2) NOT NULL CHECK (total_amount >= 0),
  payment_ref   VARCHAR(100),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  confirmed_at  TIMESTAMPTZ
);
CREATE INDEX idx_bookings_user ON bookings(user_id, created_at DESC);
CREATE INDEX idx_bookings_event ON bookings(event_id);
CREATE INDEX idx_bookings_status_pending ON bookings(status) WHERE status IN ('pending','failed');

-- BOOKING_SEATS
CREATE TABLE booking_seats (
  booking_id  UUID NOT NULL REFERENCES bookings(id) ON DELETE CASCADE,
  seat_id     INT NOT NULL REFERENCES seats(id) ON DELETE RESTRICT,
  PRIMARY KEY (booking_id, seat_id)
);
CREATE INDEX idx_bs_seat ON booking_seats(seat_id);

## Commentary: design rationales

### Why `UUID` for `bookings.id` and `users.id` instead of SERIAL?
- Predictability: SERIAL is sequential and can be enumerated; UUIDs prevent trivial enumeration and leakage of system scale.
- Idempotency: clients can generate UUIDs before API calls; `INSERT ... ON CONFLICT DO NOTHING` lets retries be safe.
- Horizontal friendliness: UUIDs avoid coordination when writing from multiple regions/services.

### Why `seats.version` (optimistic locking)?
- Enables optimistic concurrency control: update queries check `version = :old` and `SET version = version + 1`.
- Avoids long-held database locks on the confirmation path.
- Detects write-write conflicts when multiple actors try to book the same seat without extra distributed locks.

### Why `held_until` on seats instead of only application-level holds?
- Database is the source-of-truth for seat holds: preventing race windows where a server crashes after acknowledging a hold.
- Background jobs can reclaim expired holds by scanning `seats WHERE status='held' AND held_until < now()`.
- Combining DB-held timestamps with external locks (Redis) provides both throughput and correctness.

### Why a partial index on `bookings.status`?
- The common operational queries target unresolved bookings (`pending`,`failed`). A partial index keeps index size small and lookup fast for workers.
- Historical `confirmed` bookings are large but rarely scanned by the payment worker, so excluding them saves IO and memory.

## Seat-release flow on payment failure (summary)
1. `UPDATE bookings SET status='failed' WHERE id = :bookingId;`
2. `UPDATE seats SET status='available', held_until = NULL, held_by = NULL WHERE id IN (:seatIds);`
3. All changes are performed in a single DB transaction to avoid partially-released seats.


