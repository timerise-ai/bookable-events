# Engine

`createEventsEngine(deps)` wires the pure core to the host's store, gateway,
ledger and notifier. It is the only module routes and the tick call.

## Dependencies

| Dep | Type | Required | Notes |
|---|---|---|---|
| `store` | `EventStore` | yes | [firestore.md](firestore.md) or [postgres.md](postgres.md) |
| `gateway` | `PaymentGateway` | yes | `new StripeGateway(new Stripe(key))` ([stripe.md](stripe.md)) |
| `notifier` | `Notifier` | yes | Host email/SMS; receives a kind + participant + manage link |
| `ledger` | `BalanceLedger \| null` | no | Stored-credit wallet; `null` disables balance payments |
| `urls` | `EngineUrls` | yes | success / cancel / manage URLs on the host's site |
| `manageTokenSecret` | string | yes | at least 32 random bytes; rotating it invalidates outstanding guest links |
| `now` | `() => Date` | no | Injected clock; tests move time instead of sleeping |
| `policy` | `DepositPolicy` | no | [no-show-deposits.md](no-show-deposits.md) |
| `currencyPolicy` | `CurrencyPolicy` | no | Default: any currency, `usd` default |
| `resolveName` | `(event, locale) => string` | no | Event name for Checkout line items |

## Surface

| Member | Used by |
|---|---|
| `register(input)` | register route |
| `resumeCheckout(eventId, participantId)` | customer resume route |
| `cancelByCustomer(eventId, participantId)` | customer cancel route |
| `staff.checkIn / markNoShow / waive / cancel / cancelEvent` | staff and admin routes |
| `transition(eventId, participantId, action)` | webhook, tick, everything above |
| `patch(eventId, participantId, fn)` | ids and counters that are not state (session id, attempt) |
| `runEffects(result, effects)` | tick retries |
| `manageToken(id)`, `verifyManageToken(id, token)` | email links, customer routes |

`transition` is the heart: mutate in a transaction, then reschedule, then commit, then
effects. Follow-up confirmations re-enter `transition`, which is why every
action must be a no-op when already applied.

