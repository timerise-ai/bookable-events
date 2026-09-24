# Testing: lifecycles

End-to-end scenarios over the engine, the webhook handler and the tick, using
the in-memory store and fake gateway from [testing.md](testing.md). Each test
reads as a customer story: register, pay or hold, arrive or not, and the money
outcome.

| Group | Covers |
|---|---|
| Hold placed in Checkout | release on full arrival, capture after grace (not before), partial capture, waiver, early vs late cancel, venue cancel, idempotent check-in |
| Card saved, hold placed later | placement exactly at `holdDueAt` inside the Visa MIT window, decline, then email, then seat released at cutoff |
| Paid tickets | unpriced currency refused (**regression**), declined attempt keeps the seat (**regression**), refund of the frozen amount, late money refunded (**regression**), Stripe outage releases the seat, capacity, duplicate webhook |
| Manage token | per-participant HMAC verification |

```ts
// test/engine.test.ts: end-to-end lifecycles against the in-memory store and fake gateway.
import type Stripe from 'stripe';
import { beforeEach, describe, expect, it } from 'vitest';
import { createEventsEngine } from '@/lib/events/engine';
import { RegistrationError } from '@/lib/events/ports';
import { createTick } from '@/lib/events/tick';
import { createWebhookHandler } from '@/lib/events/webhook';
import { fakeGateway, makeEvent, memoryStore, recordingNotifier } from './fixtures';

const H = 3_600_000;
const D = 24 * H;
const START = Date.parse('2026-06-10T18:00:00Z');
const END = Date.parse('2026-06-10T21:00:00Z');

let evtSeq = 0;
function stripeEvent(type: string, object: Record<string, unknown>): Stripe.Event {
  return { id: `evt_${++evtSeq}`, type, data: { object } } as unknown as Stripe.Event;
}

function setup(eventOver: Parameters<typeof makeEvent>[0] = {}, startAt = START - 2 * D) {
  const store = memoryStore();
  const fake = fakeGateway();
  const notifier = recordingNotifier();
  const clock = { t: startAt };
  const engine = createEventsEngine({
    store, gateway: fake.gw, notifier, ledger: null, manageTokenSecret: 'test-secret',
    now: () => new Date(clock.t),
    urls: { success: () => 'https://x/ok', cancel: () => 'https://x/no', manage: (_e, p, t) => `https://x/m/${p}?t=${t}` },
  });
  const webhook = createWebhookHandler(engine);
  const tick = createTick(engine, webhook);
  void store.createEvent(makeEvent(eventOver));
  const part = (id: string) => store.parts.get(id)!;
  const seats = () => store.events.get('ev1')!.seatsTaken;
  return { store, fake, notifier, clock, engine, webhook, tick, part, seats };
}

const guest = { eventId: 'ev1', customerId: null, fullName: 'Ada Lovelace', email: 'ada@example.com', phone: null, locale: 'en', currency: 'usd', method: 'STRIPE' as const };

async function registerAndHold(s: ReturnType<typeof setup>, tickets: number) {
  const r = await s.engine.register({ ...guest, ticketCount: tickets });
  const pi = s.fake.authorize(2000 * tickets);
  await s.webhook.handle(stripeEvent('checkout.session.completed', {
    id: s.part(r.participantId).deposit.checkoutSessionId, payment_intent: pi, payment_status: 'unpaid',
    metadata: { kind: 'event_deposit_hold', eventId: 'ev1', participantId: r.participantId },
  }));
  return { id: r.participantId, pi };
}

