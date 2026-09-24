# Money and time

Three small modules decide what a registration costs, in which currency, and
when "the event" actually is. All three are pure and tested
([testing.md](testing.md)); everything money- or clock-shaped in the engine goes
through them.

## Currency — any ISO-4217 code, USD by default

| Rule | Why |
|---|---|
| Codes are lowercase 3-letter ISO (`usd`, `eur`, `jpy`) | Stripe wants lowercase. `normalizeCurrency` is the only way in. |
| **Any** code is accepted; Stripe decides if it can charge it | No hard-coded list of 9 currencies to maintain. A host that wants a list sets `CurrencyPolicy.allowed`. |
| Default is `usd` | `DEFAULT_CURRENCY`; the host overrides with `CurrencyPolicy.defaultCurrency`. |
| Stored amounts use the currency's **real** minor unit | 500 ISK is stored as `500`, 1.250 KWD as `1250`, $19.99 as `1999`. Reports and UIs never need a special case. |
| Stripe amounts are converted only in the gateway | `toStripeAmount` / `fromStripeAmount` are called in exactly one file. |

Stripe's quirks the converter absorbs:

| Currency class | Examples | Stored | Sent to Stripe |
|---|---|---|---|
| Two-decimal (default) | usd, eur, gbp, pln, huf, twd | cents | cents |
| Zero-decimal | jpy, krw, clp, vnd, xof, … | whole units | whole units |
| Zero-decimal, legacy two-decimal on Stripe | **isk, ugx** | whole units | × 100 (`500 ISK` → `50000`) |
| Three-decimal | bhd, jod, kwd, omr, tnd | thousandths | thousandths, **last digit must be 0** |

`assertChargeableAmount` rejects a three-decimal price not ending in 0 **when the
price is saved**, not at checkout: an admin gets a field error instead of a
customer getting a Stripe error.

Stripe also enforces a minimum charge per currency (about $0.50 equivalent; for
example 0.50 USD/EUR, 0.30 GBP, 2.00 PLN, 50 JPY). Keep ticket prices and
deposits above it; a deposit of 1 unit is rejected at Checkout.

```ts
// lib/events/currency.ts — any ISO-4217 currency; USD by default.
//
// Two different "minor units" exist and must never be mixed:
//   stored amount — the currency's real minor unit (ISK 500 = 500 krónur, KWD 1500 = 1.500 dinar)
//   Stripe amount — what the Stripe API expects, which differs for ISK/UGX (×100) only
import type { CurrencyCode } from './types';

export const DEFAULT_CURRENCY: CurrencyCode = 'usd';

// Stripe's zero-decimal currencies, plus ISK (zero-decimal in practice, sent to Stripe ×100).
const ZERO_DECIMAL = new Set([
  'bif', 'clp', 'djf', 'gnf', 'isk', 'jpy', 'kmf', 'krw', 'mga', 'pyg',
  'rwf', 'ugx', 'vnd', 'vuv', 'xaf', 'xof', 'xpf',
]);
// Stripe accepts these with three decimals, but the last digit must be 0.
const THREE_DECIMAL = new Set(['bhd', 'jod', 'kwd', 'omr', 'tnd']);
// Zero-decimal currencies that Stripe still wants as two-decimal values ("500" = 5 ISK).
const STRIPE_TWO_DECIMAL_LEGACY = new Set(['isk', 'ugx']);

/** Lowercase 3-letter code, or null. Whether Stripe supports it is checked by Stripe. */
export function normalizeCurrency(input: unknown): CurrencyCode | null {
  if (typeof input !== 'string') return null;
  const code = input.trim().toLowerCase();
  return /^[a-z]{3}$/.test(code) ? code : null;
}

/** Decimal places of the stored amount. */
export function minorUnitExponent(currency: CurrencyCode): 0 | 2 | 3 {
  if (ZERO_DECIMAL.has(currency)) return 0;
  if (THREE_DECIMAL.has(currency)) return 3;
  return 2;
}

function stripeExponent(currency: CurrencyCode): 0 | 2 | 3 {
  if (STRIPE_TWO_DECIMAL_LEGACY.has(currency)) return 2;
  return minorUnitExponent(currency);
}

/** Reject a stored amount Stripe cannot charge. Call when prices are saved, not at checkout. */
export function assertChargeableAmount(amount: number, currency: CurrencyCode): void {
  if (!Number.isSafeInteger(amount) || amount < 0) {
    throw new RangeError(`Amount must be a non-negative integer, got ${amount}`);
  }
  if (THREE_DECIMAL.has(currency) && amount % 10 !== 0) {
    throw new RangeError(`${currency.toUpperCase()} amounts must end in 0 (Stripe rounds the 3rd decimal)`);
  }
}

export function toStripeAmount(amount: number, currency: CurrencyCode): number {
  assertChargeableAmount(amount, currency);
  return amount * 10 ** (stripeExponent(currency) - minorUnitExponent(currency));
}

export function fromStripeAmount(stripeAmount: number, currency: CurrencyCode): number {
  const factor = 10 ** (stripeExponent(currency) - minorUnitExponent(currency));
  if (stripeAmount % factor !== 0) {
    throw new RangeError(`Stripe amount ${stripeAmount} is not whole ${currency.toUpperCase()}`);
  }
  return stripeAmount / factor;
}

export function formatMoney(amount: number, currency: CurrencyCode, locale: string): string {
  const exp = minorUnitExponent(currency);
  return new Intl.NumberFormat(locale, {
    style: 'currency',
    currency: currency.toUpperCase(),
    minimumFractionDigits: exp,
    maximumFractionDigits: exp,
  }).format(amount / 10 ** exp);
}
```

