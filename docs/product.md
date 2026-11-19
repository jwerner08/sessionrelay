# Product

## What it is

SessionRelay lets a recording studio take booking requests from clients, route them to the right engineer and room, quote them, collect a deposit, and confirm them, without double-booking anyone. Engineers keep living in Google Calendar; SessionRelay reads their busy time and writes their sessions back.

## Who uses it

| Actor | Logs in? | Does what |
|---|---|---|
| **Client** (band, artist, narrator, label) | No. Acts through signed email links | Requests sessions, answers questions, accepts quotes, pays deposits and balances, cancels, reschedules |
| **Engineer** | Yes, Google sign-in | Claims and quotes requests, manages working hours and OOO, connects Google Calendar, closes out sessions |
| **Admin** | Yes | Everything an engineer does, plus engineers, rooms, rates, permissions, reassignment, acting on any booking |
| **Owner** | Yes | Everything an admin does, plus studio settings, Stripe connection, Gmail connection |

One Google account can be a member of several studios (freelance engineers). Owners and admins can also take sessions themselves (`memberships.takes_sessions`).

## Tenancy

Multi-tenant from day one. Every tenant table carries `studio_id` and is protected by Postgres row-level security (see [data-model.md](data-model.md)). Each studio has its own timezone, currency, settings, session types, rooms, rates, clients and booking page.

- Public booking page: `https://sessionrelay.com/s/{studio-slug}`
- Custom studio domains: later, via Cloudflare for SaaS, with no app changes.

## Onboarding

Invite-only for the pilot. SessionRelay staff create the studio and invite its owner; the owner invites engineers by email. The code path is the same one self-serve signup will use, gated by a flag. Invite-only also keeps the user count under Google's 100-user cap for unverified apps while verification is pending (see [google.md](google.md)).

## Business model

Free during the pilot. A per-studio monthly subscription later. No percentage of deposits, because that penalizes studios for using deposits at all. No billing code in V1.

## Not in V1

- Client accounts (clients never log in)
- Full two-way Calendar sync (Google edits never change bookings)
- Generating invoices ourselves (Stripe Invoicing on the studio's account does it)
- Payment providers other than Stripe
- SMS, mobile apps, waitlists, analytics dashboards
- Buffer/setup time between sessions, equipment rental
- Custom studio domains
- Magic-link login (Google sign-in only)

## Roadmap

Ordered by dependency, not by calendar. Each phase is deployable and usable without the next.

1. **Foundation**: multi-tenant schema with RLS, Google sign-in, memberships and roles, studio settings, rooms and session types as data, CI/CD to staging and prod.
2. **Core booking loop**: availability (working hours, OOO, holds), public booking page, email verification for holds, request → claim → quote → accept → BOOKED, TTLs and expiry jobs, platform email sender.
3. **Google**: Calendar connection, free/busy polling plus live check at submit, the SessionRelay calendar, conflict flags, sending through the studio's Gmail with platform fallback. Submit for Google verification at the start of this phase; review takes weeks.
4. **Money**: Stripe Connect onboarding, deposits, `AWAITING_DEPOSIT`, refunds, deposit-exempt clients, close-out with Stripe invoices, `NO_SHOW`.
5. **Operations**: admin reassignment, `can_claim_any`, recurring series, reschedule, escalations, client anonymization.
6. **Pilot**: onboard the first studios by invitation.