```ts
// lib/events/engine.ts: orchestration. Transaction first, Stripe second, always.
import { createHmac, timingSafeEqual } from 'node:crypto';
import {
  DEFAULT_DEPOSIT_POLICY, chooseDepositStrategy, holdCutoffAt, holdPlacementDueAt, settleDueAt, type DepositPolicy,
} from './deposit-schedule';
import { effectsOf, scheduleNextAction, type Effect } from './effects';
import { applyAction, type ParticipantAction } from './participant-machine';
import {
  RegistrationError, type BalanceLedger, type EngineUrls, type EventStore, type MutationResult, type Notifier,
} from './ports';
import { ANY_CURRENCY, resolveCharge, type CurrencyPolicy } from './pricing';
import type { PaymentGateway } from './stripe-gateway';
import { eventWindowUtc } from './time';
import type { Actor, EventRecord, Participant } from './types';

export interface EngineDeps {
  store: EventStore;
  gateway: PaymentGateway;
  notifier: Notifier;
  ledger: BalanceLedger | null;
  urls: EngineUrls;
  manageTokenSecret: string;
  now?: () => Date;
  policy?: DepositPolicy;
  currencyPolicy?: CurrencyPolicy;
  resolveName?: (event: EventRecord, locale: string) => string;
}

export interface RegistrationInput {
  eventId: string;
  customerId: string | null;       // from the verified session, or null for a guest
  fullName: string;
  email: string;
  phone: string | null;
  locale: string;
  ticketCount: number;
  currency: unknown;               // raw client choice; resolveCharge validates it
  method: 'STRIPE' | 'BALANCE';
}

export interface RegistrationResult {
  participantId: string;
  manageToken: string;
  checkoutUrl: string | null;      // null = done, no redirect
}

export function createEventsEngine(deps: EngineDeps) {
  const now = deps.now ?? (() => new Date());
  const policy = deps.policy ?? DEFAULT_DEPOSIT_POLICY;
  const currencyPolicy = deps.currencyPolicy ?? ANY_CURRENCY;
  const nameOf = deps.resolveName ?? ((e: EventRecord) => (typeof e.name === 'string' ? e.name : Object.values(e.name)[0] ?? 'Event'));
  const log = (msg: string, err?: unknown) => console.error(`[events] ${msg}`, err ?? '');

  function manageToken(participantId: string): string {
    return createHmac('sha256', deps.manageTokenSecret).update(participantId).digest('base64url');
  }

  function verifyManageToken(participantId: string, token: string): boolean {
    const expected = Buffer.from(manageToken(participantId));
    const given = Buffer.from(token);
    return expected.length === given.length && timingSafeEqual(expected, given);
  }

  function reschedule(p: Participant, event: EventRecord, at: Date): Participant {
    const window = eventWindowUtc(event);
    const settle = p.deposit.status === 'HELD' ? settleDueAt(window, p.deposit.captureBefore, policy).at : null;
    const deposit = settle?.getTime() === p.deposit.settleDueAt?.getTime() ? p.deposit : { ...p.deposit, settleDueAt: settle };
    return { ...p, deposit, ...scheduleNextAction(p, window, at, policy) };
  }

  /** The only way a participant changes state. */
  async function transition(eventId: string, participantId: string, action: ParticipantAction): Promise<MutationResult | null> {
    const at = now();
    const res = await deps.store.mutateParticipant(eventId, participantId, (p, event) => {
      const next = applyAction(p, action, at);
      return next === p ? p : reschedule(next, event, at);
    });
    if (res && res.before !== res.after) await runEffects(res, effectsOf(res.before, res.after));
    return res;
  }

  /** Patch non-state fields (session ids, attempt counter) through the same transaction path. */
  async function patch(eventId: string, participantId: string, fn: (p: Participant) => Participant) {
    const at = now();
    return deps.store.mutateParticipant(eventId, participantId, (p, event) => reschedule({ ...fn(p), updatedAt: at }, event, at));
  }

  async function runEffects(res: MutationResult, effects: Effect[]): Promise<void> {
    const p = res.after;
    const currency = p.currency ?? currencyPolicy.defaultCurrency;
    for (const effect of effects) {
      try {
        switch (effect.type) {
          case 'expire_checkout':
            await deps.gateway.expireCheckout(effect.sessionId);
            break;
          case 'cancel_intent': {
            const r = await deps.gateway.cancelIntent(effect.paymentIntentId);
            if (r === 'canceled') await transition(p.eventId, p.id, { type: 'HOLD_RELEASED', paymentIntentId: effect.paymentIntentId });
            else log(`release of ${effect.paymentIntentId} found it already captured (participant ${p.id})`);
            break;
          }
          case 'capture_intent': {
            const r = await deps.gateway.captureIntent(effect.paymentIntentId, effect.amount, currency);
            await transition(p.eventId, p.id, r === 'expired'
              ? { type: 'HOLD_RELEASED', paymentIntentId: effect.paymentIntentId, reason: 'authorization_expired' }
              : { type: 'HOLD_CAPTURED', paymentIntentId: effect.paymentIntentId, amount: r });
            break;
          }
          case 'refund': {
            if (effect.method === 'BALANCE') {
              if (!deps.ledger || !p.customerId) throw new Error('balance refund without ledger/customer');
              await deps.ledger.credit(p.customerId, effect.amount, currency, `refund:${p.id}`);
              await transition(p.eventId, p.id, { type: 'REFUND_SUCCEEDED', amount: p.payment.amount });
            } else if (effect.paymentIntentId) {
              const r = await deps.gateway.refund(effect.paymentIntentId, effect.amount, currency, p.id);
              if (r === 'succeeded') await transition(p.eventId, p.id, { type: 'REFUND_SUCCEEDED', amount: p.payment.amount });
              // pending: charge.refunded completes it; the tick re-asks with the same idempotency key
            }
            break;
          }
          case 'notify':
            await deps.notifier.send(effect.kind, p, res.event, {
              manage: deps.urls.manage(p.eventId, p.id, manageToken(p.id)),
            });
            break;
        }
      } catch (err) {
        // The participant already carries RETRY_* / nextActionAt; the tick picks it up.
        log(`effect ${effect.type} failed for participant ${p.id}`, err);
      }
    }
  }

  // --- Registration ---------------------------------------------------------

  async function register(input: RegistrationInput): Promise<RegistrationResult> {
    const at = now();
    const event = await deps.store.getEvent(input.eventId);
    if (!event || event.deletedAt) throw new RegistrationError('event_not_found', 404);
    if (event.status !== 'published') throw new RegistrationError('registration_closed', 409);
    const window = eventWindowUtc(event);
    if (at >= window.start) throw new RegistrationError('event_started', 409);

    const charge = resolveCharge(event, input.currency, input.ticketCount, currencyPolicy);
    if (!charge.ok) throw new RegistrationError(charge.error, 400);
    const useBalance = charge.kind === 'paid' && input.method === 'BALANCE';
    if (useBalance && (!deps.ledger || !input.customerId)) throw new RegistrationError('login_required', 401);

    const id = deps.store.newParticipantId();
    const strategy = charge.kind === 'deposit' ? chooseDepositStrategy(window, at, policy) : null;
    const base: Participant = {
      id, eventId: event.id, locationId: event.locationId, customerId: input.customerId,
      fullName: input.fullName, email: input.email, phone: input.phone, locale: input.locale,
      ticketCount: input.ticketCount, currency: charge.currency, checkoutAttempt: 1,
      payment: {
        status: charge.kind === 'free' ? 'CONFIRMED' : 'PENDING',
        method: charge.kind === 'paid' ? (useBalance ? 'BALANCE' : 'STRIPE') : 'FREE',
        amount: charge.amount, checkoutSessionId: null, paymentIntentId: null,
        // Placeholder until the session exists; if the process dies first, the tick cleans up.
        expiresAt: charge.kind === 'free' ? null : new Date(at.getTime() + 5 * 60_000),
        refundedAmount: 0,
      },
      deposit: {
        status: charge.kind === 'deposit' ? 'AWAITING_CARD' : 'NONE', strategy,
        amountPerTicket: charge.kind === 'deposit' ? charge.depositPerTicket : 0,
        amount: charge.kind === 'deposit' ? charge.depositTotal : 0,
        stripeCustomerId: null, paymentMethodId: null, checkoutSessionId: null, paymentIntentId: null,
        captureBefore: null,
        holdDueAt: strategy === 'save_card' ? holdPlacementDueAt(window, policy) : null,
        holdCutoffAt: strategy ? holdCutoffAt(window, policy) : null,
        settleDueAt: null, captureAmount: 0, waivedBy: null, waiveReason: null, lastError: null,
      },
      attendance: { status: 'UNKNOWN', arrivedCount: 0, checkedInAt: null, checkedInBy: null },
      nextAction: null, nextActionAt: null,
      registeredAt: at, cancelledAt: null, cancelledBy: null, updatedAt: at,
    };
    const participant = reschedule(base, event, at);
    await deps.store.insertParticipant(participant); // capacity is enforced here, in the transaction
    const token = manageToken(id);

    if (charge.kind === 'free') {
      await runEffects({ before: participant, after: participant, event }, [{ type: 'notify', kind: 'registered' }]);
      return { participantId: id, manageToken: token, checkoutUrl: null };
    }

    if (useBalance && deps.ledger && input.customerId) {
      const r = await deps.ledger.debit(input.customerId, charge.amount, charge.currency, `event:${id}`);
      if (r === 'insufficient') {
        await transition(event.id, id, { type: 'CANCEL', actor: 'system', late: false, refund: 'none' });
        throw new RegistrationError('insufficient_balance', 402);
      }
      await transition(event.id, id, { type: 'BALANCE_DEBITED' });
      return { participantId: id, manageToken: token, checkoutUrl: null };
    }

    try {
      const url = await startCheckout(participant, event);
      return { participantId: id, manageToken: token, checkoutUrl: url };
    } catch (err) {
      // Seat was taken before Stripe was called: give it back now, not in 30 minutes.
      log(`checkout creation failed for ${id}`, err);
      await transition(event.id, id, { type: 'CANCEL', actor: 'system', late: false, refund: 'none' });
      throw new RegistrationError('payment_unavailable', 502);
    }
  }

  /** Creates the right Checkout Session for a PENDING participant and stores its id. */
  async function startCheckout(p: Participant, event: EventRecord): Promise<string> {
    const at = now();
    const ref = { eventId: event.id, participantId: p.id, attempt: p.checkoutAttempt };
    const urls = { successUrl: deps.urls.success(event.id, p.id), cancelUrl: deps.urls.cancel(event.id) };
    const name = nameOf(event, p.locale);
    const currency = p.currency ?? currencyPolicy.defaultCurrency;

    if (p.payment.method === 'STRIPE') {
      const s = await deps.gateway.createTicketCheckout({
        ref, email: p.email, productName: `${name} x${p.ticketCount}`, amount: p.payment.amount, currency, now: at, ...urls,
      });
      await patch(event.id, p.id, (cur) => ({ ...cur, payment: { ...cur.payment, checkoutSessionId: s.sessionId, expiresAt: s.expiresAt } }));
      return s.url;
    }

    const customerId = p.deposit.stripeCustomerId ?? (await deps.gateway.createCustomer(p.email, p.fullName, p.id));
    const s = p.deposit.strategy === 'save_card' && p.deposit.status === 'AWAITING_CARD'
      ? await deps.gateway.createSetupCheckout({ ref, customerId, now: at, ...urls })
      : await deps.gateway.createHoldCheckout({
          ref, customerId, productName: `${name}: deposit, released on arrival`,
          amount: p.deposit.amount, currency, now: at, ...urls,
        });
    await patch(event.id, p.id, (cur) => ({
      ...cur,
      payment: cur.payment.status === 'PENDING' ? { ...cur.payment, expiresAt: s.expiresAt } : cur.payment,
      deposit: { ...cur.deposit, stripeCustomerId: customerId, checkoutSessionId: s.sessionId },
    }));
    return s.url;
  }

  /** "Pay again" / "fix my card": new session for PENDING, HOLD_FAILED or HOLD_REQUIRES_ACTION. */
  async function resumeCheckout(eventId: string, participantId: string): Promise<string> {
    const res = await patch(eventId, participantId, (p) => {
      const resumable = p.payment.status === 'PENDING' ||
        (p.payment.status === 'CONFIRMED' && (p.deposit.status === 'HOLD_FAILED' || p.deposit.status === 'HOLD_REQUIRES_ACTION'));
      if (!resumable) throw new RegistrationError('registration_closed', 409);
      return { ...p, checkoutAttempt: p.checkoutAttempt + 1 };
    });
    if (!res) throw new RegistrationError('event_not_found', 404);
    const old = res.before.payment.checkoutSessionId ?? res.before.deposit.checkoutSessionId;
    if (old) await deps.gateway.expireCheckout(old).catch((e) => log('expire old session', e));
    return startCheckout(res.after, res.event);
  }

  // --- Customer and staff actions -------------------------------------------

  async function cancelByCustomer(eventId: string, participantId: string) {
    const event = await deps.store.getEvent(eventId);
    if (!event) throw new RegistrationError('event_not_found', 404);
    const { start } = eventWindowUtc(event);
    const at = now();
    if (at >= start) throw new RegistrationError('event_started', 409);
    const late = at.getTime() > start.getTime() - event.cancelDeadlineHours * 3_600_000;
    return transition(eventId, participantId, { type: 'CANCEL', actor: 'customer', late, refund: late ? 'none' : 'full' });
  }

  const staff = {
    checkIn: (eventId: string, participantId: string, count: number, by: string) =>
      transition(eventId, participantId, { type: 'CHECK_IN', count, by }),
    markNoShow: (eventId: string, participantId: string, by: string) =>
      transition(eventId, participantId, { type: 'MARK_NO_SHOW', by }),
    waive: (eventId: string, participantId: string, by: string, reason: string) =>
      transition(eventId, participantId, { type: 'WAIVE', by, reason }),
    cancel: (eventId: string, participantId: string, actor: Actor, refund: 'full' | 'none') =>
      transition(eventId, participantId, { type: 'CANCEL', actor, late: false, refund }),
    /** Venue cancels the whole event: full refunds, every hold released. Safe to re-run. */
    async cancelEvent(eventId: string, actor: Actor): Promise<{ cancelled: number; failed: string[] }> {
      const participants = await deps.store.listParticipants(eventId);
      let cancelled = 0;
      const failed: string[] = [];
      for (const p of participants) {
        try {
          const r = await transition(eventId, p.id, { type: 'CANCEL', actor, late: false, refund: 'full' });
          if (r && r.before !== r.after) cancelled++;
        } catch (err) {
          failed.push(p.id);
          log(`cancelEvent: participant ${p.id}`, err);
        }
      }
      return { cancelled, failed };
    },
  };

  return {
    register, resumeCheckout, cancelByCustomer, staff, transition, patch, runEffects,
    manageToken, verifyManageToken, now, policy, currencyPolicy, deps,
  };
}

export type EventsEngine = ReturnType<typeof createEventsEngine>;
```