## Pricing — the server decides, per currency, with no "free by accident"

A paid event is **never** free in a currency it has no price in. That single
rule is the difference between this module and one where a customer switches the
currency picker to an unpriced currency, registers for free, cancels, and is
credited the full price in the default currency.

| Event | `offeredCurrencies` | `resolveCharge(..., 'gbp')` |
|---|---|---|
| `ticketPrice: { usd: 2500, eur: 2300 }` | usd, eur | `currency_not_offered` |
| `ticketPrice: { usd: 2500, pln: 0 }` | usd | `pln` → `currency_not_offered` (a 0 on a paid event is not an offer) |
| `ticketPrice: {}` | — | `free`, any currency, `currency: null` |
| `ticketPrice: {}`, `deposit: { usd: 2000 }` | usd | `currency_not_offered`; `usd` → `deposit` |

When the client sends no currency, `resolveCharge` picks the policy default if
it is offered, else the first offered currency, so a single-currency event
"just works" without a picker.

```ts
// lib/events/pricing.ts — the server decides what a registration costs. Never the client.
import { DEFAULT_CURRENCY, normalizeCurrency } from './currency';
import type { CurrencyCode, EventRecord } from './types';

export interface CurrencyPolicy {
  /** Host restriction. Empty = any currency the event is priced in. */
  allowed: CurrencyCode[];
  defaultCurrency: CurrencyCode;
}

export const ANY_CURRENCY: CurrencyPolicy = { allowed: [], defaultCurrency: DEFAULT_CURRENCY };

export type ChargeResolution =
  | { ok: true; kind: 'free'; currency: null; amount: 0 }
  | { ok: true; kind: 'paid'; currency: CurrencyCode; amount: number; unitPrice: number }
  | { ok: true; kind: 'deposit'; currency: CurrencyCode; amount: 0; depositPerTicket: number; depositTotal: number }
  | { ok: false; error: 'currency_invalid' | 'currency_not_offered' | 'currency_not_enabled' };

/** Free means free in every currency — never "free in the currency the client picked". */
export function isFreeEvent(event: Pick<EventRecord, 'ticketPrice'>): boolean {
  return Object.values(event.ticketPrice).every((v) => v === 0);
}

/** Currencies a customer can register in. A 0 or missing price on a paid event is NOT an offer. */
export function offeredCurrencies(event: Pick<EventRecord, 'ticketPrice' | 'deposit'>): CurrencyCode[] {
  if (!isFreeEvent(event)) {
    return Object.entries(event.ticketPrice).filter(([, v]) => v > 0).map(([c]) => c);
  }
  if (event.deposit) {
    return Object.entries(event.deposit.amountPerTicket).filter(([, v]) => v > 0).map(([c]) => c);
  }
  return [];
}

export function resolveCharge(
  event: Pick<EventRecord, 'ticketPrice' | 'deposit'>,
  requestedCurrency: unknown,
  ticketCount: number,
  policy: CurrencyPolicy = ANY_CURRENCY,
): ChargeResolution {
  const offered = offeredCurrencies(event);
  if (offered.length === 0) return { ok: true, kind: 'free', currency: null, amount: 0 };

  let currency: CurrencyCode | null;
  if (requestedCurrency === undefined || requestedCurrency === null || requestedCurrency === '') {
    currency = offered.includes(policy.defaultCurrency) ? policy.defaultCurrency : (offered[0] ?? null);
  } else {
    currency = normalizeCurrency(requestedCurrency);
    if (!currency) return { ok: false, error: 'currency_invalid' };
  }
  if (!currency || !offered.includes(currency)) return { ok: false, error: 'currency_not_offered' };
  if (policy.allowed.length > 0 && !policy.allowed.includes(currency)) {
    return { ok: false, error: 'currency_not_enabled' };
  }

  if (!isFreeEvent(event)) {
    const unitPrice = event.ticketPrice[currency] ?? 0;
    return { ok: true, kind: 'paid', currency, amount: unitPrice * ticketCount, unitPrice };
  }
  const depositPerTicket = event.deposit?.amountPerTicket[currency] ?? 0;
  return {
    ok: true, kind: 'deposit', currency, amount: 0,
    depositPerTicket, depositTotal: depositPerTicket * ticketCount,
  };
}
```

## Time — wall-clock + IANA zone, converted on demand

