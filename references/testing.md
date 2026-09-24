# Testing

Three suites, 47 tests, runnable with Vitest once the templates are in place.
They are written against the template files and paths used throughout this
skill (`@/lib/events/*`).

| Suite | Tests | Proves |
|---|---|---|
| `test/core.test.ts` | 22 | Currency conversion (0/2/3 decimals, ISK/UGX legacy), pricing never free by currency switch, DST-correct time, the hold schedule |
| `test/machine.test.ts` | 8 | Same-object no-ops, purity, stale-intent guard, no seat resurrection, renewal effects |
| `test/engine.test.ts` ([testing-lifecycles.md](testing-lifecycles.md)) | 17 | Whole lifecycles against an in-memory store and a fake gateway, including three regression tests for defects found in the earlier implementation |

Scenario tests move an injected clock (`now: () => new Date(clock.t)`) instead
of sleeping, so a three-week deposit lifecycle runs in milliseconds.

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import { fileURLToPath } from 'node:url';

export default defineConfig({
  resolve: { alias: { '@': fileURLToPath(new URL('.', import.meta.url)) } },
  test: { environment: 'node', include: ['test/**/*.test.ts'] },
});
```

Hosts with `TZ` pinned in their test script (for example `TZ=UTC`) need no
change: every time function takes the zone explicitly.

## Fixtures: in-memory store and fake gateway

The in-memory store follows the `EventStore` contract exactly: `mutate`
throwing writes nothing, same-object returns write nothing, seat deltas apply
with the write. Use it for host-side tests too.

```ts
// test/fixtures.ts: in-memory EventStore + fake PaymentGateway for engine scenario tests.
import type Stripe from 'stripe';
import { seatDelta } from '@/lib/events/participant-machine';
import { RegistrationError, type EventStore, type Notifier } from '@/lib/events/ports';
import type { OffSessionHoldResult, PaymentGateway } from '@/lib/events/stripe-gateway';
import type { EventRecord, Participant } from '@/lib/events/types';

export function memoryStore(): EventStore & { events: Map<string, EventRecord>; parts: Map<string, Participant> } {
  const events = new Map<string, EventRecord>();
  const parts = new Map<string, Participant>();
  const claims = new Map<string, 'claimed' | 'done'>();
  let seq = 0;
  const clone = <T>(v: T): T => structuredClone(v);
  return {
    events, parts,
    newParticipantId: () => `p${++seq}`,
    async getEvent(id) { const e = events.get(id); return e ? clone(e) : null; },
    async createEvent(e) { events.set(e.id, clone(e)); },
    async updateEvent(id, input, now) {
      const e = events.get(id);
      if (!e) return 'not_found';
      if (input.capacity < e.seatsTaken) return 'below_seats_taken';
      events.set(id, { ...e, ...input, updatedAt: now });
      return 'ok';
    },
    async setEventStatus(id, status) { const e = events.get(id); if (e) e.status = status; },
    async getParticipant(_e, id) { const p = parts.get(id); return p ? clone(p) : null; },
    async listParticipants(eventId) { return [...parts.values()].filter((p) => p.eventId === eventId).map(clone); },
    async listDue(now, limit) {
      return [...parts.values()]
        .filter((p) => p.nextActionAt && p.nextActionAt <= now)
        .sort((a, b) => a.nextActionAt!.getTime() - b.nextActionAt!.getTime())
        .slice(0, limit).map(clone);
    },
    async insertParticipant(p) {
      const e = events.get(p.eventId);
      if (!e || e.deletedAt || e.status !== 'published') throw new RegistrationError('registration_closed', 409);
      if (e.seatsTaken + p.ticketCount > e.capacity) throw new RegistrationError('no_spots', 409);
      e.seatsTaken += p.ticketCount;
      parts.set(p.id, clone(p));
    },
    async mutateParticipant(eventId, id, mutate) {
      const e = events.get(eventId);
      const stored = parts.get(id);
      if (!e || !stored) return null;
      const before = clone(stored);
      const after = mutate(before, clone(e)); // throws, so nothing is written, like a rolled-back tx
      if (after !== before) {
        e.seatsTaken += seatDelta(before, after);
        parts.set(id, clone(after));
      }
      return { before, after, event: clone(e) };
    },
    async claimWebhookEvent(id) { if (claims.has(id)) return false; claims.set(id, 'claimed'); return true; },
    async completeWebhookEvent(id) { claims.set(id, 'done'); },
    async releaseWebhookEvent(id) { claims.delete(id); },
  };
}

