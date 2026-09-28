# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
