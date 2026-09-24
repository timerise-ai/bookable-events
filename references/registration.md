# Registration, cancellation and refunds

The customer-facing flows and the rules they enforce. The code that runs them is
in [engine.md](engine.md); the request parsing is at the bottom of this file.

## Registration

```
 POST /api/events/{eventId}/register  { fullName, email, phone?, ticketCount, currency?, paymentMethod? }
   │ parseRegistrationBody            → 400 { field, code }
   │ customer = getCustomer(req)      → identity from the session ONLY
   ▼
 engine.register
   1 event exists, not deleted, published, not started        → 404 / 409
   2 resolveCharge(event, currency, tickets, policy)          → 400 currency_*
   3 build participant (free: CONFIRMED; else PENDING)
   4 store.insertParticipant — capacity checked IN the tx      → 409 no_spots
   5 branch:
       free, no deposit → email "registered"          → { checkoutUrl: null }
       paid + balance   → ledger.debit(ref event:<id>) → CONFIRMED or CANCEL + 402
       paid + card      → ticket Checkout (30 min)     → { checkoutUrl }
       deposit          → hold or setup Checkout        → { checkoutUrl }
   6 Checkout creation throws → CANCEL (seat back now) → 502 payment_unavailable
```

| Guard | Why it is where it is |
|---|---|
| Capacity inside the insert transaction | A check before the transaction is advisory only; two requests can both see "1 left". |
| Seat taken **before** Checkout, released on failure | The alternative (Checkout first) lets two customers pay for the last seat. Releasing immediately on a Stripe error avoids a phantom seat for 30 minutes. |
| `event_started` check | A still-`published` event from yesterday must not take registrations. |
| `resolveCharge` rejects unpriced currencies | See [money-and-time.md](money-and-time.md) — the "free by currency switch" defect. |
| No `customerId` / `playerId` from the body | A forged id attaches the registration — and its refund — to someone else's account. |
| Max 10 tickets per registration (configurable) | Bounds one request's effect on capacity and on a deposit total. |

**Guests** can register. They manage the seat through the link in every email:
`/events/{eventId}/manage/{participantId}?t={token}` where the token is
`HMAC-SHA256(EVENTS_MANAGE_TOKEN_SECRET, participantId)`, compared in constant
time. Signed-in customers are recognised by `customerId` instead.

**Balance payments** (optional `BalanceLedger`) debit with ref
`event:<participantId>`. If the process dies between debit and confirm, the
tick finds the PENDING participant, asks `ledger.hasEntry(ref)` and confirms or
expires accordingly — the debit and the seat can never disagree for longer than
one tick.

**Pay again.** `POST …/participants/{id}/resume` bumps `checkoutAttempt`,
expires the previous session and returns a new Checkout URL. It works for a
`PENDING` registration and for a deposit whose off-session hold failed
(`HOLD_FAILED`, `HOLD_REQUIRES_ACTION`) — in which case the new session is a
hold Checkout, and completing it supersedes the failed intent.

## Cancellation

| Who | Route | Refund policy |
|---|---|---|
| Customer / guest | `POST /api/events/{e}/participants/{p}/cancel` | Before `cancelDeadlineHours`: full refund, hold released. After: no refund, hold captured. After start: refused (`event_started`). After check-in: refused (`already_arrived`). |
| Staff | `POST /api/staff/events/{e}/participants/{p}/cancel { refund: 'full' \| 'none' }` | Staff choose; holds are released (staff cancel is never "late"). |
| Admin, whole event | `POST /api/admin/events/{e}/cancel` | Status set to `cancelled` first (closes registration), then every seat-holding participant: full refund, hold released. Re-runnable. |

Setting `status: 'cancelled'` through the event edit form is refused
(`use_cancel_endpoint`): a bare status flip is how a cancelled event ends up
with nobody refunded.

## Refunds

| Paid with | Refund | Completes |
|---|---|---|
| Card (Stripe) | `refunds.create({ payment_intent, amount })`, idempotency key `refund:<participantId>` | Synchronously if Stripe says `succeeded`, else on `charge.refunded` |
| Balance | `ledger.credit(ref refund:<participantId>)` | Synchronously |
| Free | — | — |

