# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.3] - 2026-09-28

Fix release, from scoring the prompt-1 agent eval runs against 0.1.2. Apps built from 0.1.2 should copy in
the new `webhook.ts`; apps built from any earlier version, also `engine.ts` and the success URL in `host.ts`.

### Fixed
- The `charge.refunded` for a refunded second payment no longer marks the paid seat refunded: the handler
  ignores a payment intent that is not the participant's own (`stripe.md`). The second-payment lifecycle test
  now delivers that webhook.
- The Checkout success URL carries the manage token, so a guest's "registered" page can read their status
  (`engine.md`, `api-routes.md`).

### Changed
- `SKILL.md` quick start: no extra checks on the env vars, and the final message itself names what the
  operator must set up.
- `api-routes.md`: the runtime block in `host.ts` is copied as shipped apart from the store line.

## [0.1.2] - 2026-09-28

Fix release, from scoring the prompt-1 agent eval runs against 0.1.1. Apps built from 0.1.0 or 0.1.1 should
copy in the new `participant-machine.ts`, `engine.ts`, `webhook.ts`, `tick.ts` and the check-in list route.

### Fixed
- "Pay again" no longer loses the seat when the old session's `expired` webhook arrives before the new
  session is stored: resume clears the session ids first, and `CHECKOUT_EXPIRED` names its session, checked
  inside the transaction (`participant-lifecycle.md`, `engine.md`, `stripe.md`, `no-show-deposits.md`).
- A second payment for a seat already paid is refunded instead of failing the webhook until Stripe gives up
  (`participant-lifecycle.md`).
- Cancelling an event refunds guests who already checked in; only a customer's own cancel is refused after
  arrival (`participant-lifecycle.md`).
- The check-in list route returns the event name, date, times and zone its screen shows (`api-routes.md`).
- Three lifecycle tests hold these: 22 + 8 + 20 = 50 tests.

### Changed
- `SKILL.md` quick start: copy the blocks verbatim and edit only `host.ts`; run the fixtures and suites as
  written; keep a template you think is wrong and say why in the handover; list the five env vars in
  `.env.example` with no fallback and no other name; what the handover must name.
- An app without login keeps `getCustomer` and `getStaff` returning `null` and never takes identity from
  headers, query, body or a shared key (`SKILL.md` adaptation contract, `api-routes.md`).
- `operations.md` states the five env vars are the whole contract.

## [0.1.1] - 2026-09-28

Documentation-only release: the templates and references are unchanged from
0.1.0. The repository now carries the agent eval prompts and workflow the skill
standard requires, and the front door describes them.

### Changed
- `SKILL.md` adaptation contract names the sibling `ledger-wallet` skill as an
  implementation of the `BalanceLedger` port.
- `README.md` file table lists `evals/` and the agent eval workflow.
- `CLAUDE.md` describes `evals/` and the eval workflow, which is copied
  verbatim from the standard and never edited.

## [0.1.0] - 2026-09-25

Initial release of the `bookable-events` skill: capacity-limited registration
for free and paid events, Stripe Checkout tickets and no-show deposits, for a
Next.js App Router app on Firestore or Postgres.

### Added
- `SKILL.md` entry point: architecture and the adaptation contract, six critical
  facts, five hard rules, the quick-start order and the reference directory.
- Fourteen references: data model, money and time, participant lifecycle,
  no-show deposits, Stripe, registration, engine, Firestore store, Postgres
  store, API routes, UI, testing, testing lifecycles, operations; plus
  `provenance.md`, the audit ledger of the earlier implementation.
- No-show deposits on free events: Checkout hold when the event settles within
  6 days, saved card and off-session hold otherwise; release on check-in,
  capture on no-show or late cancel, partial capture, waivers, renewal, cutoff
  release.
- Any ISO-4217 currency with `usd` by default, including zero-decimal,
  three-decimal and the ISK and UGX Stripe representation.
- 47 Vitest tests (core, state machine, lifecycles) shipped with the templates.
- `README.md`, MIT `LICENSE`, `CLAUDE.md`.
