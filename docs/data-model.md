# Data model

Postgres 17 on Neon. Extensions: `btree_gist` (exclusion constraints on `uuid` + ranges; supported on Neon PG14 through PG18).

Conventions:

- Primary keys are `uuid DEFAULT gen_random_uuid()`.
- Times are `timestamptz`, stored in UTC, rendered in `studios.timezone`. Weekly schedules use `time` + `smallint` weekday (0 = Monday), interpreted in the studio timezone.
- Money is `bigint` minor units in `studios.currency`.
- Every tenant table has `studio_id NOT NULL`, including child tables, so every RLS policy is the same one line.
- Enumerations are `text` + `CHECK`, so adding a value is a migration, not a type rewrite.
- `created_at timestamptz NOT NULL DEFAULT now()` on every table (omitted below).

## Studios and people

```sql
CREATE TABLE studios (
  id        uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  slug      text NOT NULL UNIQUE CHECK (slug ~ '^[a-z0-9-]{3,40}$'),
  name      text NOT NULL,
  timezone  text NOT NULL,                     -- IANA, e.g. 'Europe/Berlin'
  currency  char(3) NOT NULL                   -- ISO 4217, fixed after creation
);

CREATE TABLE studio_settings (
  studio_id                       uuid PRIMARY KEY REFERENCES studios,
  min_lead_time_hours             int NOT NULL DEFAULT 72,
  max_lead_time_days              int NOT NULL DEFAULT 180,
  min_session_minutes             int NOT NULL DEFAULT 120,
  slot_step_minutes               int NOT NULL DEFAULT 30,
  email_verify_minutes            int NOT NULL DEFAULT 30,
  ttl_requested_hours             int NOT NULL DEFAULT 72,
  ttl_awaiting_client_info_hours  int NOT NULL DEFAULT 48,
  ttl_quoted_hours                int NOT NULL DEFAULT 48,
  ttl_awaiting_deposit_hours      int NOT NULL DEFAULT 24,
  hold_cutoff_hours               int NOT NULL DEFAULT 24,
  max_hold_days                   int NOT NULL DEFAULT 14,
  max_active_holds_per_client     int NOT NULL DEFAULT 3,
  deposit_percent                 int NOT NULL DEFAULT 30 CHECK (deposit_percent BETWEEN 0 AND 100),
  refund_cutoff_hours             int NOT NULL DEFAULT 48,
  no_show_window_days             int NOT NULL DEFAULT 7,
  CHECK (min_lead_time_hours > hold_cutoff_hours),
  CHECK (max_lead_time_days * 24 > min_lead_time_hours),
  CHECK (slot_step_minutes IN (15, 30, 60)),
  CHECK (min_session_minutes > 0 AND ttl_requested_hours > 0 AND ttl_awaiting_client_info_hours > 0
         AND ttl_quoted_hours > 0 AND ttl_awaiting_deposit_hours > 0 AND max_hold_days > 0)
);

CREATE TABLE studio_business_hours (
  studio_id  uuid NOT NULL REFERENCES studios,
  weekday    smallint NOT NULL CHECK (weekday BETWEEN 0 AND 6),
  opens_at   time NOT NULL,
  closes_at  time NOT NULL,
  CHECK (closes_at > opens_at)
);

CREATE TABLE users (                           -- global, not tenant-scoped
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  google_sub  text NOT NULL UNIQUE,
  email       text NOT NULL,
  name        text NOT NULL
);

CREATE TABLE memberships (
  studio_id       uuid NOT NULL REFERENCES studios,
  user_id         uuid NOT NULL REFERENCES users,
  role            text NOT NULL CHECK (role IN ('owner', 'admin', 'engineer')),
  takes_sessions  boolean NOT NULL DEFAULT true,
  can_claim_any   boolean NOT NULL DEFAULT false,
  is_active       boolean NOT NULL DEFAULT true,
  PRIMARY KEY (studio_id, user_id)
);

CREATE TABLE invitations (
  studio_id    uuid NOT NULL REFERENCES studios,
  email        text NOT NULL,
  role         text NOT NULL CHECK (role IN ('owner', 'admin', 'engineer')),
  token_hash   bytea NOT NULL UNIQUE,
  expires_at   timestamptz NOT NULL,
  accepted_at  timestamptz
);
```

## Catalog: session types, rooms, rates