The source system built instants as `` `${date}T${time}:00.000Z` `` — that is
UTC, not the venue's time, and every slot was off by the zone offset (twice a
year by a different amount). The fix is to store what staff type and convert
with the zone at the moment an instant is needed.

| Function | Use |
|---|---|
| `zonedToUtc(date, time, tz)` | Any wall-clock → instant conversion. DST-correct, no dependency. |
| `eventWindowUtc(event)` | `{ start, end }` for registration cutoffs, cancel deadlines, hold scheduling. |
| `todayIn(tz)` | "Upcoming events" lists. Never `new Date().toISOString().slice(0, 10)` — that is UTC's date, wrong for half the day in most zones. |
| `isValidTimeZone(tz)` | Admin form validation. |

A wall time skipped by the spring-forward jump (02:30 on a DST night) resolves
forward to 03:30. A repeated wall time in autumn resolves to the later
(standard-time) occurrence. Events rarely start at those minutes; the behaviour is defined so
the tick never crashes on them.

```ts
// lib/events/time.ts — event wall-clock times → UTC instants, DST-correct, no dependencies.
// Events store local date + time + IANA zone. Never build `${date}T${time}Z`: that is UTC, not local.
import type { EventRecord } from './types';

const DATE_RE = /^(\d{4})-(\d{2})-(\d{2})$/;
const TIME_RE = /^([01]\d|2[0-3]):([0-5]\d)$/;

export function isValidTimeZone(tz: string): boolean {
  try {
    new Intl.DateTimeFormat('en-US', { timeZone: tz });
    return true;
  } catch {
    return false;
  }
}

export function isValidDate(date: string): boolean {
  const m = DATE_RE.exec(date);
  if (!m) return false;
  const d = new Date(Date.UTC(Number(m[1]), Number(m[2]) - 1, Number(m[3])));
  return d.toISOString().slice(0, 10) === date;
}

export function isValidTime(time: string): boolean {
  return TIME_RE.test(time);
}

/** Offset of `tz` from UTC at instant `utcMs`, in ms (positive east of Greenwich). */
function tzOffsetMs(utcMs: number, tz: string): number {
  const parts = new Intl.DateTimeFormat('en-US', {
    timeZone: tz, hourCycle: 'h23',
    year: 'numeric', month: '2-digit', day: '2-digit',
    hour: '2-digit', minute: '2-digit', second: '2-digit',
  }).formatToParts(new Date(utcMs));
  const get = (type: Intl.DateTimeFormatPartTypes) =>
    Number(parts.find((p) => p.type === type)?.value ?? 0);
  const asUtc = Date.UTC(get('year'), get('month') - 1, get('day'), get('hour'), get('minute'), get('second'));
  return asUtc - Math.floor(utcMs / 1000) * 1000;
}

/** Local wall-clock date + time in `tz` → UTC Date. A time skipped by DST resolves forward. */
export function zonedToUtc(date: string, time: string, tz: string): Date {
  const dm = DATE_RE.exec(date);
  const tm = TIME_RE.exec(time);
  if (!dm || !tm) throw new RangeError(`Bad date/time: ${date} ${time}`);
  const guess = Date.UTC(Number(dm[1]), Number(dm[2]) - 1, Number(dm[3]), Number(tm[1]), Number(tm[2]));
  const first = tzOffsetMs(guess, tz);
  const second = tzOffsetMs(guess - first, tz);
  return new Date(guess - second);
}

export function addDaysToDate(date: string, days: number): string {
  const m = DATE_RE.exec(date);
  if (!m) throw new RangeError(`Bad date: ${date}`);
  const d = new Date(Date.UTC(Number(m[1]), Number(m[2]) - 1, Number(m[3]) + days));
  return d.toISOString().slice(0, 10);
}

/** Today's date in `tz` as YYYY-MM-DD — for "upcoming events" lists. */
export function todayIn(tz: string, now: Date = new Date()): string {
  return new Intl.DateTimeFormat('en-CA', { timeZone: tz, year: 'numeric', month: '2-digit', day: '2-digit' })
    .format(now);
}

export function eventWindowUtc(
  event: Pick<EventRecord, 'date' | 'timeStart' | 'timeEnd' | 'timeZone'>,
): { start: Date; end: Date } {
  const start = zonedToUtc(event.date, event.timeStart, event.timeZone);
  // An end time at or before the start time means the event runs past midnight.
  const endDate = event.timeEnd <= event.timeStart ? addDaysToDate(event.date, 1) : event.date;
  const end = zonedToUtc(endDate, event.timeEnd, event.timeZone);
  return { start, end };
}
```

## Checklist

- [ ] Every price input goes through `parseEventInput` (it calls
      `assertChargeableAmount` and `normalizeCurrency`).
- [ ] No code outside the gateway multiplies or divides amounts for Stripe.
- [ ] The currency picker lists `offeredCurrencies(event)`, not a global list.
- [ ] Display uses `formatMoney(amount, currency, locale)`.
- [ ] No `...Z` string concatenation with a local date anywhere in the host.
