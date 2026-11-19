# Payments

## Provider

**Stripe Connect, Standard accounts.** Each studio connects its own Stripe account through Stripe's hosted onboarding. Money goes to the studio; refunds, disputes and payouts are on the studio's account. SessionRelay takes no fee.

Payment state lives in our own `payments` table (`provider`, `provider_payment_id`, `kind`, `amount_minor`, `status`), not in Stripe objects, so a second provider (Square is the likely one) can be added if studios ask. There is no provider abstraction until that second provider exists.

Studios without Stripe connected run without deposits or online balance collection.

## Currency and amounts

- Each studio has an ISO 4217 `currency`, fixed at creation.
- All money is stored as integer minor units (`BIGINT`, 35000 = $350.00). No floats or decimals anywhere in the code path. Amounts go to Stripe unchanged.
- Rounding happens once, when the deposit is computed, with round-half-up to the minor unit.

## Deposits

Who pays a deposit:

1. Studio default: `deposit_percent` (default 30; 0 disables deposits).
2. Client override: `clients.deposit_exempt` set by an admin for trusted clients.
3. Quote override: the engineer can waive the deposit on a single quote.

Flow:

1. Client accepts a quote that requires a deposit → `AWAITING_DEPOSIT` with its own TTL (clamped like the others).
2. Client is sent to Stripe Checkout on the studio's connected account.
3. The `checkout.session.completed` webhook moves the booking to `BOOKED` and sets `payment_status = deposit_paid`.
4. Unpaid at expiry → `EXPIRED`, slot released.

Webhooks are verified with the Stripe signing secret and processed idempotently by Stripe event id.

## Refunds

| Situation | Refund |
|---|---|
| Client cancels more than `refund_cutoff_hours` (48) before the session | Full deposit |
| Client cancels within the cutoff | None |
| Studio, engineer or admin cancels | Full deposit |
| Reschedule | None; deposit moves to the new booking |
| `NO_SHOW` | None |

Refunds are enqueued by the state transition and executed by the outbox worker with a Stripe idempotency key derived from the booking id and transition, so a retry never refunds twice.

## Close-out

After a session is `COMPLETED`, the engineer records the final amount: actual hours plus line items (overtime, extras), starting from the quote. SessionRelay shows `balance = final_amount - deposit_paid` and offers:

- **Send invoice**: creates a Stripe Invoice on the studio's connected account. Stripe handles invoice numbering, the PDF, the payment page, reminders and tax (if the studio enables Stripe Tax). `payment_status = balance_due` until Stripe reports it paid.
- **Mark paid offline**: cash, bank transfer, or the studio's own accounting tool.
- **Waive**: writes off the balance.

SessionRelay does not generate invoices itself. Invoices are legal documents with per-country rules (sequential numbering, tax IDs, VAT/GST, retention), and studios already have accounting tools.

## `payment_status`

`none` → `deposit_paid` → `balance_due` → `paid`, plus `waived` and `refunded`. Independent of booking status.

## Pricing visibility

Clients see no prices before a quote: public APIs and the booking page return none. The quote email shows the breakdown (room, engineer, adjustment, total, deposit). After booking, the status page shows the accepted total and what has been paid. Engineers and admins always see full pricing.
