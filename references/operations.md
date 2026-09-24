# Operations

What must run, what must be configured, and what an operator needs to see.

## Environment

| Variable | Used by | Notes |
|---|---|---|
| `STRIPE_SECRET_KEY` | gateway | `sk_test_…` in preview, `sk_live_…` in production only |
| `STRIPE_WEBHOOK_SECRET` | webhook route | One per endpoint per environment (`whsec_…`) |
| `EVENTS_MANAGE_TOKEN_SECRET` | guest manage links | ≥ 32 random bytes (`openssl rand -base64 48`); rotating it breaks outstanding links |
| `CRON_SECRET` | tick route | Required — the route refuses every call when unset |
| `APP_BASE_URL` | Checkout return URLs, email links | The public origin, no trailing slash |

## Background work

One job: `GET /api/cron/events-tick` every **5 minutes**, with
`Authorization: Bearer $CRON_SECRET`.

```ts
// vercel.ts (or the same `crons` array in vercel.json)
import type { VercelConfig } from '@vercel/config/v1';

export const config: VercelConfig = {
  crons: [{ path: '/api/cron/events-tick', schedule: '*/5 * * * *' }],
};
```

Vercel Cron sends `Authorization: Bearer $CRON_SECRET` automatically when the
variable is set on the project.

On other platforms: any scheduler that can send the header (Cloud Scheduler,
GitHub Actions, a systemd timer). The tick processes up to 100 due participants
per run, oldest first; a backlog drains over successive runs.

| Tick action | Happens when |
|---|---|
| `EXPIRE_PENDING` | Checkout abandoned: reconciles with Stripe (recovers a lost `completed` webhook), then releases the seat |
| `PLACE_HOLD` | Saved card reaches `holdDueAt`; or a failed hold reaches `holdCutoffAt` → seat released |
| `SETTLE` | `HELD` deposit reaches `settleDueAt` → capture no-shows / release arrivals; renew if the hold would lapse first |
| `RETRY_RELEASE / CAPTURE / REFUND` | A Stripe call after a transition failed; retried every 10 min with the same idempotency key |

The response is a `TickReport` (`processed`, `failed[]`). Alert when `failed`
is non-empty on two consecutive runs — one failure is usually a Stripe blip,
two is a stuck participant.

## What operators need to see

| Question | Answer from |
|---|---|
| Who is coming, who paid, who holds a deposit? | Staff participant list (payment, amount + currency, deposit, attendance) |
| Which deposits will be charged tonight, and when? | Participants with deposit `HELD` and `settleDueAt`; header of the check-in screen |
| Whose hold failed? | deposit `HOLD_FAILED` / `HOLD_REQUIRES_ACTION` with `lastError` and `holdCutoffAt` |
| What is stuck? | `nextAction LIKE 'RETRY_%'` with `nextActionAt` more than 30 min in the past |
| Who cancelled, when, and did they get money back? | `cancelledAt`, `cancelledBy`, `payment.status`, `refundedAmount` |
| Who waived a deposit and why? | `deposit.waivedBy`, `deposit.waiveReason` |
| Did the webhook keep up? | `stripeWebhookEvents` with `claimedAt` but no `doneAt` for over 10 min |

## Reconciliation

Once a day (or on demand), compare the module with Stripe:

| Check | Query | Fix |
|---|---|---|
| Uncaptured holds with no `HELD` participant | Stripe: PaymentIntents `status=requires_capture`, `metadata.kind=event_deposit_hold` | Cancel in Stripe, or re-run the participant's `SETTLE` |
| `CONFIRMED` tickets with a refunded charge | Stripe: charges `refunded=true` for `event_ticket` intents | Replay `charge.refunded` (`stripe events resend`) |
| `CAPTURED` deposits whose intent is not `succeeded` | participants vs PaymentIntents | Tick `RETRY_CAPTURE`; if the intent is `canceled`, the hold lapsed — record `RELEASED` |

Stripe Sigma or `stripe payment_intents list --limit 100` with a metadata
filter in a script is enough at venue scale.

## Emails (host notifier)

Every `NotificationKind` needs a template in every locale, with the manage link:

| Kind | Must say |
|---|---|
| `registered` | Event, date, local time and zone, tickets, how to cancel |
| `deposit_held` | + deposit amount, "released when you check in", capture rules and deadline |
| `card_saved` | + "hold of {amount} placed on {holdDate}" |
| `hold_action_required` | Bank asks for confirmation — "Confirm now" link (resume), seat released on {cutoff} |
| `hold_failed` | Card declined — "Use another card" link (resume), seat released on {cutoff} |
| `seat_released_no_hold` | Seat released because no deposit could be held |
| `deposit_released` | Hold removed; banks can take a few days to show it |
| `deposit_captured` | Amount kept, why (no-show / late cancel / partial arrival), how to contact the venue |
| `cancelled`, `refunded`, `expired` | Outcome and amount |

## Go-live checklist

- [ ] All five env vars set in production; test keys nowhere in production.
- [ ] Webhook endpoint created in live mode with the eight event types ([stripe.md](stripe.md)).
- [ ] Tick scheduled and returning 200 with the secret, 401 without.
- [ ] Indexes / migration applied ([firestore.md](firestore.md) / [postgres.md](postgres.md)).
- [ ] Email templates for every kind, every locale.
- [ ] Staff trained on the check-in screen; waiver reasons agreed.
- [ ] Deposit policy text reviewed against local consumer law and card-network rules for the venue's country.
