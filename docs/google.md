# Google integration

Google is the first integration and is part of V1. It covers sign-in, Calendar, and Gmail.

## Sign-in

- Engineers, admins and owners sign in with Google (OpenID Connect). Any Google account works; Workspace is not required.
- Sign-in requests only `openid email profile`. Calendar and Gmail scopes are requested later, incrementally, when the user connects them.
- A user is matched by Google `sub`, not by email (emails can change).
- Clients never sign in.

## Calendar

Scope of V1: **SessionRelay writes bookings to Google, and reads busy time from Google. Google edits never change bookings.**

### Scopes

| Scope | Used for |
|---|---|
| `calendar.freebusy` | Reading busy intervals from the engineer's chosen calendars |
| `calendar.app.created` | Creating and managing the dedicated SessionRelay calendar and its events |

These are the narrowest scopes that do the job. No access to event titles or details on the engineer's own calendars.

### Reading busy time

- When connecting, the engineer picks which of their calendars count as busy (default: primary). The SessionRelay calendar is **never** in this list; otherwise SessionRelay's own events would read back as personal busy time and flag conflicts against themselves.
- A Cloud Scheduler job every 5 minutes calls the free/busy API for each connected engineer over the booking window and replaces their rows in `google_busy` (start, end, fetched_at). Only intervals are stored, never titles.
- On request submit and on claim, the API makes a **live** free/busy call for the engineer and interval in question, so anything added since the last poll is caught.
- If Google is unreachable or the token is revoked: availability falls back to the last poll, the engineer and admins see "Calendar disconnected," and the engineer gets one email.

No webhooks: change notifications would need broader read scopes, a public webhook, and channel renewal. Polling plus the live check gives the same correctness with less surface.

### Writing bookings

- On first connect, SessionRelay creates a secondary calendar named "SessionRelay - {studio}" in the engineer's account.
- Holds (`REQUESTED` through `AWAITING_DEPOSIT`) are written as **tentative** events; `BOOKED` as confirmed. Terminal states delete the event.
- Event description: client name, session type, room, and a dashboard link. No prices.
- Writes go through the [outbox](architecture.md#outbox), keyed by booking id, so retries are safe and the event id is stored on the booking.

### Conflicts

New Google busy time over an existing booking flags it; it never cancels or declines anything. See [booking.md](booking.md#conflicts-from-google-calendar).

## Gmail

Client-facing booking mail goes out from the studio's own address, so client replies reach the studio.

- The owner connects one Google account as the studio sender. Scope: `gmail.send` only (classified **sensitive** by Google).
- Messages are sent through the Gmail API as that account, with `Reply-To` set to the assigned engineer where relevant.
- Gmail limits: 500 messages/day for consumer Gmail and Workspace trial accounts, 2,000/day for paid Workspace users, measured over a rolling 24 hours.

### Platform sender and fallback

System mail always goes from `notifications@sessionrelay.com` through a transactional provider (Postmark or Resend, see [README.md](README.md#open-items)):

- invites, sign-in notices, "connect your Gmail" prompts, job alerts
- **every** client-facing message when the studio has no Gmail connected, the token is revoked, or Gmail returns an error or rate limit

The outbox worker tries Gmail first and falls back on failure, so a revoked token never silently stops quotes and expiry notices. The studio's owner is told once when fallback starts.

## Tokens

- One Google grant per user; scopes are added incrementally.
- Refresh tokens are encrypted in the application with AES-GCM using a key from Google Secret Manager before they reach the database. See [architecture.md](architecture.md#secrets).
- Disconnecting, or leaving a studio, revokes the grant at Google and deletes the ciphertext.

## Verification

Calendar and Gmail scopes require Google's OAuth app verification for an app used by people outside our organization. Until verified, the app is limited to 100 new users. `gmail.send` is a sensitive scope (not restricted), so no third-party security assessment is required.

Start verification at the beginning of phase 3. Requirements to have ready: privacy policy and terms on sessionrelay.com, a verified domain, a demo video of each scope in use, and a scope justification that matches what is listed here.

Separate OAuth clients for staging and prod.
