# Participant lifecycle

Every change to a participant — a webhook, a staff tap, a cron decision, a
customer cancel — is an **action** fed to one pure function, `applyAction`. The
engine runs it inside a database transaction, diffs before/after into
**effects**, commits, and only then talks to Stripe or sends email.

```
 request / webhook / tick
        │
        ▼
 store.mutateParticipant(tx) ──► applyAction(p, action, now)   pure, may throw
        │                              │
        │                     same object? → no write, no effects (idempotent)
        ▼                              ▼
   commit (+ seatDelta)        scheduleNextAction → nextAction / nextActionAt
        │
        ▼
 effectsOf(before, after) ──► Stripe cancel / capture / refund / expire, email
        │                              │
        │                      success → follow-up action (HOLD_RELEASED …)
        ▼                      failure → nothing; RETRY_* already scheduled
```

**Transaction first, Stripe second** is the rule. Doing it the other way round
(call Stripe, then write) leaves money moved with no record when the write fails.
Doing it this way leaves a durable intent (`RELEASING`, `CAPTURING`,
`REFUND_PENDING`) that the tick retries with the same idempotency key until
Stripe confirms.

## The three sub-states

### Payment

```
              TICKET_PAID / BALANCE_DEBITED / HOLD_AUTHORIZED / CARD_SAVED
   PENDING ─────────────────────────────────────────────────────► CONFIRMED
     │  CHECKOUT_EXPIRED → EXPIRED                                   │
     │  CANCEL           → CANCELLED                                 │ CANCEL refund=full, paid → REFUND_PENDING → REFUNDED
     ▼                                                               │ CANCEL refund=none / free → CANCELLED
  (seat released)                                                    │ GIVE_UP_HOLD → CANCELLED
                                                                     │ EXTERNAL_REFUND (full) → REFUNDED
```

### Deposit (free events with a deposit)

```
 AWAITING_CARD ──CARD_SAVED──► CARD_SAVED ──tick PLACE_HOLD──► HELD
      │                            │  HOLD_NEEDS_ACTION → HOLD_REQUIRES_ACTION ─┐
      │ HOLD_AUTHORIZED (Checkout) │  HOLD_DECLINED     → HOLD_FAILED ──────────┤ customer resumes
      └────────────────────────────┴──────────────────────────────► HELD ◄──────┘ via new Checkout
                                                                    │
    CHECK_IN all / WAIVE / early CANCEL / SETTLE nobody missing ──► RELEASING ─► RELEASED
    SETTLE some missing / late CANCEL                           ──► CAPTURING ─► CAPTURED
    authorization lapses (Stripe cancels the intent)             ──────────────► RELEASED
    HOLD_* at cutoff → GIVE_UP_HOLD → RELEASED (seat released too)
```

### Attendance

`UNKNOWN → ARRIVED (count 1..ticketCount)` on check-in; `→ NO_SHOW` when staff
mark it or when settlement finds nobody arrived. Check-in can be corrected
(count changed) until settlement.

## Rules worth knowing before changing the machine

| Rule | Consequence if broken |
|---|---|
| A repeated action returns **the same object** | Redelivered webhooks and double taps become no-ops. Returning a copy writes again and re-sends every email. |
| Money for a seat that is gone raises `LatePaymentError` | The webhook refunds / cancels it. Silently confirming would push `seatsTaken` past capacity; silently ignoring keeps the money with no seat. |
| `HOLD_RELEASED` / `HOLD_CAPTURED` carry the `paymentIntentId` and ignore a mismatch | After a renewal or an abandoned 3DS attempt there are two intents; the old one's `canceled` event must not release the live hold. |
| Partial check-in keeps the hold `HELD` | Settlement captures `amountPerTicket × missing`. Releasing on the first arrival would let a group of four send one person. |
| Late cancel = no-show | Otherwise the cheapest no-show is "cancel from the car park". The deadline is `cancelDeadlineHours` before the start. |
| No `CANCEL` after `ARRIVED` | A present customer cannot refund themselves; staff can still waive. |
| `payment_intent.payment_failed` is not an action | Stripe sends it for each declined attempt while Checkout is still open. Treating it as terminal is how a customer ends up charged with no seat. Only `checkout.session.expired` ends a PENDING registration. |

