# Stripe

Every Stripe call lives in one class, `StripeGateway`; every Stripe event lands
in one handler, `createWebhookHandler`. The engine depends on the
`PaymentGateway` type, so tests use a fake and another processor can stand in.

## Checkout Sessions the module creates

| Kind (`metadata.kind`) | `mode` | Key parameters | Completes as |
|---|---|---|---|
| `event_ticket` | `payment` | `price_data` for the frozen total, `customer_email`, `expires_at` = now + 30 min | `TICKET_PAID` |
| `event_deposit_hold` | `payment` | `payment_method_types: ['card']`, `payment_intent_data.capture_method: 'manual'`, `setup_future_usage: 'off_session'`, `customer` | `HOLD_AUTHORIZED` (intent in `requires_capture`) |
| `event_deposit_setup` | `setup` | `payment_method_types: ['card']`, `customer` (no `currency` needed with an explicit list) | `CARD_SAVED` |

Every session carries `metadata: { kind, eventId, participantId }` on the session
**and** on the PaymentIntent / SetupIntent, plus `client_reference_id`. Handlers
route on `kind`; anything without an `event_*` kind belongs to another module
and is ignored, so this handler can share a webhook endpoint with the host's
other payment flows.

| Choice | Reason |
|---|---|
| `expires_at` = 30 min (Stripe's minimum) | Stripe's default is 24 h. An unpaid seat held for a day is a seat nobody can buy; the source's 30-minute cleanup cron fighting a 24-hour session is how "pay again" links died. |
| Idempotency key `checkout:<kind>:<participantId>:<attempt>` | A retried request returns the same session; "pay again" bumps `attempt` for a new one. |
| One Stripe Customer per participant | Looking customers up by email can return someone else's customer, with their saved cards, for a shared or mistyped address. |
| No hard-coded `payment_method_types` on tickets | Dynamic payment methods follow the Dashboard. Hard-coding a local method (say, one that only works in one currency) breaks every other currency at session creation. |
| Card-only for deposits | Manual capture is supported by cards and a few BNPL wallets; bank redirects and debits cannot be held. |

## Handling Stripe events

| Event | Action |
|---|---|
| `checkout.session.completed`, `…async_payment_succeeded` | Verify, then `TICKET_PAID` / `HOLD_AUTHORIZED` / `CARD_SAVED`. Ticket: `payment_status === 'paid'` **and** `amount_total` + `currency` equal the frozen amount, or refund. |
| `checkout.session.expired`, `…async_payment_failed` | `CHECKOUT_EXPIRED` — only if it is the participant's **current** session. |
| `payment_intent.amount_capturable_updated` | `HOLD_AUTHORIZED` (a hold that became capturable after the session completed). |
| `payment_intent.canceled` (deposit) | `HOLD_RELEASED` for that intent id; `automatic` → `lastError: 'authorization_expired'`. |
| `payment_intent.succeeded` (deposit) | `HOLD_CAPTURED` with `amount_received`. |
| `charge.refunded` (ticket) | `REFUND_SUCCEEDED`, or `EXTERNAL_REFUND` for a refund issued in the Dashboard (a full refund releases the seat). Metadata is read from the PaymentIntent. |
| `payment_intent.payment_failed` | **Ignored on purpose.** Fired per declined attempt while Checkout is still open. |

Subscribe the endpoint to the eight handled types (everything above except `payment_intent.payment_failed`).

**Idempotency, three layers.** (1) `claimWebhookEvent` records the event id —
a completed id is skipped; a claim older than 10 minutes without completion is
reclaimable, because the process holding it died. (2) Every transition is a
no-op when already applied. (3) Every Stripe write carries an idempotency key.
A handler error **releases the claim and returns 500** so Stripe redelivers; a
`try/catch` that logs and returns 200 turns one transient error into a
permanently lost payment.

**Late money.** If a completion arrives for a participant whose seat is already
gone (expired, cancelled), the machine raises `LatePaymentError` and the handler
gives the money back: refund for a ticket, cancel for a hold. It never
resurrects the seat — capacity may already be resold.

```ts
// lib/events/webhook.ts — Stripe → participant transitions. Every handler is idempotent;
// the event-id ledger only saves work. A thrown error releases the claim and the route
// answers 500 so Stripe redelivers — never swallow a failure into a 200.
import type Stripe from 'stripe';
import { fromStripeAmount, toStripeAmount } from './currency';
import type { EventsEngine } from './engine';
import { LatePaymentError } from './participant-machine';
import { captureBeforeOf } from './stripe-gateway';
import type { Participant } from './types';

const KINDS = ['event_ticket', 'event_deposit_hold', 'event_deposit_setup'] as const;
type Kind = (typeof KINDS)[number];

function refOf(md: Stripe.Metadata | null | undefined): { kind: Kind; eventId: string; participantId: string } | null {
  const kind = md?.kind as Kind | undefined;
  if (!kind || !KINDS.includes(kind) || !md?.eventId || !md.participantId) return null;
  return { kind, eventId: md.eventId, participantId: md.participantId };
}

export function createWebhookHandler(engine: EventsEngine) {
  const { store, gateway } = engine.deps;

  /** Shared by the webhook and the tick's reconcile path. */
  async function fulfilCheckout(session: Stripe.Checkout.Session): Promise<void> {
    const ref = refOf(session.metadata);
    if (!ref) return; // not ours — another module owns it
    const p = await store.getParticipant(ref.eventId, ref.participantId);
    if (!p) throw new Error(`checkout ${session.id} for unknown participant ${ref.participantId}`);

    if (ref.kind === 'event_ticket') {
      const piId = typeof session.payment_intent === 'string' ? session.payment_intent : session.payment_intent?.id;
      if (session.payment_status !== 'paid' || !piId) return; // async methods finish later
      const currency = p.currency ?? '';
      const expected = toStripeAmount(p.payment.amount, currency);
      if (session.currency !== currency || session.amount_total !== expected) {
        // Never confirm a seat for a different amount than the one frozen at registration.
        await gateway.refund(piId, fromStripeAmount(session.amount_total ?? 0, session.currency ?? currency), session.currency ?? currency, `mismatch:${p.id}`);
        throw new Error(`amount mismatch on ${session.id}: got ${session.amount_total} ${session.currency}, expected ${expected} ${currency}`);
      }
      await applyPaid(p, { type: 'TICKET_PAID', paymentIntentId: piId }, () =>
        gateway.refund(piId, p.payment.amount, currency, `late:${p.id}`).then(() => undefined));
      return;
    }

    if (ref.kind === 'event_deposit_hold') {
      const piId = typeof session.payment_intent === 'string' ? session.payment_intent : session.payment_intent?.id;
      if (!piId) return;
      const pi = await gateway.retrievePaymentIntent(piId);
      if (pi.status !== 'requires_capture') return; // amount_capturable_updated will follow
      await applyHold(p, pi);
      return;
    }

    // event_deposit_setup — webhook payloads carry ids only, so read the SetupIntent itself.
    const siId = typeof session.setup_intent === 'string' ? session.setup_intent : session.setup_intent?.id;
    if (!siId) return;
    const si = await gateway.retrieveSetupIntent(siId);
    const pm = typeof si.payment_method === 'string' ? si.payment_method : si.payment_method?.id;
    if (si.status !== 'succeeded' || !pm) return;
    await applyPaid(p, { type: 'CARD_SAVED', paymentMethodId: pm }, async () => undefined);
  }

  async function applyHold(p: Participant, pi: Stripe.PaymentIntent) {
    const pm = typeof pi.payment_method === 'string' ? pi.payment_method : pi.payment_method?.id ?? null;
    await applyPaid(
      p,
      { type: 'HOLD_AUTHORIZED', paymentIntentId: pi.id, captureBefore: captureBeforeOf(pi), paymentMethodId: pm },
      () => gateway.cancelIntent(pi.id).then(() => undefined),
    );
  }

  /** Apply a "money arrived" action; if the seat is already gone, give the money back. */
  async function applyPaid(
    p: Participant,
    action: Parameters<EventsEngine['transition']>[2],
    giveBack: () => Promise<void>,
  ) {
    try {
      await engine.transition(p.eventId, p.id, action);
    } catch (err) {
      if (!(err instanceof LatePaymentError)) throw err;
      await giveBack();
      console.error(`[events] late payment returned for participant ${p.id}: ${err.message}`);
    }
  }

  async function handle(event: Stripe.Event): Promise<'processed' | 'duplicate'> {
    if (!(await store.claimWebhookEvent(event.id))) return 'duplicate';
    try {
      await dispatch(event);
      await store.completeWebhookEvent(event.id);
      return 'processed';
    } catch (err) {
      await store.releaseWebhookEvent(event.id);
      throw err;
    }
  }

  async function dispatch(event: Stripe.Event): Promise<void> {
    switch (event.type) {
      case 'checkout.session.completed':
      case 'checkout.session.async_payment_succeeded':
        return fulfilCheckout(event.data.object);

      case 'checkout.session.expired':
      case 'checkout.session.async_payment_failed': {
        const s = event.data.object;
        const ref = refOf(s.metadata);
        if (!ref) return;
        const p = await store.getParticipant(ref.eventId, ref.participantId);
        // Only the CURRENT session expiring releases the seat — "pay again" supersedes old ones.
        if (!p || (p.payment.checkoutSessionId !== s.id && p.deposit.checkoutSessionId !== s.id)) return;
        if (p.payment.status === 'PENDING') await engine.transition(p.eventId, p.id, { type: 'CHECKOUT_EXPIRED' });
        return;
      }

      // Deliberately NOT handled: payment_intent.payment_failed. Stripe sends it for every
      // declined attempt while the Checkout page is still open and the customer can retry.
      // Releasing the seat there charges a customer who retries successfully for nothing.

      case 'payment_intent.amount_capturable_updated': {
        const ref = refOf(event.data.object.metadata);
        if (!ref || ref.kind !== 'event_deposit_hold') return;
        const p = await store.getParticipant(ref.eventId, ref.participantId);
        if (!p) return;
        await applyHold(p, await gateway.retrievePaymentIntent(event.data.object.id));
        return;
      }

      case 'payment_intent.canceled': {
        const pi = event.data.object;
        const ref = refOf(pi.metadata);
        if (!ref || ref.kind !== 'event_deposit_hold') return;
        const reason = pi.cancellation_reason === 'automatic' ? 'authorization_expired' : undefined;
        await engine.transition(ref.eventId, ref.participantId, { type: 'HOLD_RELEASED', paymentIntentId: pi.id, reason });
        return;
      }

      case 'payment_intent.succeeded': {
        const pi = event.data.object;
        const ref = refOf(pi.metadata);
        if (!ref || ref.kind !== 'event_deposit_hold' || !pi.currency) return;
        await engine.transition(ref.eventId, ref.participantId, {
          type: 'HOLD_CAPTURED', paymentIntentId: pi.id, amount: fromStripeAmount(pi.amount_received, pi.currency),
        });
        return;
      }

      case 'charge.refunded': {
        const charge = event.data.object;
        const piId = typeof charge.payment_intent === 'string' ? charge.payment_intent : charge.payment_intent?.id;
        if (!piId) return;
        const ref = refOf((await gateway.retrievePaymentIntent(piId)).metadata); // the intent carries our metadata
        if (!ref || ref.kind !== 'event_ticket') return;
        const p = await store.getParticipant(ref.eventId, ref.participantId);
        if (!p) return;
        const amount = fromStripeAmount(charge.amount_refunded, charge.currency);
        await engine.transition(p.eventId, p.id, p.payment.status === 'REFUND_PENDING' && amount >= p.payment.amount
          ? { type: 'REFUND_SUCCEEDED', amount }
          : { type: 'EXTERNAL_REFUND', amount });
        return;
      }

      default:
        return;
    }
  }

  return { handle, fulfilCheckout };
}

export type WebhookHandler = ReturnType<typeof createWebhookHandler>;
```

## The gateway

```ts
// lib/events/stripe-gateway.ts — every Stripe call the module makes, in one place.
// Amounts in/out are STORED amounts; conversion to Stripe units happens only here.
import Stripe from 'stripe';
import { fromStripeAmount, toStripeAmount } from './currency';
import type { CurrencyCode } from './types';

const CHECKOUT_TTL_MS = 30 * 60_000; // Stripe's minimum; keeps unpaid seats short-lived

export interface CheckoutRef {
  eventId: string;
  participantId: string;
  attempt: number; // bump for "pay again" so the idempotency key differs
}

export type CheckoutKind = 'event_ticket' | 'event_deposit_hold' | 'event_deposit_setup';

export interface CreatedCheckout {
  sessionId: string;
  url: string;
  expiresAt: Date;
}

export type OffSessionHoldResult =
  | { status: 'held'; paymentIntentId: string; captureBefore: Date; paymentMethodId: string | null }
  | { status: 'requires_action'; paymentIntentId: string; error: string }
  | { status: 'declined'; error: string };

function metadata(kind: CheckoutKind, ref: CheckoutRef): Stripe.MetadataParam {
  return { kind, eventId: ref.eventId, participantId: ref.participantId };
}

function toCreated(session: Stripe.Checkout.Session): CreatedCheckout {
  if (!session.url) throw new Error(`Checkout session ${session.id} has no URL`);
  return { sessionId: session.id, url: session.url, expiresAt: new Date(session.expires_at * 1000) };
}

/** Authorization expiry reported by the card network; fallback = the conservative MIT window. */
export function captureBeforeOf(pi: Stripe.PaymentIntent, fallbackMs = 4 * 86_400_000): Date {
  const charge = typeof pi.latest_charge === 'object' ? pi.latest_charge : null;
  const unix = charge?.payment_method_details?.card?.capture_before;
  return unix ? new Date(unix * 1000) : new Date(pi.created * 1000 + fallbackMs);
}

/** The engine depends on this shape, so tests and other providers can stand in for Stripe. */
export type PaymentGateway = Pick<StripeGateway, keyof StripeGateway>;

export class StripeGateway {
  constructor(private readonly stripe: Stripe) {}

  async createCustomer(email: string, name: string, participantId: string): Promise<string> {
    // One customer per participant: never look customers up by email — that can return
    // someone else's customer (and their saved cards) for a shared or typo'd address.
    const c = await this.stripe.customers.create(
      { email, name, metadata: { participantId } },
      { idempotencyKey: `customer:${participantId}` },
    );
    return c.id;
  }

  async createTicketCheckout(args: {
    ref: CheckoutRef; email: string; productName: string; amount: number; currency: CurrencyCode;
    successUrl: string; cancelUrl: string; now: Date;
  }): Promise<CreatedCheckout> {
    const md = metadata('event_ticket', args.ref);
    const session = await this.stripe.checkout.sessions.create(
      {
        mode: 'payment',
        customer_email: args.email,
        client_reference_id: args.ref.participantId,
        line_items: [{
          quantity: 1,
          price_data: {
            currency: args.currency,
            unit_amount: toStripeAmount(args.amount, args.currency),
            product_data: { name: args.productName },
          },
        }],
        expires_at: Math.floor((args.now.getTime() + CHECKOUT_TTL_MS) / 1000),
        metadata: md,
        payment_intent_data: { metadata: md },
        success_url: args.successUrl,
        cancel_url: args.cancelUrl,
      },
      { idempotencyKey: `checkout:ticket:${args.ref.participantId}:${args.ref.attempt}` },
    );
    return toCreated(session);
  }

  /** hold_now: the customer authorizes the deposit in Checkout; nothing is charged. */
  async createHoldCheckout(args: {
    ref: CheckoutRef; customerId: string; productName: string; amount: number; currency: CurrencyCode;
    successUrl: string; cancelUrl: string; now: Date;
  }): Promise<CreatedCheckout> {
    const md = metadata('event_deposit_hold', args.ref);
    const session = await this.stripe.checkout.sessions.create(
      {
        mode: 'payment',
        customer: args.customerId,
        client_reference_id: args.ref.participantId,
        // Cards only: most wallets/bank methods cannot be authorized now and captured later.
        payment_method_types: ['card'],
        line_items: [{
          quantity: 1,
          price_data: {
            currency: args.currency,
            unit_amount: toStripeAmount(args.amount, args.currency),
            product_data: { name: args.productName },
          },
        }],
        payment_intent_data: {
          capture_method: 'manual',
          setup_future_usage: 'off_session', // lets the tick renew the hold for multi-day events
          metadata: md,
        },
        expires_at: Math.floor((args.now.getTime() + CHECKOUT_TTL_MS) / 1000),
        metadata: md,
        success_url: args.successUrl,
        cancel_url: args.cancelUrl,
      },
      { idempotencyKey: `checkout:hold:${args.ref.participantId}:${args.ref.attempt}` },
    );
    return toCreated(session);
  }

  /** save_card: collect a card for an off-session hold placed days later. */
  async createSetupCheckout(args: {
    ref: CheckoutRef; customerId: string; successUrl: string; cancelUrl: string; now: Date;
  }): Promise<CreatedCheckout> {
    const md = metadata('event_deposit_setup', args.ref);
    const session = await this.stripe.checkout.sessions.create(
      {
        mode: 'setup',
        customer: args.customerId,
        client_reference_id: args.ref.participantId,
        payment_method_types: ['card'], // with an explicit list, setup mode needs no currency
        setup_intent_data: { metadata: md },
        expires_at: Math.floor((args.now.getTime() + CHECKOUT_TTL_MS) / 1000),
        metadata: md,
        success_url: args.successUrl,
        cancel_url: args.cancelUrl,
      },
      { idempotencyKey: `checkout:setup:${args.ref.participantId}:${args.ref.attempt}` },
    );
    return toCreated(session);
  }

  async placeOffSessionHold(args: {
    customerId: string; paymentMethodId: string; amount: number; currency: CurrencyCode;
    ref: CheckoutRef; idempotencyKey: string;
  }): Promise<OffSessionHoldResult> {
    try {
      const pi = await this.stripe.paymentIntents.create(
        {
          amount: toStripeAmount(args.amount, args.currency),
          currency: args.currency,
          customer: args.customerId,
          payment_method: args.paymentMethodId,
          capture_method: 'manual',
          off_session: true,
          confirm: true,
          metadata: metadata('event_deposit_hold', args.ref),
          expand: ['latest_charge'],
        },
        { idempotencyKey: args.idempotencyKey },
      );
      if (pi.status === 'requires_capture') {
        return { status: 'held', paymentIntentId: pi.id, captureBefore: captureBeforeOf(pi), paymentMethodId: args.paymentMethodId };
      }
      if (pi.status === 'requires_action') {
        return { status: 'requires_action', paymentIntentId: pi.id, error: 'authentication_required' };
      }
      return { status: 'declined', error: `unexpected_status:${pi.status}` };
    } catch (err) {
      if (err instanceof Stripe.errors.StripeCardError) {
        const piId = err.payment_intent?.id;
        if (err.code === 'authentication_required' && piId) {
          return { status: 'requires_action', paymentIntentId: piId, error: err.code };
        }
        return { status: 'declined', error: err.decline_code ?? err.code ?? 'card_declined' };
      }
      throw err; // network / API errors: let the tick retry
    }
  }

  async retrievePaymentIntent(paymentIntentId: string): Promise<Stripe.PaymentIntent> {
    return this.stripe.paymentIntents.retrieve(paymentIntentId, { expand: ['latest_charge'] });
  }

  /** Release a hold. Safe to repeat: an already-cancelled intent counts as success. */
  async cancelIntent(paymentIntentId: string): Promise<'canceled' | 'already_captured'> {
    const pi = await this.stripe.paymentIntents.retrieve(paymentIntentId);
    if (pi.status === 'canceled') return 'canceled';
    if (pi.status === 'succeeded') return 'already_captured';
    await this.stripe.paymentIntents.cancel(paymentIntentId, {}, { idempotencyKey: `cancel:${paymentIntentId}` });
    return 'canceled';
  }

  /** Capture part or all of a hold. Returns the stored amount actually captured. */
  async captureIntent(paymentIntentId: string, amount: number, currency: CurrencyCode): Promise<number | 'expired'> {
    const pi = await this.stripe.paymentIntents.retrieve(paymentIntentId);
    if (pi.status === 'succeeded') return fromStripeAmount(pi.amount_received, currency);
    if (pi.status === 'canceled') return 'expired'; // authorization lapsed — nothing to take
    const captured = await this.stripe.paymentIntents.capture(
      paymentIntentId,
      { amount_to_capture: toStripeAmount(amount, currency) },
      { idempotencyKey: `capture:${paymentIntentId}` },
    );
    return fromStripeAmount(captured.amount_received, currency);
  }

  async refund(paymentIntentId: string, amount: number, currency: CurrencyCode, participantId: string): Promise<'succeeded' | 'pending'> {
    const r = await this.stripe.refunds.create(
      { payment_intent: paymentIntentId, amount: toStripeAmount(amount, currency), metadata: { participantId } },
      { idempotencyKey: `refund:${participantId}` },
    );
    return r.status === 'succeeded' ? 'succeeded' : 'pending';
  }

  async expireCheckout(sessionId: string): Promise<void> {
    const s = await this.stripe.checkout.sessions.retrieve(sessionId);
    if (s.status === 'open') await this.stripe.checkout.sessions.expire(sessionId);
  }

  async retrieveCheckout(sessionId: string): Promise<Stripe.Checkout.Session> {
    return this.stripe.checkout.sessions.retrieve(sessionId);
  }

  async retrieveSetupIntent(setupIntentId: string): Promise<Stripe.SetupIntent> {
    return this.stripe.setupIntents.retrieve(setupIntentId);
  }
}
```

## Test-mode walkthrough

Use a Stripe sandbox and the Stripe CLI:

```bash
stripe listen --forward-to localhost:3000/api/stripe/webhook   # prints whsec_… → STRIPE_WEBHOOK_SECRET
```

| # | Do | Card | Expect |
|---|---|---|---|
| 1 | Register 2 tickets for a paid event, pay | `4242 4242 4242 4242` | participant `CONFIRMED`, `seatsTaken` +2 |
| 2 | Register, open Checkout, pay with the decline card, then retry with the good card | `4000 0000 0000 9995`, then `4242…` | still `PENDING` after the decline, `CONFIRMED` after the retry |
| 3 | Register, close the Checkout tab, wait 35 min (or lower `expires_at` in dev) and run the tick | — | `EXPIRED`, seat released, `expired` email |
| 4 | Free event with deposit, event in 2 days: register, authorize | `4242…` | Dashboard shows an **uncaptured** payment; participant `HELD` |
| 5 | Staff check-in for all tickets | — | Dashboard payment **canceled**; participant `RELEASED` |
| 6 | Repeat 4, move the clock (or event) so end + 2 h has passed, run the tick | — | Dashboard payment **captured**; `CAPTURED`, `NO_SHOW` |
| 7 | Free event with deposit, event in 20 days: register | `4242…` | Checkout in setup mode; `CARD_SAVED`, `nextAction: PLACE_HOLD` |
| 8 | Repeat 7 with a card that fails off-session, run the tick at `holdDueAt` | `4000 0027 6000 3184` (always authenticate) | `HOLD_REQUIRES_ACTION`, `hold_action_required` email |
| 9 | Repeat 7 with a card that attaches but cannot be charged | `4000 0000 0000 0341` | `HOLD_FAILED`; at `holdCutoffAt` the seat is released |
| 10 | Refund a confirmed ticket in the Dashboard | — | `charge.refunded` → `REFUNDED`, seat released |
| 11 | `stripe events resend <evt_id>` for any processed event | — | handler returns `duplicate`; nothing changes |

`4000 0025 0000 3155` authenticates in Checkout and then succeeds off-session —
the happy path for step 7's hold placement.

## Checklist

- [ ] `STRIPE_WEBHOOK_SECRET` set per environment; the route verifies the raw body.
- [ ] Endpoint subscribed to the eight handled event types.
- [ ] Card payments enabled in the Dashboard for every offered currency.
- [ ] Walkthrough steps 1–11 pass in test mode before going live.
