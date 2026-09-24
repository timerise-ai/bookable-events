# API routes

Next.js App Router route handlers. Every handler is thin: parse, authorize,
call the engine, map errors. The host's auth and tenant model enter through one
file, `lib/events/host.ts`.

## Surface

| Method + path | Who | Does |
|---|---|---|
| `POST /api/events/{eventId}/register` | anyone (session optional) | register; returns `participantId`, `manageToken`, `checkoutUrl?` |
| `GET /api/events/{eventId}/participants/{id}/status` | owner or token | status for the "you're registered" page |
| `POST /api/events/{eventId}/participants/{id}/cancel` | owner or token | customer cancel ([registration.md](registration.md)) |
| `POST /api/events/{eventId}/participants/{id}/resume` | owner or token | new Checkout for PENDING / failed hold |
| `POST /api/stripe/webhook` | Stripe (signed) | [stripe.md](stripe.md) |
| `GET /api/staff/events/{eventId}/participants` | staff, same location | check-in list |
| `POST /api/staff/events/{eventId}/participants/{id}/check-in` `{ count }` | staff, same location | arrival; releases the hold when everyone arrived |
| `POST .../{id}/no-show` | staff, same location | mark no-show (settlement still decides the capture) |
| `POST .../{id}/waive` `{ reason }` | staff, same location | release the hold, record who and why |
| `POST .../{id}/cancel` `{ refund }` | staff, same location | cancel with or without refund |
| `POST /api/admin/events` | admin, target location | create |
| `PUT /api/admin/events/{eventId}` | admin, same location | update (whitelist, capacity guard) |
| `POST /api/admin/events/{eventId}/cancel` | admin, same location | cancel the event, refund everyone |
| `GET /api/cron/events-tick` | cron (`CRON_SECRET`) | due work ([no-show-deposits.md](no-show-deposits.md)) |

The public event list and detail routes are ordinary reads and are left to the
host's pattern; filter by `status = 'published'`, `deletedAt = null`, the
caller's allowed locations, and `date >= todayIn(event.timeZone)`.

**Error envelope** everywhere: `{ ok: true, data }` or
`{ ok: false, error: <code>, field? }`. Codes are stable strings the UI maps to
i18n keys (`events.errors.<code>`).

| Code | Status |
|---|---|
| `event_not_found`, `not_found` | 404 |
| `registration_closed`, `event_started`, `no_spots`, `already_arrived`, `illegal_transition`, `late_payment`, `below_seats_taken`, `use_cancel_endpoint` | 409 |
| `currency_invalid`, `currency_not_offered`, `currency_not_enabled`, `bad_count`, field codes | 400 |
| `login_required`, `unauthorized` | 401 |
| `insufficient_balance` | 402 |
| `forbidden`, `location_forbidden` | 403 |
| `payment_unavailable` | 502 |

## Authorization rules

| Rule | Defect it prevents |
|---|---|
| **Every** `[eventId]` staff/admin route calls `requireStaffForEvent`, which checks the event's `locationId` against the staff member's locations | Collection routes get the tenant check and `[id]` routes forget it, so staff of one venue reading or editing another's events. |
| Missing and foreign events both answer 404 | Probing ids must not reveal other tenants' events. |
| Participant routes accept the session owner **or** the HMAC manage token | Guests can cancel; nobody can act on someone else's seat by guessing ids. |
| The admin create route checks `canAccessLocation` on the **body's** `locationId` | An admin of location A creating events under location B. |
| Cron fails **closed** when `CRON_SECRET` is unset | `if (secret && header !== ...)` is open by default. |
| The webhook verifies the signature on the raw text body | Parsing JSON first changes the bytes and every signature fails, or worse, someone "fixes" it by skipping verification. |

## Host seam

`host.ts` is the only file that knows the host's auth, email, database client
and URLs. Replace each `HOST:` body; do not scatter these calls through the
routes.