```ts
// lib/events/participant-machine.ts — every participant change goes through `applyAction`.
//
// Pure: no I/O, no clock reads. The service runs it inside a DB transaction, diffs
// before/after with `effectsOf`, commits, and only then calls Stripe. A repeated action
// (webhook redelivery, double-clicked button) returns the SAME object — callers treat
// reference equality as "nothing to do".
import type { Actor, Participant, PaymentStatus } from './types';

export type ParticipantAction =
  | { type: 'TICKET_PAID'; paymentIntentId: string }
  | { type: 'BALANCE_DEBITED' }
  | { type: 'HOLD_AUTHORIZED'; paymentIntentId: string; captureBefore: Date; paymentMethodId: string | null }
  | { type: 'CARD_SAVED'; paymentMethodId: string }
  | { type: 'HOLD_NEEDS_ACTION'; paymentIntentId: string; error: string }
  | { type: 'HOLD_DECLINED'; error: string }
  | { type: 'CHECKOUT_EXPIRED' }
  | { type: 'CHECK_IN'; count: number; by: string }
  | { type: 'MARK_NO_SHOW'; by: string }
  | { type: 'WAIVE'; by: string; reason: string }
  | { type: 'SETTLE' }
  | { type: 'HOLD_RELEASED'; paymentIntentId: string; reason?: string }
  | { type: 'HOLD_CAPTURED'; paymentIntentId: string; amount: number }
  | { type: 'CANCEL'; actor: Actor; late: boolean; refund: 'full' | 'none' }
  | { type: 'REFUND_SUCCEEDED'; amount: number }
  | { type: 'EXTERNAL_REFUND'; amount: number }
  | { type: 'GIVE_UP_HOLD' };

export class TransitionError extends Error {
  constructor(public readonly code: string, message: string) {
    super(message);
    this.name = 'TransitionError';
  }
}

/** Money arrived for a seat that is already gone. The service must give it back. */
export class LatePaymentError extends TransitionError {
  constructor(message: string) {
    super('late_payment', message);
    this.name = 'LatePaymentError';
  }
}

const SEAT_HOLDING: ReadonlySet<PaymentStatus> = new Set<PaymentStatus>(['PENDING', 'CONFIRMED']);
const PRE_HOLD = new Set(['AWAITING_CARD', 'CARD_SAVED', 'HOLD_REQUIRES_ACTION', 'HOLD_FAILED']);
const NO_HOLD_YET = new Set(['CARD_SAVED', 'HOLD_REQUIRES_ACTION', 'HOLD_FAILED']);

export function holdsSeat(p: Pick<Participant, 'payment'>): boolean {
  return SEAT_HOLDING.has(p.payment.status);
}

/** Change to the event's seatsTaken implied by this transition. */
export function seatDelta(before: Participant | null, after: Participant): number {
  const was = before && holdsSeat(before) ? before.ticketCount : 0;
  const is = holdsSeat(after) ? after.ticketCount : 0;
  return is - was;
}

function fail(p: Participant, action: ParticipantAction): never {
  throw new TransitionError(
    'illegal_transition',
    `${action.type} not allowed (payment=${p.payment.status}, deposit=${p.deposit.status}, attendance=${p.attendance.status})`,
  );
}

function isSeatGone(p: Participant): boolean {
  return !holdsSeat(p);
}

export function applyAction(p: Participant, action: ParticipantAction, now: Date): Participant {
  const pay = p.payment;
  const dep = p.deposit;
  const att = p.attendance;

  switch (action.type) {
    case 'TICKET_PAID': {
      if (pay.status === 'CONFIRMED' && pay.paymentIntentId === action.paymentIntentId) return p;
      if (isSeatGone(p)) throw new LatePaymentError(`ticket paid for ${pay.status} participant ${p.id}`);
      if (pay.status !== 'PENDING' || pay.method !== 'STRIPE') fail(p, action);
      return { ...p, payment: { ...pay, status: 'CONFIRMED', paymentIntentId: action.paymentIntentId, expiresAt: null }, updatedAt: now };
    }

    case 'BALANCE_DEBITED': {
      if (pay.status === 'CONFIRMED') return p;
      if (pay.status !== 'PENDING' || pay.method !== 'BALANCE') fail(p, action);
      return { ...p, payment: { ...pay, status: 'CONFIRMED', expiresAt: null }, updatedAt: now };
    }

    case 'HOLD_AUTHORIZED': {
      if (dep.status === 'HELD' && dep.paymentIntentId === action.paymentIntentId) return p;
      if (isSeatGone(p)) throw new LatePaymentError(`hold authorized for ${pay.status} participant ${p.id}`);
      // HELD with a different intent = renewal of a hold that would expire before settlement.
      if (!PRE_HOLD.has(dep.status) && dep.status !== 'HELD') fail(p, action);
      return {
        ...p,
        payment: pay.status === 'PENDING' ? { ...pay, status: 'CONFIRMED', expiresAt: null } : pay,
        deposit: {
          ...dep, status: 'HELD',
          paymentIntentId: action.paymentIntentId,
          paymentMethodId: action.paymentMethodId ?? dep.paymentMethodId,
          captureBefore: action.captureBefore,
          lastError: null,
        },
        updatedAt: now,
      };
    }

    case 'CARD_SAVED': {
      if (dep.status === 'CARD_SAVED' && dep.paymentMethodId === action.paymentMethodId) return p;
      if (isSeatGone(p)) throw new LatePaymentError(`card saved for ${pay.status} participant ${p.id}`);
      if (dep.status !== 'AWAITING_CARD') fail(p, action);
      return {
        ...p,
        payment: { ...pay, status: 'CONFIRMED', expiresAt: null },
        deposit: { ...dep, status: 'CARD_SAVED', paymentMethodId: action.paymentMethodId },
        updatedAt: now,
      };
    }

    case 'HOLD_NEEDS_ACTION': {
      if (dep.status === 'HOLD_REQUIRES_ACTION') return p;
      if (!NO_HOLD_YET.has(dep.status)) fail(p, action);
      return { ...p, deposit: { ...dep, status: 'HOLD_REQUIRES_ACTION', paymentIntentId: action.paymentIntentId, lastError: action.error }, updatedAt: now };
    }

    case 'HOLD_DECLINED': {
      if (dep.status === 'HOLD_FAILED') return p;
      if (!NO_HOLD_YET.has(dep.status)) fail(p, action);
      return { ...p, deposit: { ...dep, status: 'HOLD_FAILED', lastError: action.error }, updatedAt: now };
    }

    case 'CHECKOUT_EXPIRED': {
      if (pay.status !== 'PENDING') return p; // paid, cancelled or already expired: nothing to do
      return { ...p, payment: { ...pay, status: 'EXPIRED', expiresAt: null }, updatedAt: now };
    }

    case 'CHECK_IN': {
      if (pay.status !== 'CONFIRMED') fail(p, action);
      if (!Number.isInteger(action.count) || action.count < 1 || action.count > p.ticketCount) {
        throw new TransitionError('bad_count', `arrived count must be 1..${p.ticketCount}`);
      }
      if (att.status === 'ARRIVED' && att.arrivedCount === action.count) return p;
      const attendance = { status: 'ARRIVED' as const, arrivedCount: action.count, checkedInAt: now, checkedInBy: action.by };
      const everyone = action.count === p.ticketCount;
      let deposit = dep;
      if (dep.status === 'HELD' && everyone) deposit = { ...dep, status: 'RELEASING' };
      else if (NO_HOLD_YET.has(dep.status)) deposit = { ...dep, status: 'RELEASED' }; // nothing to capture
      // HELD + partial arrival stays HELD: SETTLE captures only the missing tickets.
      return { ...p, attendance, deposit, updatedAt: now };
    }

    case 'MARK_NO_SHOW': {
      if (pay.status !== 'CONFIRMED') fail(p, action);
      if (att.status === 'NO_SHOW') return p;
      return { ...p, attendance: { status: 'NO_SHOW', arrivedCount: 0, checkedInAt: null, checkedInBy: action.by }, updatedAt: now };
    }

    case 'WAIVE': {
      if (dep.status === 'RELEASING' || dep.status === 'RELEASED') return p;
      const waived = { waivedBy: action.by, waiveReason: action.reason };
      if (dep.status === 'HELD') return { ...p, deposit: { ...dep, ...waived, status: 'RELEASING' }, updatedAt: now };
      if (NO_HOLD_YET.has(dep.status)) return { ...p, deposit: { ...dep, ...waived, status: 'RELEASED' }, updatedAt: now };
      fail(p, action);
    }

    case 'SETTLE': {
      if (dep.status !== 'HELD') return p;
      const arrived = att.status === 'ARRIVED' ? att.arrivedCount : 0;
      const missing = p.ticketCount - arrived;
      if (missing <= 0) return { ...p, deposit: { ...dep, status: 'RELEASING' }, updatedAt: now };
      return {
        ...p,
        attendance: att.status === 'UNKNOWN' ? { ...att, status: 'NO_SHOW' } : att,
        deposit: { ...dep, status: 'CAPTURING', captureAmount: dep.amountPerTicket * missing },
        updatedAt: now,
      };
    }

    case 'HOLD_RELEASED': {
      // A superseded intent (renewed hold, abandoned 3DS attempt) must not touch the live one.
      if (action.paymentIntentId !== dep.paymentIntentId || dep.status === 'RELEASED') return p;
      // HELD → RELEASED happens when the authorization lapses and Stripe cancels it.
      if (dep.status !== 'RELEASING' && dep.status !== 'HELD' && dep.status !== 'HOLD_REQUIRES_ACTION') fail(p, action);
      return { ...p, deposit: { ...dep, status: 'RELEASED', lastError: action.reason ?? dep.lastError }, updatedAt: now };
    }

    case 'HOLD_CAPTURED': {
      if (action.paymentIntentId !== dep.paymentIntentId || dep.status === 'CAPTURED') return p;
      if (dep.status !== 'CAPTURING' && dep.status !== 'HELD') fail(p, action);
      return { ...p, deposit: { ...dep, status: 'CAPTURED', captureAmount: action.amount }, updatedAt: now };
    }

    case 'CANCEL': {
      if (!holdsSeat(p)) return p;
      if (att.status === 'ARRIVED') throw new TransitionError('already_arrived', 'cannot cancel after check-in');
      const cancelled = { cancelledAt: now, cancelledBy: action.actor };
      if (pay.status === 'PENDING') {
        return { ...p, ...cancelled, payment: { ...pay, status: 'CANCELLED', expiresAt: null }, updatedAt: now };
      }
      const refundable = pay.method !== 'FREE' && pay.amount > 0 && action.refund === 'full';
      let deposit = dep;
      if (dep.status === 'HELD') {
        deposit = action.late
          ? { ...dep, status: 'CAPTURING', captureAmount: dep.amount } // late cancel = no-show
          : { ...dep, status: 'RELEASING' };
      } else if (NO_HOLD_YET.has(dep.status)) {
        deposit = { ...dep, status: 'RELEASED' };
      }
      return {
        ...p, ...cancelled, deposit,
        payment: { ...pay, status: refundable ? 'REFUND_PENDING' : 'CANCELLED' },
        updatedAt: now,
      };
    }

    case 'REFUND_SUCCEEDED': {
      if (pay.status === 'REFUNDED') return p;
      if (pay.status !== 'REFUND_PENDING') fail(p, action);
      return { ...p, payment: { ...pay, status: 'REFUNDED', refundedAmount: action.amount }, updatedAt: now };
    }

    case 'EXTERNAL_REFUND': {
      // A refund issued in the Stripe dashboard. Full refund releases the seat.
      if (action.amount <= pay.refundedAmount) return p;
      const full = action.amount >= pay.amount;
      const status: PaymentStatus = full ? 'REFUNDED' : pay.status;
      return {
        ...p,
        ...(full && holdsSeat(p) ? { cancelledAt: now, cancelledBy: 'admin' as const } : {}),
        payment: { ...pay, status, refundedAmount: action.amount },
        updatedAt: now,
      };
    }

    case 'GIVE_UP_HOLD': {
      if (!NO_HOLD_YET.has(dep.status) || !holdsSeat(p)) return p;
      return {
        ...p,
        cancelledAt: now, cancelledBy: 'system',
        payment: { ...pay, status: 'CANCELLED', expiresAt: null },
        deposit: { ...dep, status: 'RELEASED', lastError: dep.lastError ?? 'no_hold_by_cutoff' },
        updatedAt: now,
      };
    }
  }
}
```