## The contracts it depends on

`EventStore`, `BalanceLedger`, `Notifier` and `EngineUrls` are the host-facing
ports. `insertParticipant` and `mutateParticipant` are the two methods that
**must** be real transactions; everything else is a plain read or write.

```ts
// lib/events/ports.ts: what the host must implement. Everything else in lib/events is portable.
import type { NotificationKind } from './effects';
import type { EventInput } from './input';
import type { CurrencyCode, EventRecord, EventStatus, Participant } from './types';

export class RegistrationError extends Error {
  constructor(
    public readonly code:
      | 'event_not_found' | 'registration_closed' | 'event_started' | 'no_spots'
      | 'currency_invalid' | 'currency_not_offered' | 'currency_not_enabled'
      | 'login_required' | 'insufficient_balance' | 'payment_unavailable',
    public readonly status: number,
  ) {
    super(code);
    this.name = 'RegistrationError';
  }
}

export interface MutationResult {
  before: Participant;
  after: Participant;
  event: EventRecord;
}

export interface EventStore {
  newParticipantId(): string;
  getEvent(eventId: string): Promise<EventRecord | null>;
  createEvent(event: EventRecord): Promise<void>;
  /** ONE transaction: refuse a capacity below the current seatsTaken. */
  updateEvent(eventId: string, input: EventInput, now: Date): Promise<'ok' | 'not_found' | 'below_seats_taken'>;
  setEventStatus(eventId: string, status: EventStatus, now: Date): Promise<void>;
  getParticipant(eventId: string, participantId: string): Promise<Participant | null>;
  listParticipants(eventId: string): Promise<Participant[]>;
  /** Oldest-due first. The tick's only query. */
  listDue(now: Date, limit: number): Promise<Participant[]>;
  /**
   * ONE transaction: re-read the event; throw RegistrationError('registration_closed') unless
   * published and not deleted; throw 'no_spots' if seatsTaken + ticketCount > capacity;
   * insert the participant; seatsTaken += ticketCount.
   */
  insertParticipant(p: Participant): Promise<void>;
  /**
   * ONE transaction: read event + participant, call `mutate`. If it returns the same object,
   * write nothing. Otherwise write it and apply seatDelta(before, after) to seatsTaken.
   * Returns null when the participant does not exist. `mutate` may throw; nothing is written.
   */
  mutateParticipant(
    eventId: string,
    participantId: string,
    mutate: (p: Participant, event: EventRecord) => Participant,
  ): Promise<MutationResult | null>;
  /**
   * Claim a Stripe event id. false = already done, or claimed less than 10 minutes ago.
   * A claim older than that without completion is reclaimable; the process that took it died.
   */
  claimWebhookEvent(stripeEventId: string): Promise<boolean>;
  completeWebhookEvent(stripeEventId: string): Promise<void>;
  releaseWebhookEvent(stripeEventId: string): Promise<void>;
}

/** Optional stored-credit wallet. Every call is idempotent on `ref`. */
export interface BalanceLedger {
  debit(customerId: string, amount: number, currency: CurrencyCode, ref: string): Promise<'ok' | 'insufficient'>;
  credit(customerId: string, amount: number, currency: CurrencyCode, ref: string): Promise<void>;
  hasEntry(ref: string): Promise<boolean>;
}

export interface Notifier {
  send(kind: NotificationKind, participant: Participant, event: EventRecord, links: { manage: string }): Promise<void>;
}

export interface EngineUrls {
  success(eventId: string, participantId: string): string;
  cancel(eventId: string): string;
  manage(eventId: string, participantId: string, token: string): string;
}
```