export interface FakeIntent { id: string; status: 'requires_capture' | 'canceled' | 'succeeded' | 'requires_action'; amount: number; received: number }

export function fakeGateway() {
  const calls: string[] = [];
  const intents = new Map<string, FakeIntent>();
  let n = 0;
  let nextHold: OffSessionHoldResult['status'] = 'held';
  let failCheckout = false;
  const gw: PaymentGateway = {
    async createCustomer(_e, _n, pid) { calls.push(`customer:${pid}`); return `cus_${pid}`; },
    async createTicketCheckout(a) {
      if (failCheckout) throw new Error('stripe down');
      calls.push(`ticket:${a.ref.participantId}:${a.amount}${a.currency}`);
      return { sessionId: `cs_${++n}`, url: `https://pay/${n}`, expiresAt: new Date(a.now.getTime() + 30 * 60_000) };
    },
    async createHoldCheckout(a) {
      calls.push(`holdCheckout:${a.ref.participantId}:${a.amount}${a.currency}`);
      return { sessionId: `cs_${++n}`, url: `https://pay/${n}`, expiresAt: new Date(a.now.getTime() + 30 * 60_000) };
    },
    async createSetupCheckout(a) {
      calls.push(`setupCheckout:${a.ref.participantId}`);
      return { sessionId: `cs_${++n}`, url: `https://pay/${n}`, expiresAt: new Date(a.now.getTime() + 30 * 60_000) };
    },
    async placeOffSessionHold(a) {
      calls.push(`offSessionHold:${a.idempotencyKey}`);
      const id = `pi_${++n}`;
      if (nextHold === 'held') {
        intents.set(id, { id, status: 'requires_capture', amount: a.amount, received: 0 });
        return { status: 'held', paymentIntentId: id, captureBefore: new Date(Date.parse('2030-01-01')), paymentMethodId: a.paymentMethodId };
      }
      if (nextHold === 'requires_action') return { status: 'requires_action', paymentIntentId: id, error: 'authentication_required' };
      return { status: 'declined', error: 'insufficient_funds' };
    },
    async retrievePaymentIntent(id) {
      const i = intents.get(id);
      const captureBefore = Math.floor(Date.parse('2030-01-01') / 1000);
      return {
        id, status: i?.status ?? 'canceled', metadata: {}, payment_method: 'pm_1', created: 0,
        latest_charge: { payment_method_details: { card: { capture_before: captureBefore } } },
      } as unknown as Stripe.PaymentIntent;
    },
    async cancelIntent(id) {
      calls.push(`cancel:${id}`);
      const i = intents.get(id);
      if (i?.status === 'succeeded') return 'already_captured';
      if (i) i.status = 'canceled';
      return 'canceled';
    },
    async captureIntent(id, amount) {
      calls.push(`capture:${id}:${amount}`);
      const i = intents.get(id);
      if (!i || i.status === 'canceled') return 'expired';
      i.status = 'succeeded';
      i.received = amount;
      return amount;
    },
    async refund(pi, amount, currency, pid) { calls.push(`refund:${pi}:${amount}${currency}:${pid}`); return 'succeeded'; },
    async expireCheckout(id) { calls.push(`expire:${id}`); },
    async retrieveCheckout(id) { return { id, status: 'expired', metadata: {} } as unknown as Stripe.Checkout.Session; },
    async retrieveSetupIntent(id) { return { id, status: 'succeeded', payment_method: 'pm_saved' } as unknown as Stripe.SetupIntent; },
  };
  return {
    gw, calls, intents,
    /** Simulate the customer authorizing a hold in Checkout. */
    authorize(amount: number): string {
      const id = `pi_${++n}`;
      intents.set(id, { id, status: 'requires_capture', amount, received: 0 });
      return id;
    },
    setNextHold(s: OffSessionHoldResult['status']) { nextHold = s; },
    setFailCheckout(v: boolean) { failCheckout = v; },
  };
}

