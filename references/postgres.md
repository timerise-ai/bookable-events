# Postgres store

Reference `EventStore` on Postgres (Neon, Supabase, RDS, plain Postgres). It
talks to a two-method `SqlClient`, so it sits on `pg`, `postgres.js` or a
pooler without changes. A host that standardised on Drizzle or Prisma ports the
same queries to its ORM: the transaction boundaries and locks are what matter.

## Schema

```sql
CREATE TABLE events (
  id                    text PRIMARY KEY,
  location_id           text NOT NULL,
  type_id               text,
  name                  jsonb NOT NULL,            -- string or { locale: text }
  description           jsonb NOT NULL DEFAULT '""',
  image_url             text,
  date                  date NOT NULL,             -- local to time_zone
  time_start            text NOT NULL CHECK (time_start ~ '^([01][0-9]|2[0-3]):[0-5][0-9]$'),
  time_end              text NOT NULL CHECK (time_end   ~ '^([01][0-9]|2[0-3]):[0-5][0-9]$'),
  time_zone             text NOT NULL,             -- IANA
  capacity              integer NOT NULL CHECK (capacity > 0),
  seats_taken           integer NOT NULL DEFAULT 0 CHECK (seats_taken >= 0 AND seats_taken <= capacity),
  ticket_price          jsonb NOT NULL DEFAULT '{}',   -- { "usd": 2500 }
  deposit               jsonb,                         -- { "amountPerTicket": { "usd": 2000 } }
  cancel_deadline_hours integer NOT NULL DEFAULT 24,
  status                text NOT NULL CHECK (status IN ('draft','published','cancelled','completed')),
  deleted_at            timestamptz,
  created_at            timestamptz NOT NULL,
  updated_at            timestamptz NOT NULL
);
CREATE INDEX events_upcoming ON events (location_id, status, date) WHERE deleted_at IS NULL;

CREATE TABLE event_participants (
  id             text PRIMARY KEY,
  event_id       text NOT NULL REFERENCES events(id),
  location_id    text NOT NULL,
  customer_id    text,
  email          text NOT NULL,
  payment_status text NOT NULL,
  deposit_status text NOT NULL,
  next_action_at timestamptz,
  registered_at  timestamptz NOT NULL,
  data           jsonb NOT NULL                      -- the whole Participant
);
CREATE INDEX event_participants_event    ON event_participants (event_id, registered_at);
CREATE INDEX event_participants_due      ON event_participants (next_action_at) WHERE next_action_at IS NOT NULL;
CREATE INDEX event_participants_customer ON event_participants (customer_id, registered_at DESC);
CREATE INDEX event_participants_location ON event_participants (location_id, payment_status);

CREATE TABLE stripe_webhook_events (
  id         text PRIMARY KEY,                       -- evt_...
  claimed_at timestamptz NOT NULL,
  done_at    timestamptz
);
```

| Choice | Reason |
|---|---|
| `CHECK (seats_taken <= capacity)` | A second line of defence: a bug that over-sells fails the transaction instead of the venue. |
| Participant as `jsonb` + indexed copies | The state machine reads and writes the participant whole; the copied columns exist only for the queries (due work, lists, profiles, reporting). `upsertParticipant` writes both in one statement so they cannot drift. |
| `SELECT ... FOR UPDATE` on the event row in both transactions | Serializes registrations and seat-changing transitions per event. Lock order is always event, then participant, so the two paths cannot deadlock. |
| Partial index on `next_action_at` | Most participants are terminal (`null`); the tick's index stays small. |

Locking the event row on every participant write serializes one event's
updates. At door-check-in rates (a few per second) that is invisible; a host
with thousands of concurrent writers to one event can drop the event lock in
`mutateParticipant` when `seatDelta` is 0, keeping it for seat changes.

## Row-level security (Supabase)

The tables hold emails and Stripe ids. Enable RLS with **no** policies for the
`anon` and `authenticated` roles and access them only from server routes with
the service role. If the public site reads events through the client SDK, add:

