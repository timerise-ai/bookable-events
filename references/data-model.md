# Data model

Two entities carry the whole module: an **Event** (a dated, capacity-limited
occasion at one location) and a **Participant** (one registration for one or more
tickets). Everything else — payment, deposit, attendance — lives on the
participant as three sub-states that one pure state machine moves
([participant-lifecycle.md](participant-lifecycle.md)).

## Canonical vocabulary and rename table

The templates always use these names. Rename once, at adoption time, everywhere.

| Canonical | Meaning | Typical host names |
|---|---|---|
| `EventRecord` / event | Dated occasion with capacity | session, class, workshop, tournament, meetup, tasting |
| `Participant` | One registration, 1..N tickets | attendee, registration, booking, entry, signup |
| `EventType` | Optional category (`typeId`) | category, format, discipline |
| `locationId` | Tenant scope | venue, site, club, branch, workspace |
| `customerId` | Signed-in account that owns the seat | userId, memberId, playerId |
| ticket | One seat inside a registration | place, spot, pass |
| deposit | Refundable card hold against no-shows | security hold, no-show fee, guarantee |
| check-in | Staff marks arrival | attendance, arrival, door scan |

Do **not** rename Stripe terms (`PaymentIntent`, `capture_before`,
`SetupIntent`, `Checkout Session`) or currency terms — they belong to the
platform, not the domain.

## Types

```ts
// lib/events/types.ts — the neutral domain model. Every backend maps to this shape.

/** Lowercase ISO-4217 code, always produced by `normalizeCurrency`. */
export type CurrencyCode = string;

/** Amounts in the currency's ISO minor unit (cents for usd, yen for jpy, fils for kwd). */
export type PriceMap = Record<CurrencyCode, number>;

/** Plain string, or per-locale strings keyed by locale code. */
export type LocalizedText = string | Record<string, string>;

export type EventStatus = 'draft' | 'published' | 'cancelled' | 'completed';

export interface DepositConfig {
  /** Held per ticket. The currencies offered are the keys of this map. */
  amountPerTicket: PriceMap;
}

export interface EventRecord {
  id: string;
  locationId: string;            // tenant scope, always derived server-side
  typeId: string | null;
  name: LocalizedText;
  description: LocalizedText;
  imageUrl: string | null;
  date: string;                  // YYYY-MM-DD, local to timeZone
  timeStart: string;             // HH:MM, local to timeZone
  timeEnd: string;               // HH:MM, local to timeZone; earlier than timeStart = ends next day
  timeZone: string;              // IANA, e.g. "America/New_York"
  capacity: number;
  seatsTaken: number;            // denormalized; only ever changed inside the participant transaction
  ticketPrice: PriceMap;         // empty, or every value 0 → free event
  deposit: DepositConfig | null; // only allowed on free events (see validateEventInput)
  cancelDeadlineHours: number;   // customer cancels later than this before start → late cancel
  status: EventStatus;
  deletedAt: Date | null;
  createdAt: Date;
  updatedAt: Date;
}

export type PaymentStatus =
  | 'PENDING'         // seat held, waiting for Checkout / balance debit / card for deposit
  | 'CONFIRMED'       // seat is theirs
  | 'EXPIRED'         // checkout never completed; seat released
  | 'CANCELLED'       // cancelled; seat released; no money owed back (or none was taken)
  | 'REFUND_PENDING'  // cancelled; refund requested, waiting for Stripe / ledger
  | 'REFUNDED';       // cancelled; money returned

export type PaymentMethod = 'FREE' | 'STRIPE' | 'BALANCE';

export type DepositStatus =
  | 'NONE'                 // event has no deposit
  | 'AWAITING_CARD'        // Checkout (hold or setup) not completed yet
  | 'CARD_SAVED'           // card on file, hold placed later by the tick
  | 'HOLD_REQUIRES_ACTION' // off-session hold needs the customer (3DS) — email sent
  | 'HOLD_FAILED'          // off-session hold declined — email sent, retry link valid until cutoff
  | 'HELD'                 // authorization in place, capturable until captureBefore
  | 'RELEASING'            // cancel requested; the tick retries until Stripe confirms
  | 'RELEASED'             // hold cancelled (arrived, waived, early cancel, venue cancel)
  | 'CAPTURING'            // capture requested; the tick retries until Stripe confirms
  | 'CAPTURED';            // no-show or late cancel: venue kept captureAmount

export type DepositStrategy = 'hold_now' | 'save_card';

export type AttendanceStatus = 'UNKNOWN' | 'ARRIVED' | 'NO_SHOW';

export type Actor = 'customer' | 'staff' | 'admin' | 'system';

export type NextAction =
  | 'EXPIRE_PENDING'   // PENDING past expiresAt → reconcile with Stripe, then expire
  | 'PLACE_HOLD'       // CARD_SAVED / HOLD_FAILED → off-session hold or give up at cutoff
  | 'SETTLE'           // HELD after the event → capture no-shows / release arrivals
  | 'RETRY_RELEASE'
  | 'RETRY_CAPTURE'
  | 'RETRY_REFUND';

export interface ParticipantPayment {
  status: PaymentStatus;
  method: PaymentMethod;
  amount: number;                 // what was charged, frozen at registration — refunds use this
  checkoutSessionId: string | null;
  paymentIntentId: string | null;
  expiresAt: Date | null;         // PENDING only: Checkout expires_at (+ grace in the tick)
  refundedAmount: number;
}

export interface ParticipantDeposit {
  status: DepositStatus;
  strategy: DepositStrategy | null;
  amountPerTicket: number;
  amount: number;                 // amountPerTicket × ticketCount
  stripeCustomerId: string | null;
  paymentMethodId: string | null;
  checkoutSessionId: string | null;
  paymentIntentId: string | null;
  captureBefore: Date | null;     // from the charge — the authorization's real expiry
  holdDueAt: Date | null;         // save_card: when the tick places the hold
  holdCutoffAt: Date | null;      // no hold by then → seat released
  settleDueAt: Date | null;       // when the tick captures / releases
  captureAmount: number;          // requested or captured amount
  waivedBy: string | null;
  waiveReason: string | null;
  lastError: string | null;
}

export interface ParticipantAttendance {
  status: AttendanceStatus;
  arrivedCount: number;
  checkedInAt: Date | null;
  checkedInBy: string | null;
}

export interface Participant {
  id: string;
  eventId: string;
  locationId: string;             // copied from the event for scoped staff queries
  customerId: string | null;      // from the verified session — never from the request body
  fullName: string;
  email: string;
  phone: string | null;
  locale: string;
  ticketCount: number;
  currency: CurrencyCode | null;  // null only for a free event with no deposit
  payment: ParticipantPayment;
  deposit: ParticipantDeposit;
  attendance: ParticipantAttendance;
  checkoutAttempt: number;        // bumped on "pay again" — part of the Checkout idempotency key
  nextAction: NextAction | null;
  nextActionAt: Date | null;
  registeredAt: Date;
  cancelledAt: Date | null;
  cancelledBy: Actor | null;
  updatedAt: Date;
}
```

