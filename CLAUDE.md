# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A **Claude Skill package**, not an application. It ships prose + TypeScript
templates that teach another agent how to build bookable events with Stripe
payments and no-show deposits in *someone else's* Next.js codebase. There is no
`package.json`, build or dev server here. The version of record is the git tag.

```
SKILL.md                  entry point: frontmatter trigger + critical facts + hard rules + routing table
references/*.md           14 topic files + provenance.md, loaded on demand by the routing table
CHANGELOG.md              Keep a Changelog; one section per git tag
README.md                 human-facing: install, contents table, the non-negotiables
LICENSE                   MIT, identical to the siblings'
```

Sibling directories under `../` (`booking-kiosk`, `stripe-connect-subscriptions`,
`digital-signage`, …) are other skills, not dependencies. `booking-kiosk` and
`stripe-connect-subscriptions` are referenced by name from `SKILL.md`.

## Verifying the templates

Every TypeScript block in `references/` is a complete file whose first line is
a `// path` comment. To verify after an edit, extract each block to that path
in a scratch project, symlink a `node_modules` that has `stripe`,
`firebase-admin`, `next` and `vitest`, and run:

```bash
tsc --noEmit -p . && tsc --noEmit -p . --noUncheckedIndexedAccess && vitest run
```

Both type-checks must be clean and all 47 tests must pass. Blocks that need a
package the scratch project lacks (the `pg` client in `postgres.md`) and the
SQL are checked by reading. Do not add a `package.json` to this repository.

## Editing rules

**SKILL.md is a router, ≤150 lines.** Deep material belongs in `references/`.

**Three places index the references**: the *Quick start* list and *Reference
directory* table in SKILL.md, and the *What's inside* table in README.md.
Cross-links between references are relative (`[stripe.md](stripe.md)`); links
from SKILL.md are `references/`-prefixed.

**`references/provenance.md` is the audit ledger.** It never names the original
source's paths or domain terms. A new source defect fixed → a numbered entry
under *Fixed in the templates*; a questionable choice kept → *Kept
deliberately* with the reason; new capability → a row under *Added*.

**Test counts are claims.** `testing.md` states 22 + 8 + 17 = 47 tests;
`README.md` and `CHANGELOG.md` repeat the totals. Change a suite, change the
numbers.

## Content invariants

- **Vocabulary**: event / participant / ticket / deposit / check-in, scoped by
  `locationId`. The rename table is in `references/data-model.md`.
- **One state machine.** `applyAction` is pure and returns the same object for
  an already-applied action. Nothing writes `payment`, `deposit` or
  `attendance` outside `engine.transition`.
- **Transaction first, Stripe second.** No Stripe call inside `mutate`.
- **Hold timing constants** — 6-day Checkout-hold window, 4-day off-session
  window, 2 h settle grace, 6 h capture safety, 24 h cutoff, 30 min Checkout
  sessions, 5 min tick, 10 min retry backoff — appear in
  `deposit-schedule.ts`, `effects.ts`, `stripe-gateway.ts`,
  `no-show-deposits.md`, `participant-lifecycle.md`, `stripe.md` and
  `operations.md`. Change one, change all.
- **Currency**: lowercase ISO codes, `usd` default, stored in the currency's
  real minor unit, converted to Stripe units only in the gateway.
- **Env var names** in `references/operations.md` are canonical:
  `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `EVENTS_MANAGE_TOKEN_SECRET`,
  `CRON_SECRET`, `APP_BASE_URL`.
- **The Adaptation Contract table in SKILL.md is the full host boundary.** A
  template that needs something new from the host adds a row there.