```ts
// lib/events/host.ts: THE seam file. Replace each body with the host's own mechanism;
// nothing else in lib/events imports the host.
import Stripe from 'stripe';
import { getFirestore } from 'firebase-admin/firestore';
import { createFirestoreEventStore } from './backends/firestore-store';
import { createEventsEngine, type EventsEngine } from './engine';
import type { Notifier } from './ports';
import { StripeGateway } from './stripe-gateway';
import { createTick } from './tick';
import { createWebhookHandler, type WebhookHandler } from './webhook';
import type { EventRecord } from './types';

// --- Auth -----------------------------------------------------------------

export interface CustomerPrincipal { id: string; email: string }
export interface StaffPrincipal { id: string; role: 'admin' | 'staff'; locationIds: string[] | 'all' }

/** The signed-in customer, verified server-side (session cookie / bearer token). null = guest. */
export async function getCustomer(_req: Request): Promise<CustomerPrincipal | null> {
  throw new Error('HOST: return the verified customer from your auth (Clerk, NextAuth, Supabase, Firebase...)');
}

/** The signed-in staff member, or null. Role and location list come from your staff store. */
export async function getStaff(_req: Request): Promise<StaffPrincipal | null> {
  throw new Error('HOST: return the verified staff member from your auth');
}

export function canAccessLocation(staff: StaffPrincipal, locationId: string): boolean {
  return staff.locationIds === 'all' || staff.locationIds.includes(locationId);
}

// --- Notifications --------------------------------------------------------

const notifier: Notifier = {
  async send(kind, participant, _event, links) {
    // HOST: render `events.email.<kind>` in participant.locale and send via your provider.
    console.info(`[events] notify ${kind} to ${participant.email} (${links.manage})`);
  },
};

// --- Runtime --------------------------------------------------------------

let runtime: { engine: EventsEngine; webhook: WebhookHandler; tick: ReturnType<typeof createTick> } | null = null;

function required(name: string): string {
  const v = process.env[name];
  if (!v) throw new Error(`${name} is not set`);
  return v;
}

export function getEventsRuntime() {
  if (runtime) return runtime;
  const base = required('APP_BASE_URL').replace(/\/$/, '');
  const engine = createEventsEngine({
    store: createFirestoreEventStore(getFirestore()), // or createPostgresEventStore(sql, randomUUID)
    gateway: new StripeGateway(new Stripe(required('STRIPE_SECRET_KEY'))),
    notifier,
    ledger: null, // HOST: a BalanceLedger if customers hold stored credit
    manageTokenSecret: required('EVENTS_MANAGE_TOKEN_SECRET'),
    urls: {
      success: (eventId, participantId) => `${base}/events/${eventId}/registered?p=${participantId}`,
      cancel: (eventId) => `${base}/events/${eventId}`,
      manage: (eventId, participantId, token) => `${base}/events/${eventId}/manage/${participantId}?t=${token}`,
    },
    resolveName: (event: EventRecord, locale: string) =>
      typeof event.name === 'string' ? event.name : event.name[locale] ?? Object.values(event.name)[0] ?? 'Event',
  });
  const webhook = createWebhookHandler(engine);
  runtime = { engine, webhook, tick: createTick(engine, webhook) };
  return runtime;
}
```

## Shared helpers

```ts
// lib/events/http.ts: one error envelope for every events route: { ok, data } | { ok:false, error, field? }
import { NextResponse } from 'next/server';
import { TransitionError } from './participant-machine';
import { RegistrationError } from './ports';
import { canAccessLocation, getCustomer, getEventsRuntime, getStaff, type StaffPrincipal } from './host';
import type { EventRecord, Participant } from './types';

export const ok = <T>(data: T, status = 200) => NextResponse.json({ ok: true, data }, { status });
export const fail = (error: string, status: number, field?: string) =>
  NextResponse.json({ ok: false, error, ...(field ? { field } : {}) }, { status });

export function toErrorResponse(err: unknown): NextResponse {
  if (err instanceof RegistrationError) return fail(err.code, err.status);
  if (err instanceof TransitionError) return fail(err.code, 409);
  if (err instanceof HttpError) return fail(err.code, err.status);
  console.error('[events] unhandled', err);
  return fail('internal_error', 500);
}

export class HttpError extends Error {
  constructor(public readonly code: string, public readonly status: number) {
    super(code);
  }
}

/** Staff guard WITH tenant scope. Every [eventId] staff/admin route calls this, not just the list. */
export async function requireStaffForEvent(req: Request, eventId: string, role: 'admin' | 'staff' = 'staff'): Promise<{ staff: StaffPrincipal; event: EventRecord }> {
  const staff = await getStaff(req);
  if (!staff) throw new HttpError('unauthorized', 401);
  if (role === 'admin' && staff.role !== 'admin') throw new HttpError('forbidden', 403);
  const event = await getEventsRuntime().engine.deps.store.getEvent(eventId);
  // Same 404 for "missing" and "other location": do not reveal other tenants' events.
  if (!event || !canAccessLocation(staff, event.locationId)) throw new HttpError('event_not_found', 404);
  return { staff, event };
}

/** Customer owns the seat by session, or a guest holds the emailed manage token. */
export async function requireParticipantAccess(req: Request, eventId: string, participantId: string): Promise<Participant> {
  const { engine } = getEventsRuntime();
  const p = await engine.deps.store.getParticipant(eventId, participantId);
  if (!p) throw new HttpError('not_found', 404);
  const token = new URL(req.url).searchParams.get('t') ?? req.headers.get('x-manage-token');
  if (token && engine.verifyManageToken(participantId, token)) return p;
  const customer = await getCustomer(req);
  if (customer && p.customerId === customer.id) return p;
  throw new HttpError('not_found', 404);
}

/** Client-safe view: no Stripe ids, no customer ids. */
export function publicParticipant(p: Participant) {
  return {
    id: p.id, eventId: p.eventId, fullName: p.fullName, ticketCount: p.ticketCount, currency: p.currency,
    payment: { status: p.payment.status, method: p.payment.method, amount: p.payment.amount, refundedAmount: p.payment.refundedAmount },
    deposit: { status: p.deposit.status, amount: p.deposit.amount, captureAmount: p.deposit.captureAmount, holdCutoffAt: p.deposit.holdCutoffAt },
    attendance: { status: p.attendance.status, arrivedCount: p.attendance.arrivedCount },
    registeredAt: p.registeredAt, cancelledAt: p.cancelledAt,
  };
}
```