The amount is always `payment.amount − payment.refundedAmount`: what this
participant was charged, frozen at registration. A refund initiated in the
Stripe Dashboard is picked up by `charge.refunded` and recorded as
`EXTERNAL_REFUND`; a full one releases the seat.

Refunding to the original card is the default. A host that prefers store
credit for Stripe-paid tickets swaps the `refund` effect's STRIPE branch for a
`ledger.credit` — keep the idempotency ref, and keep refunding guests (who have
no wallet) to their card.

## Request parsing

`parseEventInput` is the admin form's whitelist: unknown keys are dropped, so
`seatsTaken`, `locationId`, `deletedAt` and `createdAt` can never be written
from a request body, and `capacity` below the current `seatsTaken` is refused.

```ts
// lib/events/input.ts — request bodies → typed values. Whitelists: unknown keys are dropped,
// so an admin PUT can never overwrite seatsTaken, deletedAt, locationId or createdAt.
import { assertChargeableAmount, normalizeCurrency } from './currency';
import { isFreeEvent } from './pricing';
import { isValidDate, isValidTime, isValidTimeZone } from './time';
import type { EventStatus, LocalizedText, PriceMap } from './types';

export type Parsed<T> = { ok: true; value: T } | { ok: false; field: string; code: string };

const EMAIL_RE = /^[^\s@]+@[^\s@]+\.[^\s@]{2,}$/;
const bad = (field: string, code: string) => ({ ok: false as const, field, code });

function obj(v: unknown): Record<string, unknown> {
  return v && typeof v === 'object' && !Array.isArray(v) ? (v as Record<string, unknown>) : {};
}

function str(v: unknown, max: number): string | null {
  if (typeof v !== 'string') return null;
  const t = v.trim();
  return t.length > 0 && t.length <= max ? t : null;
}

export interface RegistrationBody {
  fullName: string;
  email: string;
  phone: string | null;
  ticketCount: number;
  currency: unknown;
  method: 'STRIPE' | 'BALANCE';
  locale: string;
}

export function parseRegistrationBody(raw: unknown, maxTickets = 10): Parsed<RegistrationBody> {
  const b = obj(raw);
  const fullName = str(b.fullName, 120);
  if (!fullName || fullName.length < 2) return bad('fullName', 'required');
  const email = str(b.email, 254)?.toLowerCase();
  if (!email || !EMAIL_RE.test(email)) return bad('email', 'invalid');
  const phone = b.phone === undefined || b.phone === null || b.phone === '' ? null : str(b.phone, 32);
  if (b.phone && !phone) return bad('phone', 'invalid');
  const ticketCount = b.ticketCount === undefined ? 1 : b.ticketCount;
  if (typeof ticketCount !== 'number' || !Number.isInteger(ticketCount) || ticketCount < 1 || ticketCount > maxTickets) {
    return bad('ticketCount', 'out_of_range');
  }
  const locale = str(b.locale, 16) ?? 'en';
  return {
    ok: true,
    value: { fullName, email, phone, ticketCount, currency: b.currency, method: b.paymentMethod === 'BALANCE' ? 'BALANCE' : 'STRIPE', locale },
  };
}

function parsePriceMap(raw: unknown, field: string, allowed: string[]): Parsed<PriceMap> {
  const out: PriceMap = {};
  for (const [k, v] of Object.entries(obj(raw))) {
    const c = normalizeCurrency(k);
    if (!c) return bad(field, 'currency_invalid');
    if (allowed.length > 0 && !allowed.includes(c)) return bad(field, 'currency_not_enabled');
    if (typeof v !== 'number') return bad(field, 'amount_invalid');
    try {
      assertChargeableAmount(v, c);
    } catch {
      return bad(field, 'amount_invalid');
    }
    out[c] = v;
  }
  return { ok: true, value: out };
}

function parseLocalized(v: unknown, max: number): LocalizedText | null {
  if (typeof v === 'string') return str(v, max);
  const entries = Object.entries(obj(v)).filter((e): e is [string, string] => typeof e[1] === 'string' && e[1].trim() !== '');
  return entries.length > 0 && entries.every(([, t]) => t.length <= max) ? Object.fromEntries(entries) : null;
}

export interface EventInput {
  typeId: string | null;
  name: LocalizedText;
  description: LocalizedText;
  imageUrl: string | null;
  date: string;
  timeStart: string;
  timeEnd: string;
  timeZone: string;
  capacity: number;
  ticketPrice: PriceMap;
  deposit: { amountPerTicket: PriceMap } | null;
  cancelDeadlineHours: number;
  status: EventStatus;
}

const STATUSES: EventStatus[] = ['draft', 'published', 'cancelled', 'completed'];

/** `seatsTaken` guards capacity on update; pass 0 on create. `allowed` = host currency restriction. */
export function parseEventInput(raw: unknown, opts: { seatsTaken: number; allowedCurrencies: string[] }): Parsed<EventInput> {
  const b = obj(raw);
  const name = parseLocalized(b.name, 200);
  if (!name) return bad('name', 'required');
  const description = parseLocalized(b.description ?? '', 5000) ?? '';
  const date = typeof b.date === 'string' && isValidDate(b.date) ? b.date : null;
  if (!date) return bad('date', 'invalid');
  const timeStart = typeof b.timeStart === 'string' && isValidTime(b.timeStart) ? b.timeStart : null;
  if (!timeStart) return bad('timeStart', 'invalid');
  const timeEnd = typeof b.timeEnd === 'string' && isValidTime(b.timeEnd) ? b.timeEnd : null;
  if (!timeEnd) return bad('timeEnd', 'invalid');
  const timeZone = typeof b.timeZone === 'string' && isValidTimeZone(b.timeZone) ? b.timeZone : null;
  if (!timeZone) return bad('timeZone', 'invalid');

  const capacity = b.capacity;
  if (typeof capacity !== 'number' || !Number.isInteger(capacity) || capacity < 1 || capacity > 100_000) {
    return bad('capacity', 'out_of_range');
  }
  if (capacity < opts.seatsTaken) return bad('capacity', 'below_seats_taken');

  const price = parsePriceMap(b.ticketPrice, 'ticketPrice', opts.allowedCurrencies);
  if (!price.ok) return price;
  let deposit: EventInput['deposit'] = null;
  if (b.deposit !== undefined && b.deposit !== null) {
    const d = parsePriceMap(obj(b.deposit).amountPerTicket, 'deposit', opts.allowedCurrencies);
    if (!d.ok) return d;
    if (Object.values(d.value).some((v) => v <= 0) || Object.keys(d.value).length === 0) return bad('deposit', 'amount_invalid');
    // A paid ticket already deters no-shows, and Checkout cannot charge one amount and
    // authorize another in the same session. Deposits are for free events.
    if (!isFreeEvent({ ticketPrice: price.value })) return bad('deposit', 'paid_event');
    deposit = { amountPerTicket: d.value };
  }

  const cancelDeadlineHours = b.cancelDeadlineHours ?? 24;
  if (typeof cancelDeadlineHours !== 'number' || cancelDeadlineHours < 0 || cancelDeadlineHours > 720) {
    return bad('cancelDeadlineHours', 'out_of_range');
  }
  const status = STATUSES.includes(b.status as EventStatus) ? (b.status as EventStatus) : 'draft';
  const typeId = typeof b.typeId === 'string' && b.typeId ? b.typeId : null;
  const imageUrl = typeof b.imageUrl === 'string' && /^https:\/\//.test(b.imageUrl) ? b.imageUrl : null;

  return {
    ok: true,
    value: { typeId, name, description, imageUrl, date, timeStart, timeEnd, timeZone, capacity, ticketPrice: price.value, deposit, cancelDeadlineHours, status },
  };
}
```

## Checklist

- [ ] The register route takes identity from the session and ignores body ids.
- [ ] Every money path writes the participant before calling Stripe.
- [ ] The manage link is in every customer email.
- [ ] Cancel-event exists and the edit form cannot set `cancelled`.
- [ ] Refund amounts come from `payment.amount`, never from the event's current price.
