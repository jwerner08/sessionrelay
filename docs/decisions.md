# Decision log

Each entry: what was decided, why, and what was rejected. Details live in the linked doc. Append new decisions; when one is reversed, mark it superseded instead of deleting it.

## Product

**D1. Multi-tenant from day one.** Every tenant table has `studio_id`, protected by RLS. Retrofitting tenancy later means migrating every table and query. Rejected: single studio first. → [product.md](product.md#tenancy)

**D2. Invite-only pilot, self-serve behind a flag.** Avoids spam signups and keeps under Google's 100-user cap while verification is pending. → [product.md](product.md#onboarding)

**D3. Free during the pilot; subscription later; no cut of deposits.** A percentage fee penalizes studios for using deposits. No billing code in V1. → [product.md](product.md#business-model)

**D4. Public booking pages at `sessionrelay.com/s/{slug}`.** Simplest; custom domains can be added later through Cloudflare for SaaS. → [architecture.md](architecture.md#domains)

## Booking

**D5. Double booking prevented by Postgres exclusion constraints.** The plan's `SELECT ... FOR UPDATE` cannot lock rows that do not exist yet, so two requests for an empty slot could both succeed. → [data-model.md](data-model.md#bookings)

**D6. All times are `timestamptz`; each studio has an IANA timezone.** Google returns zoned times, and naive timestamps break across DST. → [data-model.md](data-model.md)

**D7. TTLs are clamped to the session start and a maximum hold age.** Without it, a pending hold can outlive the session it is holding. Clamp settings are per studio and validated by CHECK constraints. → [booking.md](booking.md#expiry)

**D8. Settings are typed columns per studio, not key/value strings.** Invalid values are rejected by the database, not discovered at runtime. → [data-model.md](data-model.md#studios-and-people)

**D9. "Any engineer" is the default; requests go to a shared queue; first claim wins.** Removes most of the "one engineer ignored it" escalation. "Any" requests get a provisional engineer so the exclusion constraint protects capacity. → [booking.md](booking.md#named-engineer-vs-any-engineer)

**D10. `can_claim_any` lets an engineer claim "any" requests outside their specialties, but not requests addressed to a named engineer.** Only admins move named requests, and the client is told. → [booking.md](booking.md#claiming)

**D11. Admins can reassign; reassigning a quote voids it.** `quote_version` increments and is bound into accept links, so a link from a voided quote can never accept a newer one. → [booking.md](booking.md#reassignment-admins)

**D12. Recurring requests are a series quoted and accepted once.** Twenty-six separate quotes and emails is unusable. Occurrences stay individually cancellable. → [booking.md](booking.md#recurring-series)

**D13. New clients confirm their email before a hold notifies anyone.** Guest booking lets anyone block a calendar with someone else's email. Plus Turnstile and a per-client active-hold limit. Known clients skip it. → [booking.md](booking.md#requesting-a-session)

**D14. One client record per studio per email (case-insensitive).** The plan's "update if email exists" had no uniqueness to rely on. → [data-model.md](data-model.md#clients)

**D15. OOO auto-declines holds; Google busy time only flags.** OOO is a deliberate statement in SessionRelay; personal calendars are noisy. → [booking.md](booking.md#conflicts-from-google-calendar)

**D16. Booking status and payment status are separate fields.** "Happened" and "paid" are independent facts. `FINAL_INVOICED` removed. → [booking.md](booking.md#lifecycle)

**D17. `NO_SHOW` state, settable for 7 days after a session.** Deposit is kept; the count informs deposit exemptions. → [booking.md](booking.md#no-show-and-completion)

**D18. Signed tokens for every client state change; the confirmation code is read-only.** The plan allowed accept and cancel with the bare code, which appears in every email. → [architecture.md](architecture.md#signed-links)

## Google

**D19. Google Calendar and Gmail are V1, not future work.** Engineers' real availability is in Google; without it the app holds slots they cannot work. → [google.md](google.md)

**D20. Calendar: push bookings, read free/busy; Google edits never change bookings.** Rejected: full two-way sync, where a calendar edit could move a session a client paid for. → [google.md](google.md#calendar)

**D21. Poll free/busy every 5 minutes plus a live check at submit; no webhooks.** Narrower scopes (`calendar.freebusy`, `calendar.app.created`), easier verification, nothing to renew. → [google.md](google.md#reading-busy-time)

**D22. Bookings go into a dedicated SessionRelay calendar, excluded from the busy read.** Clean removal, can be hidden, and prevents the app reading its own events back as conflicts. → [google.md](google.md#writing-bookings)

**D23. Google sign-in only.** One login path, and OAuth is needed for Calendar anyway. Rejected for now: magic links, passwords. → [google.md](google.md#sign-in)

**D24. Two senders: the studio's Gmail for client mail, a platform provider for system mail and as fallback.** A revoked Gmail token must never silently stop quotes and expiry notices. → [google.md](google.md#platform-sender-and-fallback)

**D25. Google refresh tokens are encrypted in the application with a Secret Manager key.** A database leak alone exposes no calendars or mailboxes. → [architecture.md](architecture.md#secrets)

## Money

**D26. Stripe Connect Standard; payment state kept in our own tables.** Best API and hosted onboarding; studios own their money and disputes. Square is the likely second provider if studios ask. Rejected: PayPal (onboarding and disputes), Adyen (built for far larger volume). → [payments.md](payments.md#provider)

**D27. Deposits by default, exemptions per client and per quote, paid before `BOOKED`.** `AWAITING_DEPOSIT` stops "accepted but unpaid" from holding a confirmed slot. → [payments.md](payments.md#deposits)

**D28. Refunds by rule: full before the cutoff or when the studio cancels; none after cutoff or on no-show; deposits carry over on reschedule.** → [payments.md](payments.md#refunds)

**D29. Close-out and balance collection in V1; invoices generated by Stripe, not by us.** Invoices carry per-country legal requirements; studios already have accounting tools. → [payments.md](payments.md#close-out)

**D30. Money in integer minor units; currency per studio.** No rounding drift; matches Stripe. → [payments.md](payments.md#currency-and-amounts)

## Platform

**D31. Cloudflare + Cloud Run + Neon, nothing else until there is a demonstrated need.** No Kubernetes, Redis, queue or GraphQL. → [architecture.md](architecture.md)

**D32. Go + sqlc + pgx; React with OpenAPI-generated types.** The complexity is in SQL and state machines; keep both explicit. Rejected: TypeScript backend, ORMs. → [architecture.md](architecture.md#stack)

**D33. Jobs are Cloud Scheduler calling idempotent HTTP endpoints; side effects go through a transactional outbox.** Cloud Run scales to zero, and a state change must never commit without its notification. Rejected: pg-boss, a queue service. → [architecture.md](architecture.md#outbox)

**D34. Staging and prod, with a Neon branch for staging; same image promoted.** → [architecture.md](architecture.md#environments)

**D35. Integration tests against real Postgres for the state machine, constraints and RLS.** Those are where the risk is, and they only behave correctly against Postgres. → [architecture.md](architecture.md#testing)

**D36. Client data deletion is anonymization, refused while bookings are active.** Financial records must survive; personal data must not. → [data-model.md](data-model.md#clients)

**D37. Owners and admins can also take sessions (`takes_sessions`).** Small studios' owners are often engineers. → [data-model.md](data-model.md#studios-and-people)