## Routes

```ts
// app/api/events/[eventId]/register/route.ts
import { getCustomer, getEventsRuntime } from '@/lib/events/host';
import { fail, ok, toErrorResponse } from '@/lib/events/http';
import { parseRegistrationBody } from '@/lib/events/input';

export async function POST(req: Request, { params }: { params: Promise<{ eventId: string }> }) {
  try {
    const { eventId } = await params;
    const parsed = parseRegistrationBody(await req.json().catch(() => null));
    if (!parsed.ok) return fail(parsed.code, 400, parsed.field);

    // Identity comes from the session only. A `playerId` / `customerId` in the body is ignored.
    const customer = await getCustomer(req);
    const result = await getEventsRuntime().engine.register({
      eventId,
      customerId: customer?.id ?? null,
      ...parsed.value,
    });
    return ok(result, 201);
  } catch (err) {
    return toErrorResponse(err);
  }
}
```

```ts
// app/api/events/[eventId]/participants/[participantId]/[action]/route.ts
// Customer / guest self-service: GET status, POST cancel | resume.
import { getEventsRuntime } from '@/lib/events/host';
import { fail, ok, publicParticipant, requireParticipantAccess, toErrorResponse } from '@/lib/events/http';

type Params = { params: Promise<{ eventId: string; participantId: string; action: string }> };

export async function GET(req: Request, { params }: Params) {
  try {
    const { eventId, participantId, action } = await params;
    if (action !== 'status') return fail('not_found', 404);
    return ok(publicParticipant(await requireParticipantAccess(req, eventId, participantId)));
  } catch (err) {
    return toErrorResponse(err);
  }
}

export async function POST(req: Request, { params }: Params) {
  try {
    const { eventId, participantId, action } = await params;
    await requireParticipantAccess(req, eventId, participantId);
    const { engine } = getEventsRuntime();
    if (action === 'cancel') {
      const res = await engine.cancelByCustomer(eventId, participantId);
      return res ? ok(publicParticipant(res.after)) : fail('not_found', 404);
    }
    if (action === 'resume') {
      return ok({ checkoutUrl: await engine.resumeCheckout(eventId, participantId) });
    }
    return fail('not_found', 404);
  } catch (err) {
    return toErrorResponse(err);
  }
}
```

```ts
// app/api/stripe/webhook/route.ts
// If the host already has a Stripe webhook, call `webhook.handle(event)` from it instead;
// the handler ignores anything without `metadata.kind` starting with `event_`.
import Stripe from 'stripe';
import { getEventsRuntime } from '@/lib/events/host';

export const runtime = 'nodejs';

export async function POST(req: Request) {
  const signature = req.headers.get('stripe-signature');
  const secret = process.env.STRIPE_WEBHOOK_SECRET;
  if (!signature || !secret) return new Response('missing signature', { status: 400 });

  let event: Stripe.Event;
  try {
    // Raw text body: the signature covers the exact bytes, so never JSON.parse first.
    event = Stripe.webhooks.constructEvent(await req.text(), signature, secret);
  } catch {
    return new Response('invalid signature', { status: 400 });
  }

  try {
    const result = await getEventsRuntime().webhook.handle(event);
    return Response.json({ received: true, result });
  } catch (err) {
    console.error(`[events] webhook ${event.type} ${event.id} failed`, err);
    return new Response('handler failed', { status: 500 }); // Stripe retries with backoff
  }
}
```

