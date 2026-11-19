# Notifications

All email is enqueued in the [outbox](architecture.md#outbox) by the transaction that caused it, and sent after commit.

- **Studio sender**: the studio's connected Gmail, falling back to the platform sender on failure. See [google.md](google.md#gmail).
- **Platform sender**: `notifications@sessionrelay.com` via the transactional provider.

All times render in the studio's timezone with the zone shown. Every action link is a signed token (see [architecture.md](architecture.md#signed-links)); emails never contain internal ids or bare confirmation-code actions.

## Client emails (studio sender)

| Trigger | Content | Links (token action, lifetime) |
|---|---|---|
| New client submits | Confirm your email to hold the slot | `confirm-email`, 30 min |
| Request is `REQUESTED` | Request received; quote expected within the TTL; confirmation code | `view-status`, 30 days; `cancel`, until session |
| Claimed or reassigned | Now handled by [engineer] | `view-status` |
| Engineer asks for info | Engineer's questions | `answer-info`, until `expires_at` |
| Quote sent | Session details, price breakdown, deposit, expiry | `accept-quote` and `decline-quote`, until `expires_at`, bound to `quote_version` |
| Deposit due | Pay deposit to confirm | `pay-deposit`, until `expires_at` |
| `BOOKED` | Confirmed; total; paid so far; studio address; series dates if recurring | `view-status`, `reschedule`, `cancel`, until session |
| Declined | Reason; up to 3 alternative times | `request-slot` per alternative, 7 days |
| Expired | Request or quote expired; slot released | booking page link |
| Cancelled | Details, who cancelled, refund (if any) | booking page link |
| Reschedule requested | Original and new time; original stays until new one is booked | `view-status` for the new booking |
| Balance invoice | Sent by Stripe from the studio's Stripe account, not by us | Stripe hosted invoice |

## Engineer and admin emails (platform sender)

| Trigger | To |
|---|---|
| New request (named) | Named engineer |
| New request (any engineer) | Every eligible engineer free at that time |
| Client answered questions | Assigned engineer |
| Quote accepted / deposit paid / client declined / client cancelled | Assigned engineer |
| Reminder at half TTL | Assigned engineer, or queue |
| Escalation 24h before expiry | Owner and admins |
| Google conflict flagged | Engineer (once per booking) |
| Calendar disconnected | Engineer, admins |
| Gmail fallback started | Owner |
| Invite to studio | Invitee |