```sql
ALTER TABLE events ENABLE ROW LEVEL SECURITY;
CREATE POLICY events_public_read ON events FOR SELECT TO anon, authenticated
  USING (status = 'published' AND deleted_at IS NULL);
ALTER TABLE event_participants ENABLE ROW LEVEL SECURITY;   -- no policies: server only
ALTER TABLE stripe_webhook_events ENABLE ROW LEVEL SECURITY;
```

## Implementation

```ts
// lib/events/backends/postgres-store.ts: EventStore on Postgres through a minimal client
// interface, so it drops onto node-postgres, postgres.js, Neon or Supabase's pooler alike.
import { seatDelta } from '../participant-machine';
import { RegistrationError, type EventStore, type MutationResult } from '../ports';
import type { EventRecord, Participant } from '../types';

export interface SqlClient {
  query<R>(text: string, params?: unknown[]): Promise<{ rows: R[] }>;
  /** BEGIN ... COMMIT on one connection; ROLLBACK if fn throws. */
  transaction<T>(fn: (tx: SqlClient) => Promise<T>): Promise<T>;
}

interface EventRow {
  id: string; location_id: string; type_id: string | null; name: EventRecord['name'];
  description: EventRecord['description']; image_url: string | null; date: string;
  time_start: string; time_end: string; time_zone: string; capacity: number; seats_taken: number;
  ticket_price: EventRecord['ticketPrice']; deposit: EventRecord['deposit'];
  cancel_deadline_hours: number; status: EventRecord['status'];
  deleted_at: Date | null; created_at: Date; updated_at: Date;
}

interface ParticipantRow { data: Participant }

const DATE_KEYS = [
  'registeredAt', 'cancelledAt', 'updatedAt', 'nextActionAt',
  'payment.expiresAt',
  'deposit.captureBefore', 'deposit.holdDueAt', 'deposit.holdCutoffAt', 'deposit.settleDueAt',
  'attendance.checkedInAt',
] as const;

/** jsonb holds ISO strings; turn the known date paths back into Dates. */
function reviveParticipant(raw: Participant): Participant {
  const p = structuredClone(raw) as unknown as Record<string, Record<string, unknown> | unknown>;
  for (const path of DATE_KEYS) {
    const [a, b] = path.split('.') as [string, string | undefined];
    const holder = (b ? p[a] : p) as Record<string, unknown>;
    const key = b ?? a;
    const v = holder[key];
    if (typeof v === 'string') holder[key] = new Date(v);
  }
  return p as unknown as Participant;
}

function toEvent(r: EventRow): EventRecord {
  return {
    id: r.id, locationId: r.location_id, typeId: r.type_id, name: r.name, description: r.description,
    imageUrl: r.image_url, date: r.date, timeStart: r.time_start, timeEnd: r.time_end, timeZone: r.time_zone,
    capacity: r.capacity, seatsTaken: r.seats_taken, ticketPrice: r.ticket_price, deposit: r.deposit,
    cancelDeadlineHours: r.cancel_deadline_hours, status: r.status,
    deletedAt: r.deleted_at, createdAt: r.created_at, updatedAt: r.updated_at,
  };
}

const EVENT_COLUMNS = `id, location_id, type_id, name, description, image_url, to_char(date, 'YYYY-MM-DD') AS date,
  time_start, time_end, time_zone, capacity, seats_taken, ticket_price, deposit, cancel_deadline_hours,
  status, deleted_at, created_at, updated_at`;

function upsertParticipant(tx: SqlClient, p: Participant) {
  return tx.query(
    `INSERT INTO event_participants (id, event_id, location_id, customer_id, email, payment_status,
       deposit_status, next_action_at, registered_at, data)
     VALUES ($1,$2,$3,$4,$5,$6,$7,$8,$9,$10)
     ON CONFLICT (id) DO UPDATE SET payment_status = EXCLUDED.payment_status,
       deposit_status = EXCLUDED.deposit_status, next_action_at = EXCLUDED.next_action_at, data = EXCLUDED.data`,
    [p.id, p.eventId, p.locationId, p.customerId, p.email, p.payment.status, p.deposit.status,
     p.nextActionAt, p.registeredAt, JSON.stringify(p)],
  );
}

export function createPostgresEventStore(sql: SqlClient, newId: () => string): EventStore {
  return {
    newParticipantId: newId,

    async getEvent(eventId) {
      const { rows } = await sql.query<EventRow>(`SELECT ${EVENT_COLUMNS} FROM events WHERE id = $1`, [eventId]);
      return rows[0] ? toEvent(rows[0]) : null;
    },

    async createEvent(e) {
      await sql.query(
        `INSERT INTO events (id, location_id, type_id, name, description, image_url, date, time_start, time_end,
           time_zone, capacity, seats_taken, ticket_price, deposit, cancel_deadline_hours, status, created_at, updated_at)
         VALUES ($1,$2,$3,$4,$5,$6,$7,$8,$9,$10,$11,0,$12,$13,$14,$15,$16,$16)`,
        [e.id, e.locationId, e.typeId, JSON.stringify(e.name), JSON.stringify(e.description), e.imageUrl, e.date,
         e.timeStart, e.timeEnd, e.timeZone, e.capacity, JSON.stringify(e.ticketPrice),
         e.deposit ? JSON.stringify(e.deposit) : null, e.cancelDeadlineHours, e.status, e.createdAt]);
    },

    async updateEvent(eventId, i, now) {
      return sql.transaction(async (tx) => {
        const { rows } = await tx.query<{ seats_taken: number }>(
          'SELECT seats_taken FROM events WHERE id = $1 AND deleted_at IS NULL FOR UPDATE', [eventId]);
        const cur = rows[0];
        if (!cur) return 'not_found' as const;
        if (i.capacity < cur.seats_taken) return 'below_seats_taken' as const;
        await tx.query(
          `UPDATE events SET type_id=$2, name=$3, description=$4, image_url=$5, date=$6, time_start=$7,
             time_end=$8, time_zone=$9, capacity=$10, ticket_price=$11, deposit=$12,
             cancel_deadline_hours=$13, status=$14, updated_at=$15 WHERE id = $1`,
          [eventId, i.typeId, JSON.stringify(i.name), JSON.stringify(i.description), i.imageUrl, i.date,
           i.timeStart, i.timeEnd, i.timeZone, i.capacity, JSON.stringify(i.ticketPrice),
           i.deposit ? JSON.stringify(i.deposit) : null, i.cancelDeadlineHours, i.status, now]);
        return 'ok' as const;
      });
    },

    async setEventStatus(eventId, status, now) {
      await sql.query('UPDATE events SET status = $2, updated_at = $3 WHERE id = $1', [eventId, status, now]);
    },

    async getParticipant(eventId, participantId) {
      const { rows } = await sql.query<ParticipantRow>(
        'SELECT data FROM event_participants WHERE event_id = $1 AND id = $2', [eventId, participantId]);
      return rows[0] ? reviveParticipant(rows[0].data) : null;
    },

    async listParticipants(eventId) {
      const { rows } = await sql.query<ParticipantRow>(
        'SELECT data FROM event_participants WHERE event_id = $1 ORDER BY registered_at', [eventId]);
      return rows.map((r) => reviveParticipant(r.data));
    },

    async listDue(now, limit) {
      const { rows } = await sql.query<ParticipantRow>(
        `SELECT data FROM event_participants WHERE next_action_at <= $1
         ORDER BY next_action_at LIMIT $2`, [now, limit]);
      return rows.map((r) => reviveParticipant(r.data));
    },

    async insertParticipant(p) {
      await sql.transaction(async (tx) => {
        // Row lock serializes concurrent registrations for the same event.
        const { rows } = await tx.query<EventRow>(`SELECT ${EVENT_COLUMNS} FROM events WHERE id = $1 FOR UPDATE`, [p.eventId]);
        const event = rows[0] ? toEvent(rows[0]) : null;
        if (!event || event.deletedAt || event.status !== 'published') throw new RegistrationError('registration_closed', 409);
        if (event.seatsTaken + p.ticketCount > event.capacity) throw new RegistrationError('no_spots', 409);
        await tx.query('UPDATE events SET seats_taken = seats_taken + $2, updated_at = $3 WHERE id = $1',
          [p.eventId, p.ticketCount, p.registeredAt]);
        await upsertParticipant(tx, p);
      });
    },

    async mutateParticipant(eventId, participantId, mutate): Promise<MutationResult | null> {
      return sql.transaction(async (tx) => {
        const ev = await tx.query<EventRow>(`SELECT ${EVENT_COLUMNS} FROM events WHERE id = $1 FOR UPDATE`, [eventId]);
        const pr = await tx.query<ParticipantRow>(
          'SELECT data FROM event_participants WHERE event_id = $1 AND id = $2 FOR UPDATE', [eventId, participantId]);
        if (!ev.rows[0] || !pr.rows[0]) return null;
        const event = toEvent(ev.rows[0]);
        const before = reviveParticipant(pr.rows[0].data);
        const after = mutate(before, event);
        if (after === before) return { before, after, event };
        const delta = seatDelta(before, after);
        if (delta !== 0) {
          await tx.query('UPDATE events SET seats_taken = seats_taken + $2, updated_at = $3 WHERE id = $1',
            [eventId, delta, after.updatedAt]);
        }
        await upsertParticipant(tx, after);
        return { before, after, event };
      });
    },

    async claimWebhookEvent(stripeEventId) {
      const { rows } = await sql.query<{ id: string }>(
        `INSERT INTO stripe_webhook_events (id, claimed_at) VALUES ($1, now())
         ON CONFLICT (id) DO UPDATE SET claimed_at = now()
           WHERE stripe_webhook_events.done_at IS NULL
             AND stripe_webhook_events.claimed_at < now() - interval '10 minutes'
         RETURNING id`, [stripeEventId]);
      return rows.length === 1;
    },

    async completeWebhookEvent(stripeEventId) {
      await sql.query('UPDATE stripe_webhook_events SET done_at = now() WHERE id = $1', [stripeEventId]);
    },

    async releaseWebhookEvent(stripeEventId) {
      await sql.query('DELETE FROM stripe_webhook_events WHERE id = $1 AND done_at IS NULL', [stripeEventId]);
    },
  };
}
```

A `SqlClient` over `pg`:

```ts
// lib/events/backends/pg-client.ts: requires `pg` and `@types/pg`
import { Pool, type PoolClient } from 'pg';
import type { SqlClient } from '@/lib/events/backends/postgres-store';

export function pgClient(pool: Pool): SqlClient {
  const wrap = (c: Pool | PoolClient): SqlClient => ({
    query: async <R>(text: string, params?: unknown[]) => ({ rows: (await c.query(text, params)).rows as R[] }),
    transaction: async (fn) => {
      if (c !== pool) return fn(wrap(c)); // already inside one
      const client = await pool.connect();
      try {
        await client.query('BEGIN');
        const result = await fn(wrap(client));
        await client.query('COMMIT');
        return result;
      } catch (err) {
        await client.query('ROLLBACK');
        throw err;
      } finally {
        client.release();
      }
    },
  });
  return wrap(pool);
}
```

## Checklist

- [ ] Migration applied in the host's migration tool, not by hand.
- [ ] Server routes use a role that bypasses RLS; the browser never touches participants.
- [ ] `date` is read back as `YYYY-MM-DD` text (the store's `to_char`), never as a JS `Date`.