```ts
// app/api/staff/events/[eventId]/participants/route.ts: the check-in list.
import { getEventsRuntime } from '@/lib/events/host';
import { ok, requireStaffForEvent, toErrorResponse } from '@/lib/events/http';

export async function GET(req: Request, { params }: { params: Promise<{ eventId: string }> }) {
  try {
    const { eventId } = await params;
    const { event } = await requireStaffForEvent(req, eventId);
    const participants = await getEventsRuntime().engine.deps.store.listParticipants(eventId);
    return ok({
      event: { id: event.id, capacity: event.capacity, seatsTaken: event.seatsTaken },
      participants: participants.map((p) => ({
        id: p.id, fullName: p.fullName, email: p.email, phone: p.phone, ticketCount: p.ticketCount,
        currency: p.currency, payment: p.payment.status, paid: p.payment.amount, method: p.payment.method,
        deposit: p.deposit.status, depositAmount: p.deposit.amount, captureBefore: p.deposit.captureBefore,
        depositError: p.deposit.lastError, attendance: p.attendance, registeredAt: p.registeredAt,
        cancelledAt: p.cancelledAt, cancelledBy: p.cancelledBy,
      })),
    });
  } catch (err) {
    return toErrorResponse(err);
  }
}
```

```ts
// app/api/staff/events/[eventId]/participants/[participantId]/[action]/route.ts
// POST check-in {count} | no-show | waive {reason} | cancel {refund: 'full'|'none'}
import { getEventsRuntime } from '@/lib/events/host';
import { fail, ok, publicParticipant, requireStaffForEvent, toErrorResponse } from '@/lib/events/http';

type Params = { params: Promise<{ eventId: string; participantId: string; action: string }> };

export async function POST(req: Request, { params }: Params) {
  try {
    const { eventId, participantId, action } = await params;
    const { staff } = await requireStaffForEvent(req, eventId);
    const body = (await req.json().catch(() => ({}))) as Record<string, unknown>;
    const ops = getEventsRuntime().engine.staff;

    let res;
    switch (action) {
      case 'check-in': {
        const count = typeof body.count === 'number' ? body.count : Number.NaN;
        res = await ops.checkIn(eventId, participantId, count, staff.id);
        break;
      }
      case 'no-show':
        res = await ops.markNoShow(eventId, participantId, staff.id);
        break;
      case 'waive': {
        const reason = typeof body.reason === 'string' ? body.reason.trim().slice(0, 500) : '';
        if (!reason) return fail('reason_required', 400, 'reason');
        res = await ops.waive(eventId, participantId, staff.id, reason);
        break;
      }
      case 'cancel':
        res = await ops.cancel(eventId, participantId, 'staff', body.refund === 'none' ? 'none' : 'full');
        break;
      default:
        return fail('not_found', 404);
    }
    return res ? ok(publicParticipant(res.after)) : fail('not_found', 404);
  } catch (err) {
    return toErrorResponse(err);
  }
}
```

```ts
// app/api/admin/events/route.ts: POST create. Location must be one the admin manages.
import { randomUUID } from 'node:crypto';
import { canAccessLocation, getEventsRuntime, getStaff } from '@/lib/events/host';
import { fail, ok, toErrorResponse } from '@/lib/events/http';
import { parseEventInput } from '@/lib/events/input';

const ALLOWED_CURRENCIES: string[] = []; // HOST: deployment restriction; empty = any currency

export async function POST(req: Request) {
  try {
    const staff = await getStaff(req);
    if (!staff || staff.role !== 'admin') return fail('forbidden', 403);
    const body = (await req.json().catch(() => null)) as Record<string, unknown> | null;
    const locationId = typeof body?.locationId === 'string' ? body.locationId : '';
    if (!locationId || !canAccessLocation(staff, locationId)) return fail('location_forbidden', 403, 'locationId');

    const parsed = parseEventInput(body, { seatsTaken: 0, allowedCurrencies: ALLOWED_CURRENCIES });
    if (!parsed.ok) return fail(parsed.code, 400, parsed.field);

    const { engine } = getEventsRuntime();
    const now = engine.now();
    const id = randomUUID();
    await engine.deps.store.createEvent({
      ...parsed.value, id, locationId, seatsTaken: 0, deletedAt: null, createdAt: now, updatedAt: now,
    });
    return ok({ id }, 201);
  } catch (err) {
    return toErrorResponse(err);
  }
}
```