## Effects and the due-work pointer

`effectsOf(before, after)` is a pure diff. The engine executes it after commit;
each successful Stripe call feeds a confirming action back in
(`cancel_intent` → `HOLD_RELEASED`, `capture_intent` → `HOLD_CAPTURED`,
`refund` → `REFUND_SUCCEEDED`). Stripe's own webhooks deliver the same
confirmations, so whichever arrives first wins and the other is a no-op.

`scheduleNextAction` computes the only thing the background tick queries:

| Participant state | `nextAction` | `nextActionAt` |
|---|---|---|
| `payment PENDING` | `EXPIRE_PENDING` | Checkout `expires_at` + 5 min |
| `deposit RELEASING` | `RETRY_RELEASE` | now + 10 min |
| `deposit CAPTURING` | `RETRY_CAPTURE` | now + 10 min |
| `payment REFUND_PENDING` | `RETRY_REFUND` | now + 10 min |
| `deposit CARD_SAVED` | `PLACE_HOLD` | `holdDueAt` |
| `deposit HOLD_FAILED / HOLD_REQUIRES_ACTION` | `PLACE_HOLD` (gives up) | `holdCutoffAt` |
| `deposit HELD` | `SETTLE` | `settleDueAt(window, captureBefore)` |
| anything else | `null` | `null` |