```sql
CREATE TABLE session_types (
  id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  studio_id  uuid NOT NULL REFERENCES studios,
  name       text NOT NULL,
  is_active  boolean NOT NULL DEFAULT true
);
CREATE UNIQUE INDEX ON session_types (studio_id, lower(name));

CREATE TABLE rooms (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  studio_id    uuid NOT NULL REFERENCES studios,
  name         text NOT NULL,
  description  text,
  is_active    boolean NOT NULL DEFAULT true,
  UNIQUE (studio_id, name)
);

CREATE TABLE room_capabilities (               -- row present = allowed
  studio_id        uuid NOT NULL REFERENCES studios,
  room_id          uuid NOT NULL REFERENCES rooms,
  session_type_id  uuid NOT NULL REFERENCES session_types,
  is_preferred     boolean NOT NULL DEFAULT false,
  PRIMARY KEY (room_id, session_type_id)
);

CREATE TABLE room_rates (
  studio_id          uuid NOT NULL REFERENCES studios,
  room_id            uuid NOT NULL REFERENCES rooms,
  session_type_id    uuid REFERENCES session_types,   -- NULL = default for the room
  hourly_rate_minor  bigint NOT NULL CHECK (hourly_rate_minor >= 0),
  UNIQUE NULLS NOT DISTINCT (room_id, session_type_id)
);
```

New studios are seeded with session types Tracking, Overdubs, Mixing, Mastering, Audiobook, Podcast, which the owner can edit.

## Engineers

All engineer tables reference `(studio_id, user_id)` in `memberships`, so an engineer's specialties, rates and hours are per studio.

```sql
CREATE TABLE engineer_specialties (
  studio_id        uuid NOT NULL,
  engineer_id      uuid NOT NULL,
  session_type_id  uuid NOT NULL REFERENCES session_types,
  priority         int NOT NULL DEFAULT 0,          -- higher = more preferred
  PRIMARY KEY (studio_id, engineer_id, session_type_id),
  FOREIGN KEY (studio_id, engineer_id) REFERENCES memberships (studio_id, user_id)
);

CREATE TABLE engineer_rates (
  studio_id          uuid NOT NULL,
  engineer_id        uuid NOT NULL,
  session_type_id    uuid NOT NULL REFERENCES session_types,
  hourly_rate_minor  bigint NOT NULL CHECK (hourly_rate_minor >= 0),
  PRIMARY KEY (studio_id, engineer_id, session_type_id),
  FOREIGN KEY (studio_id, engineer_id) REFERENCES memberships (studio_id, user_id)
);

CREATE TABLE engineer_working_hours (
  studio_id    uuid NOT NULL,
  engineer_id  uuid NOT NULL,
  weekday      smallint NOT NULL CHECK (weekday BETWEEN 0 AND 6),
  starts_at    time NOT NULL,
  ends_at      time NOT NULL,
  CHECK (ends_at > starts_at),
  FOREIGN KEY (studio_id, engineer_id) REFERENCES memberships (studio_id, user_id)
);

CREATE TABLE engineer_ooo (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  studio_id    uuid NOT NULL,
  engineer_id  uuid NOT NULL,
  start_at     timestamptz NOT NULL,
  end_at       timestamptz NOT NULL,
  reason       text,
  CHECK (end_at > start_at),
  FOREIGN KEY (studio_id, engineer_id) REFERENCES memberships (studio_id, user_id)
);
```

## Clients

```sql
CREATE TABLE clients (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  studio_id       uuid NOT NULL REFERENCES studios,
  name            text NOT NULL,
  email           text NOT NULL,
  phone           text,
  notes           text,
  deposit_exempt  boolean NOT NULL DEFAULT false,
  anonymized_at   timestamptz
);
CREATE UNIQUE INDEX ON clients (studio_id, lower(email));
```

Anonymization overwrites name, email (with a unique placeholder), phone and notes, and sets `anonymized_at`. It is refused while the client has blocking bookings. Bookings and payments stay intact.

## Bookings

