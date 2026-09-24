# bookable-events

An [Agent Skill](https://agentskills.io) that teaches an agent to build **bookable events with Stripe
payments and anti-no-show deposits** in a **Next.js App Router** app: capacity-limited registration for
free and paid events, Stripe Checkout tickets, a refundable card hold on free events that is released when
the guest checks in and captured when they don't show, staff check-in, refunds and event cancellation.
Firestore or Postgres; any ISO currency, USD by default.

Events are the easy half. The hard half is money moving while seats are scarce: a seat taken before
payment, a payment that lands after the seat expired, a card hold that lapses before a no-show can be
charged. The skill runs **every change through one pure state machine inside a database transaction, and
calls Stripe only after commit**. A durable intent (`RELEASING`, `CAPTURING`, `REFUND_PENDING`) is left
behind, and a single five-minute tick retries it with the same idempotency key until Stripe confirms.

It is extracted from the events module of a production multi-location venue-booking system, and ships as
the **hardened** version: a three-pass audit found fifteen defect groups, several of which leaked money —
a paid event that was free in an unpriced currency, a declined card attempt that deleted the seat while
Checkout stayed open. [`references/provenance.md`](references/provenance.md) names every one and the fix.

## Install

```bash
npx skills add timerise-ai/bookable-events
```

Name agents with `-a`, for example `npx skills add timerise-ai/bookable-events -a claude-code -a codex`.
Or clone it into an agent's skills directory — for Claude Code:

```bash
git clone https://github.com/timerise-ai/bookable-events.git ~/.claude/skills/bookable-events
```

To scope it to one project, clone into that project's `.claude/skills/`. The current release is **0.1.0** —
see [`CHANGELOG.md`](CHANGELOG.md).

## Activation

The skill activates when a task matches its description — capacity-limited events, free events with
no-shows, a card hold released on arrival, event refunds and cancellation — and on phrases like "no-show
deposit", "card hold", "capture_method manual", "release the deposit on check-in", "charge no-shows",
"Stripe Checkout tickets", "cancel event and refund everyone". Invoke it explicitly with `/bookable-events`
in Claude Code, `$bookable-events` in Codex CLI, or from `/skills` in Gemini CLI.

## What's inside

| File | Contents |
|---|---|
| `SKILL.md` | Entry point: when (not) to use, architecture, six critical facts, five hard rules, quick start, Adaptation Contract, reference directory |
| `references/data-model.md` | Rename table, the neutral types, why the shape is what it is, seat accounting |
| `references/money-and-time.md` | Any-currency conversion (0/2/3 decimals, ISK/UGX), pricing that is never free by accident, DST-correct time |
| `references/participant-lifecycle.md` | The pure state machine, effects diff, and the one due-work pointer |
| `references/no-show-deposits.md` | Hybrid hold (Checkout hold or saved card + off-session hold), timeline and policy, outcomes, disclosure, the tick |
| `references/stripe.md` | Checkout Sessions, webhook handling and idempotency, the gateway, an 11-step test-mode walkthrough |
| `references/registration.md` | Registration, cancellation and refund flows, guest manage links, request parsing |
| `references/engine.md` | `createEventsEngine`, its dependencies and the host-facing ports |
| `references/firestore.md` | Firestore store, indexes, security rules |
| `references/postgres.md` | Postgres schema, store, RLS, a `pg` client |
| `references/api-routes.md` | Route surface, error codes, authorization rules, the host seam file, every route handler |
| `references/ui.md` | Event card, registration dialog, manage page, staff check-in screen, admin form |
| `references/testing.md` | Vitest config, in-memory store and fake gateway, core and state-machine suites |
| `references/testing-lifecycles.md` | 17 end-to-end lifecycle tests, including three regression tests |
| `references/operations.md` | Environment, the tick, what operators need to see, reconciliation, emails, go-live |
| `references/provenance.md` | Defects fixed, choices kept deliberately, additions, fix order for the original |

## The non-negotiables

1. **Transaction first, Stripe second.** No Stripe call inside a transaction; no state written after a
   Stripe call without a durable intent first.
2. **Identity never comes from the request body.** Session for customers, HMAC link for guests, a location-
   scoped guard on every staff and admin `[eventId]` route.
3. **Never capture before settlement.** A hold that would lapse is renewed or released, never captured early.
4. **Cancelling an event is an action, not a status.** It refunds and releases everyone.
5. **A webhook error is a 500.** Release the claim and let Stripe retry.

## Not this

| Not this | Use instead |
|---|---|
| Marketplace splits, payouts, connected accounts, subscriptions | The sibling [`stripe-connect-subscriptions`](https://github.com/timerise-ai/stripe-connect-subscriptions) skill |
| Walk-up touchscreen registration | The sibling [`booking-kiosk`](https://github.com/timerise-ai/booking-kiosk) skill |
| Hourly resource booking (lanes, courts, rooms) | The host's booking engine; events only block its slots |
| Seat maps, ticket resale | A ticketing platform |

## Contributing

Pure markdown. The code blocks are compiled and tested before release: `references/testing.md` and
`references/testing-lifecycles.md` carry the suites that run against them. Adding or renaming a reference
means updating the quick start and reference directory in `SKILL.md`, the table above, and relative
cross-links. See `CLAUDE.md` for the editing conventions.

## Part of the Timerise skills

One of the [Timerise skills](https://github.com/timerise-ai/skills) — production-extracted modules for
**Next.js App Router** apps, sharing one layout: a `SKILL.md` entry point, `references/` loaded on demand,
and a seam contract carrying the module's non-negotiables.

## Author

Built and maintained by [Timerise](https://timerise.ai).

## License

MIT — see [LICENSE](LICENSE).
