# Provenance

This is the engineering ledger for the person editing the skill. The templates were written by the engineer
who owns the events module of a multi-location venue-booking system (Next.js App Router, React, Firestore,
Stripe), and audited against that earlier implementation. It ran event types, capacity-limited
registration, free, card-paid and wallet-paid tickets, a Stripe webhook, customer cancellation with wallet
credit, and a cleanup cron for unpaid seats. The audit made three passes: correctness, code quality and
operator usability.

The ledger separates three things:

- **Fixed in the templates**: what the audit changed, and where the templates and their tests hold it.
- **Kept deliberately**: earlier choices that look questionable and stay, with the reason each is safe.
- **Added**: what was designed in the skill and has never run in production.

## Fixed in the templates

### 1. A paid event was free in any currency it had no price in
Registration treated a missing or zero price in the *chosen* currency as "free",
and the admin form pre-filled `0` for every enabled currency. Cancelling that
free seat then credited the wallet with the price in the default currency: a
loop that minted stored credit.
**Shipped:** `offeredCurrencies` / `resolveCharge` refuse unpriced currencies;
free means free in every currency; refunds use the frozen `payment.amount`
([money-and-time.md](money-and-time.md)). Regression tests in both suites.

### 2. One declined card attempt deleted the seat
`payment_intent.payment_failed`, sent for each failed attempt while Checkout is
still open, deleted the participant. A successful retry then charged the
customer for a seat that no longer existed, and the error was swallowed.
**Shipped:** that event is deliberately not handled; only session expiry ends a
PENDING registration; late money is refunded ([stripe.md](stripe.md)).

### 3. Session expiry was not handled for events
The expiry handler covered other payment types only, so abandoned seats waited
for the cron. **Shipped:** `checkout.session.expired` moves to `CHECKOUT_EXPIRED` for
the participant's current session.

### 4. Cleanup could delete a participant who had just paid
The cron read the status outside the transaction and deleted inside it without
re-checking. **Shipped:** every change is one transaction around the pure state
machine; the tick reconciles with Stripe before expiring and fulfils a
completed session it finds ([participant-lifecycle.md](participant-lifecycle.md)).

### 5. Refunds were never refunds
Card-paid seats were "refunded" as wallet credit at the *current* price; guests
without a wallet got nothing while the response reported a refund; a failed
credit was swallowed after the status flipped; Dashboard refunds left the seat
confirmed. **Shipped:** real Stripe refunds of the frozen amount, a durable
`REFUND_PENDING` retried by the tick, `charge.refunded` handling
([registration.md](registration.md)).

### 6. Wallet payments were not atomic
Balance check, seat, then debit: three writes; a crash in between left a paid
seat unpaid, and the debit had no reference to the seat. **Shipped:** debit with
ref `event:<participantId>`; the tick settles an interrupted one from the ledger.

### 7. No webhook idempotency, and errors answered 200
**Shipped:** event-id claims with stale-claim recovery, no-op transitions,
idempotency keys on every Stripe write, and 500 on handler failure so Stripe
retries.

### 8. The admin update wrote the raw request body, without a location check
Any field (seat counter, deleted flag, location) could be overwritten, by an
admin of any location. **Shipped:** `parseEventInput` whitelist, capacity at least
seats taken in the transaction, `requireStaffForEvent` on every `[eventId]` route
([api-routes.md](api-routes.md)).

### 9. Cancelling an event refunded nobody
A status flip left every participant confirmed and charged. **Shipped:** a
cancel-event action that closes registration, then refunds and releases every
seat; the edit form cannot set `cancelled`.

### 10. Registration trusted a player id from the request body
Anyone could attach a registration to another account, whose owner could then
cancel it and collect the refund. **Shipped:** identity from the session only;
guests use an HMAC manage link.

### 11. Unpaid seats outlived failed session creation
The seat was written before Checkout, and a Checkout error (for example a
payment method valid for only one currency) left it held for half an hour.
**Shipped:** the seat is released the moment session creation fails;
payment methods follow the Dashboard instead of a hard-coded list.

### 12. "Pay again" could never work
The cleanup ran 30 minutes after registration while sessions lived 24 hours, and
a new session did not expire the old one, leaving two payable sessions per seat.
**Shipped:** 30-minute sessions, attempt-scoped idempotency keys, old session
expired on resume, expiry honoured only for the current session.

### 13. The cron was open when its secret was unset
**Shipped:** fail closed, constant-time comparison.

### 14. Times were UTC pretending to be local
"Today" was UTC's date; slot instants were built by appending `Z` to local
wall-clock strings. **Shipped:** IANA zone on every event, `zonedToUtc`,
`todayIn` ([money-and-time.md](money-and-time.md)).

### 15. Smaller defects
A slot-block setting could not be switched off once set (dropped `undefined`
key); the customer lookup for Stripe matched by email only; draft and deleted
events were publicly readable; event emails did not exist; the success redirect
landed on the home page with a query parameter nothing read; participant lists
were hard-coded in one language; the paid amount and currency were not shown to
staff. Slot blocking stays with the host's booking module, so the first item
is out of scope here; each of the others is addressed in the corresponding
reference.

## Kept deliberately

| Choice | Why it stays |
|---|---|
| Denormalized seat counter on the event | Capacity must be checked in one transaction without scanning participants; it is safe because only `seatDelta` changes it. |
| Seat taken before Checkout | The alternative double-sells the last seat. Made safe by 30-minute sessions and immediate release on failure. |
| Participants as a subcollection / one jsonb row | The machine reads and writes a participant whole. |
| Stored-credit wallet as a payment method | Useful to venues with prepaid balances; kept as the optional `BalanceLedger` port. |
| Up to 10 tickets per registration | Bounds one request's effect on capacity; configurable. |

## Added

Designed in the skill and covered by its tests; none of it ran in the earlier implementation.

| Addition | Where |
|---|---|
| No-show deposits: hybrid card hold (Checkout hold within 6 days, saved card and off-session hold otherwise), release on check-in, capture on no-show or late cancel, partial capture, waivers, renewal for long events, cutoff release | [no-show-deposits.md](no-show-deposits.md) |
| Staff check-in with arrival counts | [ui.md](ui.md), [api-routes.md](api-routes.md) |
| Any ISO-4217 currency, USD default, 0, 2 and 3-decimal handling | [money-and-time.md](money-and-time.md) |
| One due-work pointer and a single tick | [participant-lifecycle.md](participant-lifecycle.md) |
| Postgres store | [postgres.md](postgres.md) |
| Guest manage links | [registration.md](registration.md) |
| Lane and slot blocking during events stays with the host's booking module | integration point only |

## Order of work on an existing module

When upgrading a module that still has these defects, fix them in this order, most money at risk first:

1. Unpriced-currency free tickets and the price-based wallet refund (entry 1).
2. Seat deletion on `payment_intent.payment_failed` (entry 2), then session expiry (entry 3).
3. The cleanup race (entry 4) and the unverified player id (entry 10).
4. The raw-body admin update and missing location checks (entry 8).
5. Cancel-event refunds (entry 9), unpaid-seat release (entry 11), pay-again (entry 12), cron auth (entry 13).
6. Real refunds (entry 5), wallet atomicity (entry 6), webhook idempotency (entry 7).
7. Time zones (entry 14) and the smaller items (entry 15).