The 5-minute grace on PENDING lets the `checkout.session.completed` webhook win
the race against the tick; when it does not (webhook lost, endpoint down), the
tick retrieves the session and fulfils it itself.

```ts
// lib/events/effects.ts — what must happen outside the database after a transition,
// and when the tick must look at this participant next. Both pure.
import { settleDueAt, type DepositPolicy, type EventWindow, DEFAULT_DEPOSIT_POLICY } from './deposit-schedule';
import type { NextAction, Participant } from './types';

export type NotificationKind =
  | 'registered' | 'deposit_held' | 'card_saved' | 'hold_action_required' | 'hold_failed'
  | 'seat_released_no_hold' | 'deposit_released' | 'deposit_captured' | 'cancelled' | 'refunded' | 'expired';

export type Effect =
  | { type: 'expire_checkout'; sessionId: string }
  | { type: 'cancel_intent'; paymentIntentId: string }       // release a hold / abandon a 3DS intent
  | { type: 'capture_intent'; paymentIntentId: string; amount: number }
  | { type: 'refund'; method: 'STRIPE' | 'BALANCE'; paymentIntentId: string | null; amount: number }
  | { type: 'notify'; kind: NotificationKind };

export function effectsOf(before: Participant, after: Participant): Effect[] {
  if (before === after) return [];
  const out: Effect[] = [];
  const bp = before.payment, ap = after.payment, bd = before.deposit, ad = after.deposit;

  // Seat lost while a Checkout Session may still be open: kill the session so it can't be paid.
  if (bp.status === 'PENDING' && ap.status !== 'PENDING' && ap.status !== 'CONFIRMED') {
    if (ap.checkoutSessionId) out.push({ type: 'expire_checkout', sessionId: ap.checkoutSessionId });
    if (ad.checkoutSessionId && ad.checkoutSessionId !== ap.checkoutSessionId) {
      out.push({ type: 'expire_checkout', sessionId: ad.checkoutSessionId });
    }
    out.push({ type: 'notify', kind: ap.status === 'EXPIRED' ? 'expired' : 'cancelled' });
  }

  if (ad.status === 'RELEASING' && bd.status !== 'RELEASING' && ad.paymentIntentId) {
    out.push({ type: 'cancel_intent', paymentIntentId: ad.paymentIntentId });
  }
  // An intent that is no longer the live hold: abandoned 3DS attempt, or the old half of a renewal.
  if (bd.paymentIntentId && bd.paymentIntentId !== ad.paymentIntentId &&
      (bd.status === 'HELD' || bd.status === 'HOLD_REQUIRES_ACTION')) {
    out.push({ type: 'cancel_intent', paymentIntentId: bd.paymentIntentId });
  }
  if (bd.status === 'HOLD_REQUIRES_ACTION' && ad.status === 'RELEASED' && bd.paymentIntentId) {
    out.push({ type: 'cancel_intent', paymentIntentId: bd.paymentIntentId });
  }
  if (ad.status === 'CAPTURING' && bd.status !== 'CAPTURING' && ad.paymentIntentId) {
    out.push({ type: 'capture_intent', paymentIntentId: ad.paymentIntentId, amount: ad.captureAmount });
  }
  if (ap.status === 'REFUND_PENDING' && bp.status !== 'REFUND_PENDING' && ap.method !== 'FREE') {
    out.push({ type: 'refund', method: ap.method, paymentIntentId: ap.paymentIntentId, amount: ap.amount - ap.refundedAmount });
  }

  if (bp.status === 'PENDING' && ap.status === 'CONFIRMED') {
    const kind = ad.status === 'HELD' ? 'deposit_held' : ad.status === 'CARD_SAVED' ? 'card_saved' : 'registered';
    out.push({ type: 'notify', kind });
  }
  if (bp.status === 'CONFIRMED' && (ap.status === 'CANCELLED' || ap.status === 'REFUND_PENDING')) {
    out.push({ type: 'notify', kind: after.cancelledBy === 'system' && bd.status !== 'HELD' ? 'seat_released_no_hold' : 'cancelled' });
  }
  if (bd.status !== ad.status) {
    if (ad.status === 'HOLD_REQUIRES_ACTION') out.push({ type: 'notify', kind: 'hold_action_required' });
    if (ad.status === 'HOLD_FAILED') out.push({ type: 'notify', kind: 'hold_failed' });
    if (ad.status === 'RELEASED' && bd.status === 'RELEASING') out.push({ type: 'notify', kind: 'deposit_released' });
    if (ad.status === 'CAPTURED') out.push({ type: 'notify', kind: 'deposit_captured' });
  }
  if (ap.status === 'REFUNDED' && bp.status !== 'REFUNDED') out.push({ type: 'notify', kind: 'refunded' });
  return out;
}

const RETRY_MS = 10 * 60_000;
const PENDING_GRACE_MS = 5 * 60_000; // let the checkout.session.* webhook win the race first

/** The single due-work pointer the tick queries on: `nextActionAt <= now`. */
export function scheduleNextAction(
  p: Participant,
  window: EventWindow,
  now: Date,
  policy: DepositPolicy = DEFAULT_DEPOSIT_POLICY,
): { nextAction: NextAction | null; nextActionAt: Date | null } {
  const at = (ms: number) => new Date(ms);
  const pay = p.payment, dep = p.deposit;

  if (pay.status === 'PENDING') {
    const expires = pay.expiresAt ?? now;
    return { nextAction: 'EXPIRE_PENDING', nextActionAt: at(expires.getTime() + PENDING_GRACE_MS) };
  }
  if (dep.status === 'RELEASING') return { nextAction: 'RETRY_RELEASE', nextActionAt: at(now.getTime() + RETRY_MS) };
  if (dep.status === 'CAPTURING') return { nextAction: 'RETRY_CAPTURE', nextActionAt: at(now.getTime() + RETRY_MS) };
  if (pay.status === 'REFUND_PENDING') return { nextAction: 'RETRY_REFUND', nextActionAt: at(now.getTime() + RETRY_MS) };
  if (pay.status !== 'CONFIRMED') return { nextAction: null, nextActionAt: null };

  if (dep.status === 'CARD_SAVED' && dep.holdDueAt) return { nextAction: 'PLACE_HOLD', nextActionAt: dep.holdDueAt };
  if ((dep.status === 'HOLD_FAILED' || dep.status === 'HOLD_REQUIRES_ACTION') && dep.holdCutoffAt) {
    return { nextAction: 'PLACE_HOLD', nextActionAt: dep.holdCutoffAt }; // at cutoff: give up
  }
  if (dep.status === 'HELD') {
    return { nextAction: 'SETTLE', nextActionAt: settleDueAt(window, dep.captureBefore, policy).at };
  }
  return { nextAction: null, nextActionAt: null };
}
```

## Checklist

- [ ] No code writes `payment`, `deposit` or `attendance` except through
      `engine.transition` (or `engine.patch` for ids and timestamps).
- [ ] No Stripe call happens inside a database transaction.
- [ ] Every new action has a no-op branch for "already applied".
- [ ] A new sub-state has a row in the `scheduleNextAction` table.