export function recordingNotifier(): Notifier & { sent: string[] } {
  const sent: string[] = [];
  return { sent, async send(kind, p) { sent.push(`${kind}:${p.id}`); } };
}

export function makeEvent(over: Partial<EventRecord> = {}): EventRecord {
  const t = new Date('2026-01-01T00:00:00Z');
  return {
    id: 'ev1', locationId: 'loc1', typeId: null, name: 'Open range night', description: '', imageUrl: null,
    date: '2026-06-10', timeStart: '18:00', timeEnd: '21:00', timeZone: 'UTC',
    capacity: 10, seatsTaken: 0, ticketPrice: {}, deposit: { amountPerTicket: { usd: 2000 } },
    cancelDeadlineHours: 24, status: 'published', deletedAt: null, createdAt: t, updatedAt: t,
    ...over,
  };
}
```

## Pure core

```ts
// test/core.test.ts: the pure core (currency, pricing, time, schedule).
import { describe, expect, it } from 'vitest';
import {
  formatMoney, fromStripeAmount, minorUnitExponent, normalizeCurrency, toStripeAmount,
} from '@/lib/events/currency';
import {
  chooseDepositStrategy, holdCutoffAt, holdPlacementDueAt, settleDueAt, DEFAULT_DEPOSIT_POLICY,
} from '@/lib/events/deposit-schedule';
import { resolveCharge, isFreeEvent } from '@/lib/events/pricing';
import { eventWindowUtc, todayIn, zonedToUtc } from '@/lib/events/time';

const H = 3_600_000;
const D = 24 * H;

describe('currency', () => {
  it('normalizes codes and rejects junk', () => {
    expect(normalizeCurrency(' USD ')).toBe('usd');
    expect(normalizeCurrency('us')).toBeNull();
    expect(normalizeCurrency('US$')).toBeNull();
    expect(normalizeCurrency(840)).toBeNull();
  });

  it('knows 0-, 2- and 3-decimal currencies', () => {
    expect(minorUnitExponent('usd')).toBe(2);
    expect(minorUnitExponent('jpy')).toBe(0);
    expect(minorUnitExponent('isk')).toBe(0);
    expect(minorUnitExponent('kwd')).toBe(3);
  });

  it('converts to Stripe units, including the ISK/UGX x100 legacy', () => {
    expect(toStripeAmount(1999, 'usd')).toBe(1999);
    expect(toStripeAmount(500, 'jpy')).toBe(500);
    expect(toStripeAmount(500, 'isk')).toBe(50000);
    expect(toStripeAmount(500, 'ugx')).toBe(50000);
    expect(toStripeAmount(1250, 'kwd')).toBe(1250);
  });

  it('round-trips and rejects fractional krona from Stripe', () => {
    for (const [a, c] of [[1999, 'usd'], [500, 'jpy'], [500, 'isk'], [1250, 'kwd']] as const) {
      expect(fromStripeAmount(toStripeAmount(a, c), c)).toBe(a);
    }
    expect(() => fromStripeAmount(50050, 'isk')).toThrow(RangeError);
  });

  it('refuses 3-decimal amounts Stripe cannot charge', () => {
    expect(() => toStripeAmount(1255, 'kwd')).toThrow(/end in 0/);
    expect(() => toStripeAmount(-1, 'usd')).toThrow(RangeError);
    expect(() => toStripeAmount(10.5, 'usd')).toThrow(RangeError);
  });

  it('formats with the right number of decimals', () => {
    expect(formatMoney(1999, 'usd', 'en-US')).toBe('$19.99');
    expect(formatMoney(500, 'jpy', 'en-US')).toBe('\u00a5500');
    expect(formatMoney(1250, 'kwd', 'en-US')).toMatch(/1\.250/);
  });
});

