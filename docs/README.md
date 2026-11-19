# SessionRelay docs

SessionRelay is multi-tenant scheduling and booking software for recording studios. It coordinates rooms, engineers, holds, quotes, deposits and client bookings in one workflow, and it treats each engineer's Google Calendar as a source of truth for when they are busy.

## Reading order

| Doc | What it covers |
|---|---|
| [product.md](product.md) | Who it is for, tenancy, roles, scope, roadmap |
| [booking.md](booking.md) | Booking lifecycle, holds, availability, quoting, claiming, recurring, reschedule, cancellation |
| [google.md](google.md) | Google sign-in, Calendar read/write, Gmail sending, OAuth scopes and verification |
| [payments.md](payments.md) | Stripe Connect, deposits, refunds, close-out, invoices, currency |
| [notifications.md](notifications.md) | Every email the system sends, who sends it, and which signed links it carries |
| [data-model.md](data-model.md) | Postgres schema, constraints, row-level security |
| [architecture.md](architecture.md) | Stack, infrastructure, jobs, environments, CI/CD, security, testing, monitoring |
| [decisions.md](decisions.md) | Decision log: what was decided, why, and what was rejected |
| [research/studio-interview-guide.md](research/studio-interview-guide.md) | Questions for interviewing a studio owner before building for them |

## Open items

These are known gaps that need an answer before or during the phase that depends on them.

| Item | Needed by |
|---|---|
| Postmark or Resend for the platform sender | Phase 2 |
| Confirm the sensitivity class of `calendar.freebusy` and `calendar.app.created` in the Google Cloud consent screen config (Google's docs do not state it) | Phase 3 |
| Cloud Run custom domain wiring for `api.sessionrelay.com` (Cloud Run domain mapping vs. a load balancer) | Phase 1 |
