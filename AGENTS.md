# SessionRelay

Multi-tenant booking software for recording studios. Design lives in `docs/`; work is tracked in GitHub.

## Docs are the source of truth

Read `docs/README.md` first; it maps every doc and lists open items. Before designing or building anything, read the doc for that area:

- Booking states, holds, availability, claiming → `docs/booking.md`
- Schema, constraints, RLS → `docs/data-model.md`
- Google sign-in, Calendar, Gmail → `docs/google.md`
- Stripe, deposits, refunds → `docs/payments.md`
- Emails and signed links → `docs/notifications.md`
- Stack, infra, jobs, CI, testing → `docs/architecture.md`
- Why something is the way it is → `docs/decisions.md`

When a change alters a design, update the doc and `docs/decisions.md` in the same PR. Decisions are numbered (D1, D2, ...); append new ones, and mark reversed ones superseded rather than deleting them.

## Work tracking

- Project board: https://github.com/users/jwerner08/projects/1
- One milestone per roadmap phase; the milestone description holds the phase goal as a demo.
- Each phase after the current one is a single issue labeled `epic`. Break an epic into work items only when the previous phase is nearly done.
- Every work item has four sections: **Why**, **What** (checklist), **Definition of Done** (observable checks, including one that fails if the work is wrong), **Related resources** (doc anchors, decision numbers, `Depends on: #n`). The issue form in `.github/ISSUE_TEMPLATE/work-item.yml` enforces this.
- Lines starting with "Decide:" in an issue are open design questions. Resolve them with the user while doing that issue, then record the outcome in `docs/`.
- PRs close their issue with `Closes #n`.