describe('resolveCharge', () => {
  const paid = { ticketPrice: { usd: 2500, eur: 2300 }, deposit: null };
  const free = { ticketPrice: {}, deposit: null };
  const freeWithDeposit = { ticketPrice: { usd: 0 }, deposit: { amountPerTicket: { usd: 2000, jpy: 3000 } } };

  it('defaults to USD', () => {
    expect(resolveCharge(paid, undefined, 2)).toEqual({ ok: true, kind: 'paid', currency: 'usd', amount: 5000, unitPrice: 2500 });
  });

  it('falls back to the first offered currency when the default is not offered', () => {
    const r = resolveCharge({ ticketPrice: { eur: 900 }, deposit: null }, undefined, 1);
    expect(r).toMatchObject({ ok: true, currency: 'eur', amount: 900 });
  });

  it('REGRESSION: a currency the paid event is not priced in is refused, never free', () => {
    expect(resolveCharge(paid, 'gbp', 10)).toEqual({ ok: false, error: 'currency_not_offered' });
    const zeroInOne = { ticketPrice: { usd: 2500, pln: 0 }, deposit: null };
    expect(isFreeEvent(zeroInOne)).toBe(false);
    expect(resolveCharge(zeroInOne, 'pln', 10)).toEqual({ ok: false, error: 'currency_not_offered' });
  });

  it('honours a host currency restriction', () => {
    expect(resolveCharge(paid, 'eur', 1, { allowed: ['usd'], defaultCurrency: 'usd' }))
      .toEqual({ ok: false, error: 'currency_not_enabled' });
  });

  it('rejects malformed codes', () => {
    expect(resolveCharge(paid, 'dollars', 1)).toEqual({ ok: false, error: 'currency_invalid' });
  });

  it('free events cost nothing in any currency', () => {
    expect(resolveCharge(free, 'xyz', 3)).toEqual({ ok: true, kind: 'free', currency: null, amount: 0 });
  });

  it('free events with a deposit hold per ticket in an offered currency', () => {
    expect(resolveCharge(freeWithDeposit, 'jpy', 3))
      .toEqual({ ok: true, kind: 'deposit', currency: 'jpy', amount: 0, depositPerTicket: 3000, depositTotal: 9000 });
    expect(resolveCharge(freeWithDeposit, 'eur', 1)).toEqual({ ok: false, error: 'currency_not_offered' });
  });
});