## Why the shape is what it is

| Decision | Reason |
|---|---|
| `date` + `timeStart` + `timeEnd` + `timeZone`, never a UTC timestamp | Staff type wall-clock times. A UTC instant stored at edit time silently moves by an hour when DST rules change or the admin's browser is in another zone. Convert with `eventWindowUtc` ([money-and-time.md](money-and-time.md)) at the moment you need an instant. |
| `timeEnd <= timeStart` means "ends next day" | Evening events past midnight need no extra field. |
| `seatsTaken` denormalized on the event | Capacity must be checked inside one transaction without scanning participants. It is written **only** by `insertParticipant` and `mutateParticipant` via `seatDelta` — never by an admin form. |
| `payment.amount` frozen at registration | Refunds use what was charged, not today's price. Editing a price after sales must not change anyone's refund. |
| `deposit.amount` frozen at registration | Same reason for holds and captures. |
| `currency: null` only for free, deposit-less events | Every money-bearing registration records the currency it was charged in; no fallback currency is ever guessed later. |
| Three sub-states instead of one status | "Paid but no-show", "free but hold failed", "refunded after arrival was impossible" are all real combinations. One enum explodes into dozens of values; three orthogonal ones stay readable. |
| `nextAction` + `nextActionAt` | One indexed field is the whole background-work queue. The tick asks "who is due?" with a single range query instead of five status-specific scans. |
| `customerId` from the session only | A body field lets anyone attach a registration to someone else's account (and their refund). |
| No manage-token column | Guest links use `HMAC(secret, participantId)`: recomputable for every email, verifiable in constant time, nothing to leak from the database. |
| `deletedAt` instead of `_deleted` boolean | A timestamp answers "when" for the audit trail and cannot drift from a separate flag. |

## Seat accounting

A participant **holds seats** exactly while `payment.status` is `PENDING` or
`CONFIRMED`. `seatDelta(before, after)` turns every transition into the change to
`seatsTaken`, applied in the same transaction:

| Transition | Seat effect |
|---|---|
| insert (any status) | `+ticketCount` (capacity checked) |
| `PENDING → CONFIRMED` | 0 |
| `PENDING → EXPIRED / CANCELLED` | `−ticketCount` |
| `CONFIRMED → CANCELLED / REFUND_PENDING / REFUNDED` | `−ticketCount` |
| anything → `PENDING / CONFIRMED` from a released state | **never** — the machine raises `LatePaymentError` and the money goes back |

The last row is the rule that makes the counter trustworthy: no transition ever
re-takes a seat without the capacity check, so `seatsTaken` can never exceed
`capacity` through the state machine.

## Deposit fields, in the order they fill

| Field | Set when |
|---|---|
| `strategy`, `amountPerTicket`, `amount`, `holdCutoffAt`, `holdDueAt` | Registration |
| `stripeCustomerId`, `checkoutSessionId` | Checkout (hold or setup) created |
| `paymentMethodId` | Card saved, or hold authorized |
| `paymentIntentId`, `captureBefore` | Hold authorized (Checkout or off-session) |
| `captureAmount` | Settlement or late cancel decides what to take |
| `waivedBy`, `waiveReason` | Staff waive |
| `lastError` | Off-session decline / 3DS / authorization lapse |

`settleDueAt` is refreshed on every write while the deposit is `HELD`, so staff
screens can show "no-shows charged at …". The tick reads the same value through
`nextActionAt` ([no-show-deposits.md](no-show-deposits.md)).

## Storage

The domain model is backend-neutral. Two reference stores implement the
`EventStore` contract: [firestore.md](firestore.md) and
[postgres.md](postgres.md). Both keep the participant as a whole document /
jsonb value plus a few indexed columns, because the state machine always reads
and writes the participant as a unit.