describe('deposit: hold placed in Checkout (event at most 6 days away)', () => {
  let s: ReturnType<typeof setup>;
  beforeEach(() => { s = setup(); });

  it('holds, then releases the whole hold when everyone arrives', async () => {
    const { id, pi } = await registerAndHold(s, 2);
    expect(s.fake.calls).toContain(`holdCheckout:${id}:4000usd`);
    expect(s.part(id).payment.status).toBe('CONFIRMED');
    expect(s.part(id).deposit.status).toBe('HELD');
    expect(s.seats()).toBe(2);

    s.clock.t = START + 10 * 60_000;
    await s.engine.staff.checkIn('ev1', id, 2, 'staff1');
    expect(s.fake.calls).toContain(`cancel:${pi}`);
    expect(s.part(id).deposit.status).toBe('RELEASED');
    expect(s.notifier.sent).toEqual(expect.arrayContaining([`deposit_held:${id}`, `deposit_released:${id}`]));
  });

  it('captures a no-show after the grace period, and not before', async () => {
    const { id, pi } = await registerAndHold(s, 2);
    s.clock.t = END + 1 * H;
    await s.tick();
    expect(s.part(id).deposit.status).toBe('HELD');

    s.clock.t = END + 2 * H;
    await s.tick();
    expect(s.fake.calls).toContain(`capture:${pi}:4000`);
    expect(s.part(id).deposit.status).toBe('CAPTURED');
    expect(s.part(id).attendance.status).toBe('NO_SHOW');
    expect(s.notifier.sent).toContain(`deposit_captured:${id}`);
  });

  it('partial arrival captures only the missing tickets', async () => {
    const { id, pi } = await registerAndHold(s, 3);
    s.clock.t = START;
    await s.engine.staff.checkIn('ev1', id, 1, 'staff1');
    expect(s.part(id).deposit.status).toBe('HELD');
    s.clock.t = END + 2 * H;
    await s.tick();
    expect(s.fake.calls).toContain(`capture:${pi}:4000`);
    expect(s.part(id).deposit.captureAmount).toBe(4000);
  });

  it('a waiver releases the hold and records who and why', async () => {
    const { id, pi } = await registerAndHold(s, 1);
    await s.engine.staff.waive('ev1', id, 'mgr1', 'flat tyre, called ahead');
    expect(s.fake.calls).toContain(`cancel:${pi}`);
    expect(s.part(id).deposit).toMatchObject({ status: 'RELEASED', waivedBy: 'mgr1', waiveReason: 'flat tyre, called ahead' });
  });

  it('early cancel releases; late cancel counts as a no-show', async () => {
    const early = await registerAndHold(s, 1);
    await s.engine.cancelByCustomer('ev1', early.id);
    expect(s.part(early.id).deposit.status).toBe('RELEASED');
    expect(s.seats()).toBe(0);

    const late = await registerAndHold(s, 1);
    s.clock.t = START - 2 * H; // inside the 24 h deadline
    await s.engine.cancelByCustomer('ev1', late.id);
    expect(s.fake.calls).toContain(`capture:${late.pi}:2000`);
    expect(s.part(late.id).deposit.status).toBe('CAPTURED');
  });

  it('cancelling the event releases every hold', async () => {
    const a = await registerAndHold(s, 1);
    const b = await registerAndHold(s, 2);
    const r = await s.engine.staff.cancelEvent('ev1', 'admin');
    expect(r).toEqual({ cancelled: 2, failed: [] });
    expect(s.fake.calls).toEqual(expect.arrayContaining([`cancel:${a.pi}`, `cancel:${b.pi}`]));
    expect(s.seats()).toBe(0);
  });

  it('check-in is idempotent and cannot be followed by a cancel', async () => {
    const { id } = await registerAndHold(s, 1);
    await s.engine.staff.checkIn('ev1', id, 1, 'staff1');
    const again = await s.engine.staff.checkIn('ev1', id, 1, 'staff1');
    expect(again!.before).toBe(again!.after);
    await expect(s.engine.cancelByCustomer('ev1', id)).rejects.toMatchObject({ code: 'already_arrived' });
  });
});

describe('deposit: card saved, hold placed later (event > 6 days away)', () => {
  it('saves the card, then the tick places the hold inside the Visa MIT window', async () => {
    const s = setup({}, START - 20 * D);
    const r = await s.engine.register({ ...guest, ticketCount: 1 });
    expect(s.fake.calls).toContain(`setupCheckout:${r.participantId}`);
    await s.webhook.handle(stripeEvent('checkout.session.completed', {
      id: 'cs_x', setup_intent: 'seti_1', metadata: { kind: 'event_deposit_setup', eventId: 'ev1', participantId: r.participantId },
    }));
    const p = s.part(r.participantId);
    expect(p.deposit.status).toBe('CARD_SAVED');
    expect(p.nextAction).toBe('PLACE_HOLD');
    expect(END + 2 * H - p.nextActionAt!.getTime()).toBe(4 * D);

    s.clock.t = p.nextActionAt!.getTime() - 1;
    await s.tick();
    expect(s.part(r.participantId).deposit.status).toBe('CARD_SAVED');

    s.clock.t = p.nextActionAt!.getTime();
    await s.tick();
    expect(s.fake.calls).toContain(`offSessionHold:hold:${r.participantId}`);
    expect(s.part(r.participantId).deposit.status).toBe('HELD');
  });

  it('a declined off-session hold emails the customer, then frees the seat at the cutoff', async () => {
    const s = setup({}, START - 20 * D);
    s.fake.setNextHold('declined');
    const r = await s.engine.register({ ...guest, ticketCount: 2 });
    await s.webhook.handle(stripeEvent('checkout.session.completed', {
      id: 'cs_x', setup_intent: 'seti_1', metadata: { kind: 'event_deposit_setup', eventId: 'ev1', participantId: r.participantId },
    }));
    s.clock.t = s.part(r.participantId).nextActionAt!.getTime();
    await s.tick();
    expect(s.part(r.participantId).deposit.status).toBe('HOLD_FAILED');
    expect(s.notifier.sent).toContain(`hold_failed:${r.participantId}`);
    expect(s.seats()).toBe(2);

    s.clock.t = START - 24 * H;
    await s.tick();
    expect(s.part(r.participantId).payment.status).toBe('CANCELLED');
    expect(s.part(r.participantId).cancelledBy).toBe('system');
    expect(s.seats()).toBe(0);
    expect(s.notifier.sent).toContain(`seat_released_no_hold:${r.participantId}`);
  });
});