describe('time', () => {
  it('converts local wall-clock to UTC across DST', () => {
    // New York: EST (UTC-5) in January, EDT (UTC-4) in July.
    expect(zonedToUtc('2026-01-15', '19:00', 'America/New_York').toISOString()).toBe('2026-01-16T00:00:00.000Z');
    expect(zonedToUtc('2026-07-15', '19:00', 'America/New_York').toISOString()).toBe('2026-07-15T23:00:00.000Z');
    // Warsaw on the spring-forward day: 10:00 is already CEST (UTC+2).
    expect(zonedToUtc('2026-03-29', '10:00', 'Europe/Warsaw').toISOString()).toBe('2026-03-29T08:00:00.000Z');
  });

  it('resolves a wall time skipped by DST forward', () => {
    // 02:30 does not exist in Warsaw on 2026-03-29; it lands at 03:30 CEST = 01:30Z.
    expect(zonedToUtc('2026-03-29', '02:30', 'Europe/Warsaw').toISOString()).toBe('2026-03-29T01:30:00.000Z');
  });

  it('resolves a repeated autumn wall time to the later (standard-time) occurrence', () => {
    // Warsaw falls back from 03:00 CEST to 02:00 CET on 2026-10-25; 02:30 CET = 01:30Z.
    expect(zonedToUtc('2026-10-25', '02:30', 'Europe/Warsaw').toISOString()).toBe('2026-10-25T01:30:00.000Z');
  });

  it('handles events that run past midnight', () => {
    const w = eventWindowUtc({ date: '2026-06-01', timeStart: '22:00', timeEnd: '01:00', timeZone: 'UTC' });
    expect(w.end.getTime() - w.start.getTime()).toBe(3 * H);
  });

  it('computes today in the venue zone, not UTC', () => {
    const lateEvening = new Date('2026-06-01T23:30:00Z');
    expect(todayIn('UTC', lateEvening)).toBe('2026-06-01');
    expect(todayIn('Europe/Warsaw', lateEvening)).toBe('2026-06-02');
    expect(todayIn('America/Los_Angeles', lateEvening)).toBe('2026-06-01');
  });
});

describe('deposit schedule', () => {
  const window = { start: new Date('2026-06-10T18:00:00Z'), end: new Date('2026-06-10T21:00:00Z') };
  const settle = window.end.getTime() + DEFAULT_DEPOSIT_POLICY.settleGraceMs; // 23:00Z

  it('holds in Checkout when settlement is at most 6 days away, else saves the card', () => {
    expect(chooseDepositStrategy(window, new Date(settle - 6 * D))).toBe('hold_now');
    expect(chooseDepositStrategy(window, new Date(settle - 6 * D - 1))).toBe('save_card');
  });

  it('places the off-session hold inside the shortest (Visa MIT) window', () => {
    const due = holdPlacementDueAt(window);
    expect(settle - due.getTime()).toBe(4 * D);
    expect(due.getTime()).toBeLessThanOrEqual(holdCutoffAt(window).getTime());
  });

  it('never schedules placement after the cutoff for long events', () => {
    const long = { start: new Date('2026-06-10T08:00:00Z'), end: new Date('2026-06-14T18:00:00Z') };
    expect(holdPlacementDueAt(long).getTime()).toBe(holdCutoffAt(long).getTime());
  });

  it('settles after the grace period, clamped before capture_before', () => {
    expect(settleDueAt(window, null)).toEqual({ at: new Date(settle), clamped: false });
    expect(settleDueAt(window, new Date(settle + 7 * H))).toEqual({ at: new Date(settle), clamped: false });
    const tight = new Date(settle + 2 * H);
    expect(settleDueAt(window, tight)).toEqual({ at: new Date(tight.getTime() - 6 * H), clamped: true });
  });
});
```

## State machine

```ts
// test/machine.test.ts: transition table guarantees the engine relies on.
import { describe, expect, it } from 'vitest';
import { effectsOf } from '@/lib/events/effects';
import { applyAction, LatePaymentError, seatDelta, TransitionError } from '@/lib/events/participant-machine';
import type { Participant } from '@/lib/events/types';

const T = new Date('2026-06-01T00:00:00Z');

function held(over: Partial<Participant['deposit']> = {}): Participant {
  return {
    id: 'p1', eventId: 'ev1', locationId: 'loc1', customerId: null, fullName: 'A', email: 'a@x.io', phone: null,
    locale: 'en', ticketCount: 2, currency: 'usd', checkoutAttempt: 1,
    payment: { status: 'CONFIRMED', method: 'FREE', amount: 0, checkoutSessionId: 'cs_1', paymentIntentId: null, expiresAt: null, refundedAmount: 0 },
    deposit: {
      status: 'HELD', strategy: 'hold_now', amountPerTicket: 2000, amount: 4000, stripeCustomerId: 'cus_1',
      paymentMethodId: 'pm_1', checkoutSessionId: 'cs_1', paymentIntentId: 'pi_live', captureBefore: T,
      holdDueAt: null, holdCutoffAt: T, settleDueAt: null, captureAmount: 0, waivedBy: null, waiveReason: null, lastError: null,
      ...over,
    },
    attendance: { status: 'UNKNOWN', arrivedCount: 0, checkedInAt: null, checkedInBy: null },
    nextAction: null, nextActionAt: null, registeredAt: T, cancelledAt: null, cancelledBy: null, updatedAt: T,
  };
}