```ts
// app/api/admin/events/[eventId]/route.ts: PUT update (whitelisted body, capacity guard).
import { getEventsRuntime } from '@/lib/events/host';
import { fail, ok, requireStaffForEvent, toErrorResponse } from '@/lib/events/http';
import { parseEventInput } from '@/lib/events/input';

const ALLOWED_CURRENCIES: string[] = []; // HOST: same restriction as the create route

export async function PUT(req: Request, { params }: { params: Promise<{ eventId: string }> }) {
  try {
    const { eventId } = await params;
    const { event } = await requireStaffForEvent(req, eventId, 'admin');
    const parsed = parseEventInput(await req.json().catch(() => null), {
      seatsTaken: event.seatsTaken, allowedCurrencies: ALLOWED_CURRENCIES,
    });
    if (!parsed.ok) return fail(parsed.code, 400, parsed.field);
    // Cancelling goes through /cancel so participants are refunded, never a bare status flip.
    if (parsed.value.status === 'cancelled' && event.status !== 'cancelled') return fail('use_cancel_endpoint', 409, 'status');
    // Prices and deposits of an event with registrations are frozen per participant anyway
    // (payment.amount / deposit.amount), so editing them only affects new registrations.

    const { engine } = getEventsRuntime();
    const r = await engine.deps.store.updateEvent(eventId, parsed.value, engine.now());
    if (r === 'not_found') return fail('event_not_found', 404);
    if (r === 'below_seats_taken') return fail('below_seats_taken', 409, 'capacity');
    return ok({ id: eventId });
  } catch (err) {
    return toErrorResponse(err);
  }
}
```

```ts
// app/api/admin/events/[eventId]/cancel/route.ts: cancel the event, refund and release everyone.
import { getEventsRuntime } from '@/lib/events/host';
import { ok, requireStaffForEvent, toErrorResponse } from '@/lib/events/http';

export async function POST(req: Request, { params }: { params: Promise<{ eventId: string }> }) {
  try {
    const { eventId } = await params;
    await requireStaffForEvent(req, eventId, 'admin');
    const { engine } = getEventsRuntime();
    // 1. Close registration first, so nobody takes a seat while we are refunding.
    await engine.deps.store.setEventStatus(eventId, 'cancelled', engine.now());
    // 2. Every seat-holding participant: full refund, holds released. Re-runnable if it times out.
    const result = await engine.staff.cancelEvent(eventId, 'admin');
    return ok(result);
  } catch (err) {
    return toErrorResponse(err);
  }
}
```

```ts
// app/api/cron/events-tick/route.ts: schedule every 5 minutes.
import { timingSafeEqual } from 'node:crypto';
import { getEventsRuntime } from '@/lib/events/host';

export const runtime = 'nodejs';
export const maxDuration = 300;

function authorized(req: Request): boolean {
  const secret = process.env.CRON_SECRET;
  if (!secret) return false; // fail closed: an unset secret must not open the endpoint
  const given = Buffer.from(req.headers.get('authorization') ?? '');
  const expected = Buffer.from(`Bearer ${secret}`);
  return given.length === expected.length && timingSafeEqual(given, expected);
}

export async function GET(req: Request) {
  if (!authorized(req)) return new Response('unauthorized', { status: 401 });
  const report = await getEventsRuntime().tick(100);
  if (report.failed.length > 0) console.error('[events] tick failures', report.failed);
  return Response.json(report);
}
```

## Checklist

- [ ] `grep -rn "requireStaffForEvent" app/api/staff app/api/admin` lists every `[eventId]` route.
- [ ] No route reads `customerId`, `playerId`, `seatsTaken` or prices from a request body.
- [ ] `STRIPE_WEBHOOK_SECRET`, `CRON_SECRET`, `EVENTS_MANAGE_TOKEN_SECRET` are set in every environment.
- [ ] If the host already has a Stripe webhook, it calls `webhook.handle(event)` instead of adding a second endpoint.