```sql
CREATE TABLE recurring_series (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  studio_id   uuid NOT NULL REFERENCES studios,
  client_id   uuid NOT NULL REFERENCES clients,
  frequency   text NOT NULL CHECK (frequency IN ('WEEKLY', 'BIWEEKLY')),
  until_date  date NOT NULL
);

CREATE TABLE bookings (
  id                     uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  studio_id              uuid NOT NULL REFERENCES studios,
  confirmation_code      text NOT NULL UNIQUE,
  client_id              uuid NOT NULL REFERENCES clients,
  session_type_id        uuid NOT NULL REFERENCES session_types,
  room_id                uuid NOT NULL REFERENCES rooms,
  engineer_id            uuid NOT NULL,      -- provisional until claimed_at is set
  requested_engineer_id  uuid,               -- set when the client named an engineer
  claimed_at             timestamptz,
  series_id              uuid REFERENCES recurring_series,
  rescheduled_from_id    uuid REFERENCES bookings,
  start_at               timestamptz NOT NULL,
  end_at                 timestamptz NOT NULL,

  status      text NOT NULL CHECK (status IN (
                'PENDING_EMAIL', 'REQUESTED', 'AWAITING_CLIENT_INFO', 'QUOTED',
                'AWAITING_DEPOSIT', 'BOOKED', 'COMPLETED', 'NO_SHOW',
                'EXPIRED', 'DECLINED', 'CANCELLED')),
  expires_at  timestamptz,

  quote_version          int NOT NULL DEFAULT 0,
  room_rate_minor        bigint,
  engineer_rate_minor    bigint,
  adjustment_minor       bigint,
  adjustment_reason      text,
  quoted_amount_minor    bigint,
  deposit_minor          bigint,             -- 0 when waived or exempt
  quote_sent_at          timestamptz,
  quote_accepted_at      timestamptz,

  final_amount_minor     bigint,
  payment_status         text NOT NULL DEFAULT 'none' CHECK (payment_status IN (
                           'none', 'deposit_paid', 'balance_due', 'paid', 'waived', 'refunded')),

  cancelled_at           timestamptz,
  cancelled_by           text CHECK (cancelled_by IN ('CLIENT', 'ENGINEER', 'ADMIN', 'SYSTEM')),
  reason_code            text,               -- see booking.md#reason-codes
  reason_text            text,

  client_notes           text,
  engineer_notes         text,
  conflict_flagged_at    timestamptz,
  reminded_at            timestamptz,
  escalated_at           timestamptz,
  google_event_id        text,

  CHECK (end_at > start_at),
  FOREIGN KEY (studio_id, engineer_id) REFERENCES memberships (studio_id, user_id),

  EXCLUDE USING gist (engineer_id WITH =, tstzrange(start_at, end_at) WITH &&)
    WHERE (status IN ('PENDING_EMAIL', 'REQUESTED', 'AWAITING_CLIENT_INFO',
                      'QUOTED', 'AWAITING_DEPOSIT', 'BOOKED')),
  EXCLUDE USING gist (room_id WITH =, tstzrange(start_at, end_at) WITH &&)
    WHERE (status IN ('PENDING_EMAIL', 'REQUESTED', 'AWAITING_CLIENT_INFO',
                      'QUOTED', 'AWAITING_DEPOSIT', 'BOOKED'))
);

CREATE INDEX ON bookings (expires_at)
  WHERE status IN ('PENDING_EMAIL', 'REQUESTED', 'AWAITING_CLIENT_INFO', 'QUOTED', 'AWAITING_DEPOSIT');
CREATE INDEX ON bookings (studio_id, start_at);
CREATE UNIQUE INDEX ON bookings (rescheduled_from_id)
  WHERE rescheduled_from_id IS NOT NULL
    AND status IN ('PENDING_EMAIL', 'REQUESTED', 'AWAITING_CLIENT_INFO', 'QUOTED', 'AWAITING_DEPOSIT');

CREATE TABLE booking_events (                  -- append-only history of every transition
  id             bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  studio_id      uuid NOT NULL REFERENCES studios,
  booking_id     uuid NOT NULL REFERENCES bookings,
  from_status    text,
  to_status      text NOT NULL,
  actor_kind     text NOT NULL CHECK (actor_kind IN ('CLIENT', 'ENGINEER', 'ADMIN', 'SYSTEM')),
  actor_user_id  uuid REFERENCES users,
  data           jsonb NOT NULL DEFAULT '{}',
  at             timestamptz NOT NULL DEFAULT now()
);
```

`tstzrange` is half-open (`[start, end)`), so back-to-back sessions do not conflict. The partial unique index on `rescheduled_from_id` enforces "one active reschedule per booking."