describe('paid tickets', () => {
  const paidEvent = { ticketPrice: { usd: 2500, eur: 2300 }, deposit: null };

  it('REGRESSION: an unpriced currency is refused, not treated as free', async () => {
    const s = setup(paidEvent);
    await expect(s.engine.register({ ...guest, currency: 'gbp', ticketCount: 10 }))
      .rejects.toMatchObject({ code: 'currency_not_offered' });
    expect(s.seats()).toBe(0);
  });

  it('REGRESSION: a declined card inside open Checkout does not release the seat', async () => {
    const s = setup(paidEvent);
    const r = await s.engine.register({ ...guest, ticketCount: 1 });
    await s.webhook.handle(stripeEvent('payment_intent.payment_failed', {
      id: 'pi_declined', metadata: { kind: 'event_ticket', eventId: 'ev1', participantId: r.participantId },
    }));
    expect(s.part(r.participantId).payment.status).toBe('PENDING');
    expect(s.seats()).toBe(1);
  });

  it('confirms on payment, and refunds the frozen amount on an early cancel', async () => {
    const s = setup(paidEvent);
    const r = await s.engine.register({ ...guest, ticketCount: 2 });
    const cs = s.part(r.participantId).payment.checkoutSessionId;
    await s.webhook.handle(stripeEvent('checkout.session.completed', {
      id: cs, payment_intent: 'pi_t', payment_status: 'paid', amount_total: 5000, currency: 'usd',
      metadata: { kind: 'event_ticket', eventId: 'ev1', participantId: r.participantId },
    }));
    expect(s.part(r.participantId).payment.status).toBe('CONFIRMED');

    s.store.events.get('ev1')!.ticketPrice = { usd: 9900 }; // price change must not affect the refund
    await s.engine.cancelByCustomer('ev1', r.participantId);
    expect(s.fake.calls).toContain(`refund:pi_t:5000usd:${r.participantId}`);
    expect(s.part(r.participantId).payment.status).toBe('REFUNDED');
    expect(s.seats()).toBe(0);
  });

  it('REGRESSION: money arriving after the seat expired is refunded, not lost', async () => {
    const s = setup(paidEvent);
    const r = await s.engine.register({ ...guest, ticketCount: 1 });
    s.clock.t += 36 * 60_000;
    await s.tick(); // session reported expired, so the seat is released
    expect(s.part(r.participantId).payment.status).toBe('EXPIRED');
    expect(s.seats()).toBe(0);

    await s.webhook.handle(stripeEvent('checkout.session.completed', {
      id: 'cs_late', payment_intent: 'pi_late', payment_status: 'paid', amount_total: 2500, currency: 'usd',
      metadata: { kind: 'event_ticket', eventId: 'ev1', participantId: r.participantId },
    }));
    expect(s.fake.calls).toContain(`refund:pi_late:2500usd:late:${r.participantId}`);
    expect(s.seats()).toBe(0);
  });

  it('a Stripe outage at checkout gives the seat back immediately', async () => {
    const s = setup(paidEvent);
    s.fake.setFailCheckout(true);
    await expect(s.engine.register({ ...guest, ticketCount: 3 })).rejects.toBeInstanceOf(RegistrationError);
    expect(s.seats()).toBe(0);
  });

  it('enforces capacity', async () => {
    const s = setup({ ...paidEvent, capacity: 3 });
    await s.engine.register({ ...guest, ticketCount: 3 });
    await expect(s.engine.register({ ...guest, ticketCount: 1 })).rejects.toMatchObject({ code: 'no_spots' });
  });

  it('ignores a redelivered webhook', async () => {
    const s = setup(paidEvent);
    const r = await s.engine.register({ ...guest, ticketCount: 1 });
    const evt = stripeEvent('checkout.session.completed', {
      id: 'cs', payment_intent: 'pi_t', payment_status: 'paid', amount_total: 2500, currency: 'usd',
      metadata: { kind: 'event_ticket', eventId: 'ev1', participantId: r.participantId },
    });
    expect(await s.webhook.handle(evt)).toBe('processed');
    expect(await s.webhook.handle(evt)).toBe('duplicate');
  });
});

describe('manage token', () => {
  it('verifies only the token for that participant', () => {
    const s = setup();
    const t = s.engine.manageToken('p1');
    expect(s.engine.verifyManageToken('p1', t)).toBe(true);
    expect(s.engine.verifyManageToken('p2', t)).toBe(false);
    expect(s.engine.verifyManageToken('p1', 'x')).toBe(false);
  });
});
```
