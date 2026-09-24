# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

An [Agent Skill](https://agentskills.io) package: markdown only. There is no `package.json`, build or dev
server here, and nothing in this repository executes. It teaches an agent to build bookable events with
Stripe payments and no-show deposits in *someone else's* **Next.js App Router** app, on Firestore or
Postgres. The version of record is the git tag.

The commands and code in `references/` describe the app the agent will generate, not this repository. The
one thing checked here is that the templates compile and their tests pass, in a scratch project; the recipe
is below.

The skill was written by the engineer who owns the module, and audited against the earlier implementation.
`references/provenance.md` is the rationale layer and the ledger of that audit: what the audit changed and
how the templates verify it, what was kept deliberately with the reason it is safe, and what was designed
here and has never run in production. Read it before "simplifying" anything.

Sibling directories under `../` (`booking-kiosk`, `stripe-connect-subscriptions`, `digital-signage`, and so
on) are other skills, not dependencies. `booking-kiosk` and `stripe-connect-subscriptions` are referenced by
name from `SKILL.md` and `README.md`. The standard every skill follows is `../skills/STANDARD.md`.

## Structure

- `SKILL.md`: entry point, loaded whole on every activation, so it stays between 130 and 160 lines, the
  closing index line aside. The frontmatter `description` is the trigger surface. The body carries the
  architecture and, under it, the **adaptation contract** table: this skill keeps its seam there instead of
  in a `references/adaptation.md`, and that table is the full host boundary. It closes with a line linking
  the skills index.
- `README.md`: the human-facing front door, in the section order every Timerise skill shares.
- `CHANGELOG.md`: Keep a Changelog, newest release first. The version lives here, in the README's
  current-release line, and in the git tag, and the three agree.
- `LICENSE`: MIT, identical to the siblings'.
- `references/*.md`: fourteen topic files plus `provenance.md`, loaded on demand. `data-model.md` holds the
  rename table; `money-and-time.md` and `participant-lifecycle.md` the pure core; `engine.md`,
  `registration.md`, `stripe.md` and `no-show-deposits.md` the flows; `firestore.md` and `postgres.md` the
  stores; `api-routes.md` and `ui.md` the surface; `testing.md` and `testing-lifecycles.md` the suites;
  `operations.md` running it; `provenance.md` the ledger.

## Editing conventions

- **Code blocks name their destination on the first line**: `// lib/events/types.ts: description`. The path
  is the first word after `//`, without the trailing colon. A block that continues a file already
  introduced omits it.
- **The code blocks are compiled and run.** Write each block to its path in a scratch project (use the
  scratchpad, never this repo), symlink a `node_modules` that has `stripe`, `firebase-admin`, `next` and
  `vitest`, and run:

  ```bash
  tsc --noEmit -p . && tsc --noEmit -p . --noUncheckedIndexedAccess && vitest run
  vitest run test/engine.test.ts        # one suite
  vitest run -t 'name of the test'      # one test
  ```

  No `tsconfig.json` ships with the templates. The scratch one should be `strict` and must map the `@/`
  alias the imports use (`"paths": { "@/*": ["./*"] }`); `vitest.config.ts` (in `testing.md`) maps the same
  alias for the tests. Both type-checks must be clean and all 47 tests must pass. Blocks that need a package
  the scratch project lacks (the `pg` client in `postgres.md`, and `postgres-store.ts` which imports it),
  `vercel.ts`, and the SQL are checked by reading. Do not add a `package.json` to this repository.
- **Test counts are claims.** `testing.md` states 22 + 8 + 17 = 47 tests; `README.md`, `SKILL.md` and
  `CHANGELOG.md` repeat the totals. Change a suite, change the numbers.
- **Keep the three indexes in sync** with `references/`: the quick start and the reference directory in
  `SKILL.md`, and the file table in `README.md`. Cross-links between references are relative
  (`[stripe.md](stripe.md)`); links from `SKILL.md` are `references/`-prefixed.
- **Identifiers are shared across files.** `applyAction`, `effectsOf`, `seatDelta`, `createEventsEngine`,
  `EventStore`, `PaymentGateway`, `StripeGateway`, `BalanceLedger`, `Notifier`, `requireStaffForEvent`,
  `nextActionAt`, the participant states and action names. Rename in all files or none.
- **Two kinds of name.** The domain vocabulary the host renames is the rename table in `data-model.md`:
  event, participant, ticket, deposit, check-in, scoped by `locationId`. The identifiers above are the
  authoring contract of this repository, which the host may rename in its own app but which must stay
  consistent here.
- **One state machine.** `applyAction` is pure and returns the same object for an already-applied action.
  Nothing writes `payment`, `deposit` or `attendance` outside `engine.transition`.
- **Transaction first, Stripe second.** No Stripe call inside `mutate`.
- **The non-negotiables are never presented as optional.** The five hard rules in `SKILL.md` and the five
  non-negotiables in `README.md` are one list, in one order. Changing one is a MAJOR release and says what
  broke.
- **Measured numbers are load-bearing.** The hold timing constants (6-day Checkout-hold window, 4-day
  off-session window, 2 h settle grace, 6 h capture safety, 24 h cutoff, 30 min Checkout sessions, 5 min
  tick, 10 min retry backoff) appear in `deposit-schedule.ts`, `effects.ts`, `stripe-gateway.ts`,
  `no-show-deposits.md`, `participant-lifecycle.md`, `stripe.md` and `operations.md`. Change one, change
  all. Do not restate a number loosely and do not invent new ones.
- **Currency**: lowercase ISO codes, `usd` default, stored in the currency's real minor unit, converted to
  Stripe units only in the gateway.
- **Env var names** in `references/operations.md` are canonical: `STRIPE_SECRET_KEY`,
  `STRIPE_WEBHOOK_SECRET`, `EVENTS_MANAGE_TOKEN_SECRET`, `CRON_SECRET`, `APP_BASE_URL`.
- **A template that needs something new from the host adds a row** to the adaptation contract in `SKILL.md`.
- **Do not remove the odd-looking parts.** The unhandled `payment_intent.payment_failed`, the seat taken
  before Checkout, the denormalized seat counter, attempt-scoped idempotency keys, the stale-claim recovery.
  Each is a ledger entry. Check `provenance.md` before touching one.
- **`provenance.md` stays truthful.** It never names the earlier implementation's paths or domain terms. A
  new defect fixed is a numbered entry under *Fixed in the templates*; a questionable choice kept goes under
  *Kept deliberately* with the reason; a new capability is a row under *Added*.
- **Origin wording.** Call the codebase the audit ran against "the earlier implementation", and follow
  section 2 of the standard: none of its banned words, no denials, no corpus figures, and no defect counts
  or failure stories on the front door (README intro, `SKILL.md`, the description, this file).
- **Plain punctuation.** No em-dashes, en-dashes, arrows, middle dots, ellipsis characters or smart quotes
  anywhere in this repository's markdown, code blocks included; diagrams are plain ASCII. The only
  non-ASCII characters are those in proper names; a currency symbol in code is written as an escape.
  `LC_ALL=C grep -rn '[^ -~	]' --include='*.md' .` must print nothing. Prose wraps at 110 columns; table
  rows and commands stay on one line.
- **Claims are verifiable.** A changed factual claim says how it was verified: against the Stripe API
  reference, Stripe test mode, the IANA time zone database, or a reproduction. Never from memory.
- **Commits follow Conventional Commits**, and no file or commit message names a tool or a model as author.