Confirmation codes: `BK-XXXX-XXXX` from a 32-character alphabet without `0 O 1 I`, drawn from a CSPRNG (40 bits). Uniqueness is enforced by the constraint; on collision, regenerate.

## Payments

```sql
CREATE TABLE stripe_accounts (
  studio_id          uuid PRIMARY KEY REFERENCES studios,
  stripe_account_id  text NOT NULL UNIQUE,
  charges_enabled    boolean NOT NULL DEFAULT false
);

CREATE TABLE payments (
  id                   uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  studio_id            uuid NOT NULL REFERENCES studios,
  booking_id           uuid REFERENCES bookings,
  series_id            uuid REFERENCES recurring_series,
  kind                 text NOT NULL CHECK (kind IN ('deposit', 'balance', 'refund')),
  provider             text NOT NULL CHECK (provider IN ('stripe', 'offline')),
  provider_payment_id  text,
  amount_minor         bigint NOT NULL,
  status               text NOT NULL CHECK (status IN ('pending', 'succeeded', 'failed')),
  CHECK (num_nonnulls(booking_id, series_id) = 1)
);

CREATE TABLE stripe_events (                   -- webhook idempotency
  stripe_event_id  text PRIMARY KEY,
  processed_at     timestamptz NOT NULL DEFAULT now()
);
```

## Google

```sql
CREATE TABLE google_grants (                   -- global, one per user
  user_id                  uuid PRIMARY KEY REFERENCES users,
  scopes                   text[] NOT NULL,
  refresh_token_encrypted  bytea NOT NULL,
  key_version              int NOT NULL,
  revoked_at               timestamptz
);

CREATE TABLE google_calendar_links (           -- per studio membership
  studio_id              uuid NOT NULL,
  engineer_id            uuid NOT NULL,
  sessionrelay_calendar_id  text NOT NULL,
  busy_calendar_ids      text[] NOT NULL,      -- never contains sessionrelay_calendar_id
  last_polled_at         timestamptz,
  PRIMARY KEY (studio_id, engineer_id),
  FOREIGN KEY (studio_id, engineer_id) REFERENCES memberships (studio_id, user_id),
  CHECK (NOT (sessionrelay_calendar_id = ANY (busy_calendar_ids)))
);

CREATE TABLE google_busy (
  studio_id    uuid NOT NULL,
  engineer_id  uuid NOT NULL,
  start_at     timestamptz NOT NULL,
  end_at       timestamptz NOT NULL,
  FOREIGN KEY (studio_id, engineer_id) REFERENCES memberships (studio_id, user_id)
);
CREATE INDEX ON google_busy USING gist (engineer_id, tstzrange(start_at, end_at));

CREATE TABLE studio_gmail_senders (
  studio_id     uuid PRIMARY KEY REFERENCES studios,
  user_id       uuid NOT NULL REFERENCES users,  -- whose google_grant sends
  sender_email  text NOT NULL,
  failing_since timestamptz
);
```

## Outbox

```sql
CREATE TABLE outbox (
  id               bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  studio_id        uuid NOT NULL REFERENCES studios,
  kind             text NOT NULL,            -- 'email', 'calendar_upsert', 'calendar_delete', 'stripe_refund', ...
  idempotency_key  text NOT NULL UNIQUE,
  payload          jsonb NOT NULL,
  attempts         int NOT NULL DEFAULT 0,
  next_attempt_at  timestamptz NOT NULL DEFAULT now(),
  done_at          timestamptz,
  last_error       text
);
CREATE INDEX ON outbox (next_attempt_at) WHERE done_at IS NULL;
```

## Row-level security

Every table with `studio_id` gets:

```sql
ALTER TABLE <t> ENABLE ROW LEVEL SECURITY;
ALTER TABLE <t> FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant ON <t>
  USING (studio_id = current_setting('app.studio_id')::uuid)
  WITH CHECK (studio_id = current_setting('app.studio_id')::uuid);
```

- The API connects as role `app` (no `BYPASSRLS`). Each request opens a transaction and runs `SET LOCAL app.studio_id = ...` from the authenticated membership, or from the slug on public routes. An unset value makes `current_setting` raise, so a forgotten tenant context fails loudly instead of leaking.
- Scheduled jobs that sweep all studios connect as role `app_jobs` with `BYPASSRLS`, and that role is used only by job handlers.
- Migrations run as the database owner.
- `users` and `google_grants` are global and accessed only through the authenticated user's own id.
