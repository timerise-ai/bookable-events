# No-show deposits

A free event costs the venue nothing to book and everything to no-show: the
seat was refused to someone who would have come. The fix is a **refundable card
hold** — money reserved on the customer's card, never charged if they arrive,
captured if they do not.

This is Stripe's *place a hold on a payment method* (separate authorization and
capture), driven by the participant state machine. Nobody is charged at signup;
arrival releases the hold; a no-show captures it; the venue keeps what it
captures.

## The constraint that shapes everything: holds expire

| Hold type | Who authorizes | Online validity (major networks) |
|---|---|---|
| Customer-initiated (CIT) — placed while the customer is in Checkout | Customer | ~7 days (Visa, Mastercard, Amex, Discover) |
| Merchant-initiated (MIT) — placed later with a saved card, `off_session` | Server | 7 days, **Visa: ~4 days 18 hours** |

After expiry Stripe cancels the PaymentIntent (`payment_intent.canceled`,
`cancellation_reason: 'automatic'`) and there is nothing left to capture. The
authoritative expiry is the charge's `payment_method_details.card.capture_before`;
the gateway reads it into `deposit.captureBefore` on every authorization.

So one mechanism cannot cover an event three weeks away. The hybrid:

| Event settles (end + grace) within | Strategy | What the customer does at signup | Hold placed |
|---|---|---|---|
| 6 days | `hold_now` | Authorizes the deposit in Checkout (`capture_method: 'manual'`) | Immediately (CIT, ~7 days) |
| more than 6 days | `save_card` | Saves a card in Checkout (`mode: 'setup'`) | By the tick at settle − 4 days (MIT, fits Visa's window) |

Both paths are card-only: most wallets and bank-redirect methods cannot be
authorized now and captured later, and SEPA/ACH debits cannot be held at all.

## Timeline (defaults from `DEFAULT_DEPOSIT_POLICY`)

```
 signup ─────── holdDueAt ─────── holdCutoffAt ─── start ──── end ── +2 h settle
   │  save_card    │ off-session hold   │ no hold yet?    │ check-in  │ capture no-shows,
   │  (Checkout    │ placed; decline →  │ seat released,  │ releases  │ release everyone
   │   setup)      │ email + retry link │ email sent      │ the hold  │ who arrived
   └─ hold_now: hold placed here, in Checkout ──────────────────────────┘
        settle − 4 d                start − 24 h                              ≤ capture_before − 6 h
```

| Policy field | Default | Change it when |
|---|---|---|
| `customerHoldWindowMs` | 6 days | Never above 6 — leave a day of margin inside the 7-day CIT window. |
| `merchantHoldWindowMs` | 4 days | Never above 4 — Visa MIT is 4 d 18 h. |
| `settleGraceMs` | 2 h | Staff need longer to finish check-ins after the event. |
| `captureSafetyMs` | 6 h | Your tick runs less often than every 5 minutes. |
| `holdCutoffBeforeStartMs` | 24 h | You want more time to re-offer released seats. |

Multi-day events whose hold would lapse before settlement are **renewed**: at
`capture_before − 6 h`, if the event has not ended, the tick places a fresh
off-session hold with the card saved during Checkout (`setup_future_usage:
'off_session'`) and cancels the old one. If renewal fails the deposit is
released, not captured early — charging someone before they had the chance to
arrive is worse than losing one deposit.

## What each outcome costs the customer

| Outcome | Deposit | Seat | Email |
|---|---|---|---|
| Arrives (all tickets) | Released at check-in | Used | `deposit_released` |
| Arrives with 1 of 3 tickets | 2 × per-ticket captured at settlement | Used | `deposit_captured` |
| Doesn't arrive | Captured in full at settlement | — | `deposit_captured` |
| Cancels before the deadline | Released | Released | `cancelled`, `deposit_released` |
| Cancels after the deadline | Captured in full, immediately | Released | `cancelled`, `deposit_captured` |
| Venue cancels the event | Released | Released | `cancelled`, `deposit_released` |
| Staff waive (illness, called ahead) | Released, `waivedBy` + reason stored | Unchanged | `deposit_released` |
| Off-session hold declined | None held | Kept until cutoff | `hold_failed` with a "use another card" link |
| Still no hold at cutoff | None | Released | `seat_released_no_hold` |

## Disclosure and consent

Saving a card for a later merchant-initiated charge requires the customer's
agreement to that use. Show, next to the registration button and in the
confirmation email, in the customer's language (string keys, not literals):

- the deposit amount per ticket and in total, with currency;
- that nothing is charged now (hold_now: "a temporary hold appears on your
  card"; save_card: "we will place a hold on this card on {holdDate}");
- that the hold is released when they check in, and captured if they do not
  arrive or cancel after {deadline};
- how to cancel (the manage link).

Whatever Checkout itself displays about saving the card, this copy is the
venue's own policy and must be shown by the venue.

## Tick handlers for deposits

`PLACE_HOLD`, `SETTLE` and the retry actions live in the tick. It is the only
cron the module needs — schedule it every 5 minutes
([operations.md](operations.md)).

```ts
// lib/events/tick.ts — the one cron. Runs every 5 minutes; works off `nextActionAt <= now`.
// Every branch is safe to run twice: Stripe calls carry idempotency keys and transitions
// on an already-moved participant are no-ops.
import type { EventsEngine } from './engine';
import type { WebhookHandler } from './webhook';
import { settleDueAt } from './deposit-schedule';
import { eventWindowUtc } from './time';
import type { Participant } from './types';

export interface TickReport {
  processed: number;
  failed: { participantId: string; action: string; error: string }[];
}

export function createTick(engine: EventsEngine, webhook: WebhookHandler) {
  const { store, gateway, ledger } = engine.deps;

  async function expirePending(p: Participant) {
    if (p.payment.method === 'BALANCE') {
      // The process died between debit and confirm: the ledger entry decides.
      const debited = ledger ? await ledger.hasEntry(`event:${p.id}`) : false;
      await engine.transition(p.eventId, p.id, debited ? { type: 'BALANCE_DEBITED' } : { type: 'CHECKOUT_EXPIRED' });
      return;
    }
    const sessionId = p.payment.checkoutSessionId ?? p.deposit.checkoutSessionId;
    if (sessionId) {
      const session = await gateway.retrieveCheckout(sessionId);
      if (session.status === 'complete') {
        await webhook.fulfilCheckout(session); // the webhook was lost or late — recover it
        return;
      }
      if (session.status === 'open') await gateway.expireCheckout(sessionId);
    }
    await engine.transition(p.eventId, p.id, { type: 'CHECKOUT_EXPIRED' });
  }

  async function placeHold(p: Participant) {
    const now = engine.now();
    const dep = p.deposit;
    if (dep.holdCutoffAt && now >= dep.holdCutoffAt) {
      await engine.transition(p.eventId, p.id, { type: 'GIVE_UP_HOLD' });
      return;
    }
    if (dep.status !== 'CARD_SAVED' || !dep.stripeCustomerId || !dep.paymentMethodId || !p.currency) return;
    const r = await gateway.placeOffSessionHold({
      customerId: dep.stripeCustomerId, paymentMethodId: dep.paymentMethodId,
      amount: dep.amount, currency: p.currency,
      ref: { eventId: p.eventId, participantId: p.id, attempt: p.checkoutAttempt },
      idempotencyKey: `hold:${p.id}`,
    });
    if (r.status === 'held') {
      await engine.transition(p.eventId, p.id, { type: 'HOLD_AUTHORIZED', paymentIntentId: r.paymentIntentId, captureBefore: r.captureBefore, paymentMethodId: r.paymentMethodId });
    } else if (r.status === 'requires_action') {
      await engine.transition(p.eventId, p.id, { type: 'HOLD_NEEDS_ACTION', paymentIntentId: r.paymentIntentId, error: r.error });
    } else {
      await engine.transition(p.eventId, p.id, { type: 'HOLD_DECLINED', error: r.error });
    }
  }

  async function settle(p: Participant) {
    const event = await store.getEvent(p.eventId);
    if (!event) return;
    const window = eventWindowUtc(event);
    const now = engine.now();
    const due = settleDueAt(window, p.deposit.captureBefore, engine.policy);
    if (due.clamped && now < window.end) {
      // The authorization would lapse before the event ends (multi-day event, short network
      // window). Place a fresh hold and let effectsOf cancel the old one.
      const dep = p.deposit;
      if (dep.stripeCustomerId && dep.paymentMethodId && p.currency && dep.paymentIntentId) {
        const r = await gateway.placeOffSessionHold({
          customerId: dep.stripeCustomerId, paymentMethodId: dep.paymentMethodId,
          amount: dep.amount, currency: p.currency,
          ref: { eventId: p.eventId, participantId: p.id, attempt: p.checkoutAttempt },
          idempotencyKey: `renew:${dep.paymentIntentId}`,
        });
        if (r.status === 'held') {
          await engine.transition(p.eventId, p.id, { type: 'HOLD_AUTHORIZED', paymentIntentId: r.paymentIntentId, captureBefore: r.captureBefore, paymentMethodId: r.paymentMethodId });
          return;
        }
      }
      // Renewal impossible: release rather than charge someone before they had a chance to arrive.
      await engine.transition(p.eventId, p.id, { type: 'WAIVE', by: 'system', reason: 'hold_renewal_failed' });
      return;
    }
    await engine.transition(p.eventId, p.id, { type: 'SETTLE' });
  }

  /** Re-issue the Stripe/ledger call for a RELEASING / CAPTURING / REFUND_PENDING participant. */
  async function retry(p: Participant) {
    const event = await store.getEvent(p.eventId);
    if (!event) return;
    const d = p.deposit;
    const effects =
      p.nextAction === 'RETRY_RELEASE' && d.paymentIntentId ? [{ type: 'cancel_intent' as const, paymentIntentId: d.paymentIntentId }]
      : p.nextAction === 'RETRY_CAPTURE' && d.paymentIntentId ? [{ type: 'capture_intent' as const, paymentIntentId: d.paymentIntentId, amount: d.captureAmount }]
      : p.nextAction === 'RETRY_REFUND' && p.payment.method !== 'FREE'
        ? [{ type: 'refund' as const, method: p.payment.method, paymentIntentId: p.payment.paymentIntentId, amount: p.payment.amount - p.payment.refundedAmount }]
      : [];
    // Push nextActionAt forward first, so a crash mid-call cannot hot-loop this participant.
    const bumped = await engine.patch(p.eventId, p.id, (cur) => cur);
    if (bumped) await engine.runEffects({ ...bumped, before: bumped.after }, effects);
  }

  return async function tick(limit = 100): Promise<TickReport> {
    const due = await store.listDue(engine.now(), limit);
    const report: TickReport = { processed: 0, failed: [] };
    for (const p of due) {
      try {
        switch (p.nextAction) {
          case 'EXPIRE_PENDING': await expirePending(p); break;
          case 'PLACE_HOLD': await placeHold(p); break;
          case 'SETTLE': await settle(p); break;
          case 'RETRY_RELEASE':
          case 'RETRY_CAPTURE':
          case 'RETRY_REFUND': await retry(p); break;
          case null: break;
        }
        report.processed++;
      } catch (err) {
        report.failed.push({ participantId: p.id, action: String(p.nextAction), error: err instanceof Error ? err.message : String(err) });
        // Recompute the schedule: RETRY_* rows back off 10 minutes, the rest retry next tick.
        await engine.patch(p.eventId, p.id, (cur) => cur).catch(() => undefined);
      }
    }
    return report;
  };
}
```

## The schedule functions

```ts
// lib/events/deposit-schedule.ts — when to hold, when to give up, when to settle.
//
// Card authorizations expire. Online, a hold the customer authorizes in Checkout (a
// customer-initiated transaction) lasts ~7 days on the major networks. A hold the server
// places later with a saved card (merchant-initiated) lasts only ~4 days 18 hours on Visa.
// The charge's `capture_before` is the authoritative expiry — always prefer it once known.
import type { DepositStrategy } from './types';

const HOUR = 3_600_000;
const DAY = 24 * HOUR;

export interface DepositPolicy {
  /** Max distance from now to settlement for a hold placed in Checkout (CIT, ~7 days). */
  customerHoldWindowMs: number;
  /** Lifetime assumed for an off-session hold (MIT; Visa is the shortest at 4d18h). */
  merchantHoldWindowMs: number;
  /** Time after the event ends for staff to finish check-ins before no-shows are captured. */
  settleGraceMs: number;
  /** Capture at least this long before the authorization's capture_before. */
  captureSafetyMs: number;
  /** No hold in place this long before the start → release the seat. */
  holdCutoffBeforeStartMs: number;
}

export const DEFAULT_DEPOSIT_POLICY: DepositPolicy = {
  customerHoldWindowMs: 6 * DAY,
  merchantHoldWindowMs: 4 * DAY,
  settleGraceMs: 2 * HOUR,
  captureSafetyMs: 6 * HOUR,
  holdCutoffBeforeStartMs: 24 * HOUR,
};

export interface EventWindow {
  start: Date;
  end: Date;
}

export function settleTargetAt(window: EventWindow, policy: DepositPolicy = DEFAULT_DEPOSIT_POLICY): Date {
  return new Date(window.end.getTime() + policy.settleGraceMs);
}

/** Hold in Checkout now if the hold survives until settlement; otherwise save the card. */
export function chooseDepositStrategy(
  window: EventWindow,
  now: Date,
  policy: DepositPolicy = DEFAULT_DEPOSIT_POLICY,
): DepositStrategy {
  const needed = settleTargetAt(window, policy).getTime() - now.getTime();
  return needed <= policy.customerHoldWindowMs ? 'hold_now' : 'save_card';
}

/** save_card: place the off-session hold late enough that it outlives settlement. */
export function holdPlacementDueAt(window: EventWindow, policy: DepositPolicy = DEFAULT_DEPOSIT_POLICY): Date {
  const due = settleTargetAt(window, policy).getTime() - policy.merchantHoldWindowMs;
  return new Date(Math.min(due, holdCutoffAt(window, policy).getTime()));
}

export function holdCutoffAt(window: EventWindow, policy: DepositPolicy = DEFAULT_DEPOSIT_POLICY): Date {
  return new Date(window.start.getTime() - policy.holdCutoffBeforeStartMs);
}

/**
 * When the tick settles a HELD deposit. `clamped` = the authorization expires before the
 * normal settlement time; if that lands before the event ends, the tick renews the hold
 * instead of capturing (see the tick's SETTLE handler).
 */
export function settleDueAt(
  window: EventWindow,
  captureBefore: Date | null,
  policy: DepositPolicy = DEFAULT_DEPOSIT_POLICY,
): { at: Date; clamped: boolean } {
  const target = settleTargetAt(window, policy).getTime();
  if (!captureBefore) return { at: new Date(target), clamped: false };
  const latest = captureBefore.getTime() - policy.captureSafetyMs;
  return latest < target ? { at: new Date(latest), clamped: true } : { at: new Date(target), clamped: false };
}
```

## Checklist

- [ ] Deposits are only configurable on free events (`parseEventInput` refuses
      `deposit` on a paid event).
- [ ] The Stripe account has card payments enabled for every currency a
      deposit is offered in.
- [ ] The staff check-in screen exists and is used — without it every attendee
      is a "no-show" at settlement ([ui.md](ui.md)).
- [ ] The tick runs every 5 minutes and alerts on `failed` entries.
- [ ] Disclosure copy is on the registration form and in the confirmation email.
- [ ] Test mode walkthrough in [stripe.md](stripe.md) passes, including the
      3DS card for the off-session path.
