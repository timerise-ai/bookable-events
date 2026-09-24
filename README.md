# bookable-events

[![Agent Skills](https://img.shields.io/badge/Agent_Skills-open_format-059669)](https://agentskills.io)
[![skills.sh](https://img.shields.io/badge/skills.sh-npx_skills_add-059669)](https://www.skills.sh)
[![Claude Code](https://img.shields.io/badge/Claude_Code-compatible-059669)](https://docs.claude.com/en/docs/claude-code/skills)
[![Codex CLI](https://img.shields.io/badge/Codex_CLI-compatible-059669)](https://developers.openai.com/codex/skills)
[![Gemini CLI](https://img.shields.io/badge/Gemini_CLI-compatible-059669)](https://github.com/google-gemini/gemini-cli/blob/main/docs/cli/skills.md)

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

This skill was written by the engineer who has shipped this module. The earlier implementation it was
audited against was the events module of a multi-location venue-booking system. The templates hold the
properties such a module has to hold: a paid event is never free in a currency it has no price in; a
declined attempt inside open Checkout keeps the seat, and only session expiry releases it; money that
arrives for a released seat is refunded; a refund is the amount frozen at registration; a no-show hold is
placed inside the card network's window and captured only after settlement; a redelivered webhook or a
repeated action changes nothing. The three suites, 47 tests, state each one, and
[`references/provenance.md`](references/provenance.md) has the record.

## Install

One command, via the [skills.sh](https://www.skills.sh) CLI, which installs the skill into every
skills-compatible agent it detects, including Claude Code, Codex CLI and Gemini CLI:

```bash
npx skills add timerise-ai/bookable-events
```

Name the agents instead with `-a`, for example
`npx skills add timerise-ai/bookable-events -a claude-code -a codex`.

### Manual install

Nothing here is Claude-specific: the skill is a plain [Agent Skills](https://agentskills.io) folder,
`SKILL.md` plus markdown references with no file that calls a model, so cloning it into an agent's skills
directory is all an install is. For Claude Code:

```bash
git clone https://github.com/timerise-ai/bookable-events.git ~/.claude/skills/bookable-events
```

To scope it to a single project instead, clone it into that project's `.claude/skills/` directory. For another
agent, clone into that agent's skills directory, or symlink the Claude Code copy so one `git pull` updates
every agent:

```bash
mkdir -p ~/.agents/skills
ln -s ~/.claude/skills/bookable-events ~/.agents/skills/bookable-events
```

Update the skill with `git pull` in its directory. The current release is **0.1.0**. See
[CHANGELOG.md](CHANGELOG.md). The [skills index](https://github.com/timerise-ai/skills) lists the other
Timerise Skills and how to install them all at once.

## Activation

The skill activates automatically when a task matches its description: capacity-limited classes, workshops,
tournaments or open days; free events with no-shows; a card hold released on arrival and captured on a
no-show; event refunds and cancellation; or auditing an existing event-registration module. Phrases such as
"no-show deposit", "card hold", "capture_method manual", "release the deposit on check-in", "charge
no-shows", "Stripe Checkout tickets" and "cancel event and refund everyone" match it. Invoke it explicitly
with `/bookable-events` in Claude Code, `$bookable-events` in Codex CLI, or from `/skills` in Gemini CLI.

Each host matches a task against the description its own way, so invoke the skill explicitly on a first run
rather than assuming it fired. Only `SKILL.md` is read up front; the `references/` files load on demand, so
the skill stays cheap in context until a topic is actually needed.

## What's inside

| File | Contents |
|---|---|
| `SKILL.md` | Entry point: when (not) to use, architecture and the adaptation contract, six critical facts, five hard rules, quick start, reference directory |
| `README.md` | This front door |
| `CHANGELOG.md` | Keep a Changelog, one section per release, newest first |
| `CLAUDE.md` | What this repository is and the conventions for editing the skill itself |
| `LICENSE` | MIT |
| `references/data-model.md` | Rename table, the neutral types, why the shape is what it is, seat accounting |
| `references/money-and-time.md` | Any-currency conversion (0, 2 and 3 decimals, ISK and UGX), pricing that is never free by accident, DST-correct time |
| `references/participant-lifecycle.md` | The pure state machine, the effects diff, and the one due-work pointer |
| `references/no-show-deposits.md` | Hybrid hold (Checkout hold, or saved card and off-session hold), timeline and policy, outcomes, disclosure, the tick |
| `references/stripe.md` | Checkout Sessions, webhook handling and idempotency, the gateway, an 11-step test-mode walkthrough |
| `references/registration.md` | Registration, cancellation and refund flows, guest manage links, request parsing |
| `references/engine.md` | `createEventsEngine`, its dependencies and the host-facing ports |
| `references/firestore.md` | Firestore store, indexes, security rules |
| `references/postgres.md` | Postgres schema, store, RLS, a `pg` client |
| `references/api-routes.md` | Route surface, error codes, authorization rules, the host seam file, every route handler |
| `references/ui.md` | Event card, registration dialog, manage page, staff check-in screen, admin form |
| `references/testing.md` | Vitest config, in-memory store and fake gateway, the core and state-machine suites |
| `references/testing-lifecycles.md` | 17 end-to-end lifecycle tests, including three regression tests |
| `references/operations.md` | Environment, the tick, what operators need to see, reconciliation, emails, go-live |
| `references/provenance.md` | The engineering ledger: what the audit changed and how the templates verify it, what was kept on purpose, what is new in the skill, and the order of work on an existing module |

The seam is the adaptation contract table in `SKILL.md`, which this skill keeps there instead of in a
`references/adaptation.md`, with the rename table in `references/data-model.md`. It bounds the store, behind
the `EventStore` port with Firestore and Postgres implementations; identity and location scope, behind the
three functions of `lib/events/host.ts`; stored credit and notifications, behind the optional `BalanceLedger`
and the `Notifier` ports; and the host's strings, styling, UI primitives and scheduler. Stripe itself sits
behind `PaymentGateway`, so the tests run against a fake.

## The five non-negotiables

1. **Transaction first, Stripe second.** No Stripe call inside a transaction, and no state written after a
   Stripe call without a durable intent first, because transaction callbacks re-run and a failed write after
   a capture loses the record of money moved. The engine is the only writer of `payment`, `deposit` and
   `attendance`, and the lifecycle suite drives every flow through it.
2. **Identity never comes from the request body.** Customers come from the session, guests from an HMAC
   manage link, staff from a location-scoped guard on every staff and admin `[eventId]` route, because a body
   field lets anyone attach a registration to someone else's account and collect its refund. A lifecycle test
   holds that a manage token verifies only for its own participant.
3. **Never capture before settlement.** A hold that would lapse is renewed or released, never captured early,
   because check-in must stay possible until the event is over. The lifecycle suite holds that a no-show is
   captured after the grace period and not before.
4. **Cancelling an event is an action, not a status.** It closes registration, then refunds and releases every
   seat; a status flip would leave every participant charged. A lifecycle test cancels an event and checks
   every hold is released.
5. **A webhook error is a 500.** The handler releases its event-id claim and answers 500 so Stripe retries,
   because a swallowed error is money nobody recorded. A redelivered webhook is a no-op, which a lifecycle
   test holds.

Everything else is the host app's: authentication, tenancy, strings, locale, styling, the email provider,
the scheduler and the store behind the seam.

## Requirements

Next.js App Router, the `stripe` package, and either `firebase-admin` or a Postgres client (`pg`, or the
host's own). Vitest for the shipped suites. A scheduler that can call one route every five minutes, such as
Vercel Cron. Nothing else is added: time zones use the platform's `Intl`, and request parsing is
dependency-free.

## Verification

Every TypeScript block in `references/` names its destination on its first line and is written to compile
as one project under `strict` and `noUncheckedIndexedAccess`. Before a release the blocks are written to a
scratch project, type-checked both ways, and the three suites run under Vitest: 22 core tests, 8
state-machine tests and 17 lifecycle tests, 47 in all. The `pg` client and the SQL are checked by reading.
`CLAUDE.md` has the recipe.

## Not this

| Not this | Use instead |
|---|---|
| Marketplace splits, payouts, connected accounts, subscriptions | The sibling [`stripe-connect-subscriptions`](https://github.com/timerise-ai/stripe-connect-subscriptions) skill |
| Walk-up touchscreen registration | The sibling [`booking-kiosk`](https://github.com/timerise-ai/booking-kiosk) skill, which can call this engine |
| Hourly resource booking (lanes, courts, rooms) | The host's booking engine; events only block its slots |
| Seat maps, ticket resale | A ticketing platform |

## Contributing

Issues and pull requests are welcome here. Pure markdown, with no build step, but the code blocks are checked:
every block names its destination on the first line, and every TypeScript block is written to compile as one
project under `strict` and `noUncheckedIndexedAccess` and to pass its suites under Vitest, 47 tests. Claims in
this skill are meant to be verifiable: if you change a factual claim, say how you verified it, whether against
the Stripe API reference, Stripe test mode, the card networks' authorization rules as Stripe documents them,
the IANA time zone database, or a reproduction.

Adding, removing or renaming a file in `references/` means updating the quick start and the reference
directory table in `SKILL.md`, the file table above, and any relative cross-links. Every odd-looking part of
the templates is there for a reason, and `references/provenance.md` is the ledger that must stay truthful:
read it before simplifying anything, and add an entry for anything you change. Commits follow Conventional
Commits and releases follow [STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) in the
index; `CLAUDE.md` carries the full editing conventions.

## Part of the Timerise Skills

This is one of the [Timerise Skills](https://github.com/timerise-ai/skills): modules for **Next.js App
Router** apps written by our own senior engineers from the modules they have shipped, not synthetic, each
published as its own repository and indexed there. They share one layout, so an agent that has read one knows
how to read the next: a `SKILL.md` entry point, `references/` loaded on demand, and a seam contract carrying
the module's non-negotiables.

## Author

Built and maintained by [Timerise](https://timerise.ai).

## License

MIT. See [LICENSE](LICENSE).