describe('participant machine', () => {
  it('returns the same object for a repeated action (the engine\'s "nothing to do")', () => {
    const p = held();
    expect(applyAction(p, { type: 'HOLD_AUTHORIZED', paymentIntentId: 'pi_live', captureBefore: T, paymentMethodId: null }, T)).toBe(p);
    const settled = applyAction(p, { type: 'SETTLE' }, T);
    expect(applyAction(settled, { type: 'SETTLE' }, T)).toBe(settled);
  });

  it('does not mutate its input', () => {
    const p = held();
    const snapshot = structuredClone(p);
    applyAction(p, { type: 'CHECK_IN', count: 2, by: 's' }, T);
    expect(p).toEqual(snapshot);
  });

  it('ignores confirmations for a superseded intent', () => {
    const p = held({ status: 'RELEASING' });
    expect(applyAction(p, { type: 'HOLD_RELEASED', paymentIntentId: 'pi_old' }, T)).toBe(p);
    expect(applyAction(p, { type: 'HOLD_RELEASED', paymentIntentId: 'pi_live' }, T).deposit.status).toBe('RELEASED');
  });

  it('refuses to capture after a full arrival and to release after capture', () => {
    const arrived = applyAction(held(), { type: 'CHECK_IN', count: 2, by: 's' }, T);
    expect(arrived.deposit.status).toBe('RELEASING');
    expect(applyAction(arrived, { type: 'SETTLE' }, T)).toBe(arrived);
    const captured = held({ status: 'CAPTURED', captureAmount: 4000 });
    expect(() => applyAction(captured, { type: 'WAIVE', by: 's', reason: 'r' }, T)).toThrow(TransitionError);
  });

  it('rejects impossible arrival counts', () => {
    expect(() => applyAction(held(), { type: 'CHECK_IN', count: 3, by: 's' }, T)).toThrow(/1\.\.2/);
    expect(() => applyAction(held(), { type: 'CHECK_IN', count: 0, by: 's' }, T)).toThrow(TransitionError);
  });

  it('money for a released seat raises LatePaymentError, never resurrects the seat', () => {
    const gone: Participant = { ...held({ status: 'AWAITING_CARD', paymentIntentId: null }), payment: { ...held().payment, status: 'EXPIRED' } };
    expect(() => applyAction(gone, { type: 'HOLD_AUTHORIZED', paymentIntentId: 'pi_x', captureBefore: T, paymentMethodId: null }, T))
      .toThrow(LatePaymentError);
  });

  it('counts seats only while PENDING or CONFIRMED', () => {
    const p = held();
    const cancelled = applyAction(p, { type: 'CANCEL', actor: 'customer', late: false, refund: 'full' }, T);
    expect(seatDelta(p, cancelled)).toBe(-2);
    expect(seatDelta(null, p)).toBe(2);
  });

  it('renewal cancels the old intent and keeps the new one', () => {
    const p = held();
    const renewed = applyAction(p, { type: 'HOLD_AUTHORIZED', paymentIntentId: 'pi_new', captureBefore: T, paymentMethodId: 'pm_1' }, T);
    expect(renewed.deposit.paymentIntentId).toBe('pi_new');
    expect(effectsOf(p, renewed)).toContainEqual({ type: 'cancel_intent', paymentIntentId: 'pi_live' });
  });
});
```

## Lifecycles

The 17 scenario tests are in [testing-lifecycles.md](testing-lifecycles.md).
