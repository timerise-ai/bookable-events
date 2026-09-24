# Firestore store

Reference `EventStore` on Firestore with `firebase-admin`. Server-only: clients
never read participants directly.

## Layout

| Path | Contents | Client access |
|---|---|---|
| `events/{eventId}` | `EventRecord` minus `id` | read if `published` and not deleted |
| `events/{eventId}/participants/{participantId}` | `Participant` minus `id` | none |
| `stripeWebhookEvents/{stripeEventId}` | `{ claimedAt, doneAt }` | none |

Participants are a subcollection so one event's list is one query, and a
**collection-group** query over `participants` serves the tick and "my
registrations".

## Traps this implementation handles

| Trap | Handling |
|---|---|
| Transaction callbacks are **re-run** on contention | `mutate` is the pure state machine; running it twice is harmless. Never call Stripe or send email inside it. |
| All reads must precede writes in a transaction | `mutateParticipant` reads event and participant together first. |
| Timestamps come back as `Timestamp`, not `Date` | `revive` converts recursively on every read. Writes accept `Date`. |
| `undefined` is rejected on write | The model uses `null` for every empty field. |
| `tx.create` fails if the id exists | Used for inserts, so a retried registration with the same id cannot overwrite. |
| A `where` + `orderBy` on a collection group needs an explicit index | Declared below. Without it `listDue` fails at runtime with a link to create it. |

```ts
// lib/events/backends/firestore-store.ts — EventStore on Firestore (firebase-admin, server only).
//
// Layout:  events/{eventId}
//          events/{eventId}/participants/{participantId}
//          stripeWebhookEvents/{stripeEventId}
// Index:   collection-group `participants`, field `nextActionAt` ascending (for listDue).
import { FieldValue, Timestamp, type Firestore } from 'firebase-admin/firestore';
import { seatDelta } from '../participant-machine';
import { RegistrationError, type EventStore, type MutationResult } from '../ports';
import type { EventRecord, Participant } from '../types';

const CLAIM_TTL_MS = 10 * 60_000;

/** Firestore returns Timestamps; the domain uses Dates. Convert recursively on read. */
function revive<T>(value: unknown): T {
  if (value instanceof Timestamp) return value.toDate() as T;
  if (Array.isArray(value)) return value.map((v) => revive(v)) as T;
  if (value && typeof value === 'object') {
    return Object.fromEntries(Object.entries(value).map(([k, v]) => [k, revive(v)])) as T;
  }
  return value as T;
}

export function createFirestoreEventStore(db: Firestore): EventStore {
  const events = db.collection('events');
  const participants = (eventId: string) => events.doc(eventId).collection('participants');
  const webhookEvents = db.collection('stripeWebhookEvents');

  const readEvent = (snap: FirebaseFirestore.DocumentSnapshot): EventRecord | null =>
    snap.exists ? revive<EventRecord>({ ...snap.data(), id: snap.id }) : null;
  const readParticipant = (snap: FirebaseFirestore.DocumentSnapshot): Participant | null =>
    snap.exists ? revive<Participant>({ ...snap.data(), id: snap.id }) : null;

  return {
    newParticipantId: () => events.doc().id,

    async getEvent(eventId) {
      return readEvent(await events.doc(eventId).get());
    },

    async createEvent(event) {
      const { id, ...data } = event;
      await events.doc(id).create(data);
    },

    async updateEvent(eventId, input, now) {
      const ref = events.doc(eventId);
      return db.runTransaction(async (tx) => {
        const current = readEvent(await tx.get(ref));
        if (!current || current.deletedAt) return 'not_found' as const;
        if (input.capacity < current.seatsTaken) return 'below_seats_taken' as const;
        // `input` is whitelisted by parseEventInput; seatsTaken / locationId are never in it.
        tx.update(ref, { ...input, updatedAt: now });
        return 'ok' as const;
      });
    },

    async setEventStatus(eventId, status, now) {
      await events.doc(eventId).update({ status, updatedAt: now });
    },

    async getParticipant(eventId, participantId) {
      return readParticipant(await participants(eventId).doc(participantId).get());
    },

    async listParticipants(eventId) {
      const snap = await participants(eventId).orderBy('registeredAt', 'asc').get();
      return snap.docs.map((d) => readParticipant(d)).filter((p): p is Participant => p !== null);
    },

    async listDue(now, limit) {
      const snap = await db.collectionGroup('participants')
        .where('nextActionAt', '<=', now)
        .orderBy('nextActionAt', 'asc')
        .limit(limit)
        .get();
      return snap.docs.map((d) => readParticipant(d)).filter((p): p is Participant => p !== null);
    },

    async insertParticipant(p) {
      const eventRef = events.doc(p.eventId);
      await db.runTransaction(async (tx) => {
        const event = readEvent(await tx.get(eventRef));
        if (!event || event.deletedAt || event.status !== 'published') {
          throw new RegistrationError('registration_closed', 409);
        }
        if (event.seatsTaken + p.ticketCount > event.capacity) throw new RegistrationError('no_spots', 409);
        tx.update(eventRef, { seatsTaken: FieldValue.increment(p.ticketCount), updatedAt: p.registeredAt });
        const { id, ...data } = p;
        tx.create(participants(p.eventId).doc(id), data);
      });
    },

    async mutateParticipant(eventId, participantId, mutate): Promise<MutationResult | null> {
      const eventRef = events.doc(eventId);
      const partRef = participants(eventId).doc(participantId);
      return db.runTransaction(async (tx) => {
        // All reads before any write — a Firestore transaction rule.
        const [eventSnap, partSnap] = await Promise.all([tx.get(eventRef), tx.get(partRef)]);
        const event = readEvent(eventSnap);
        const before = readParticipant(partSnap);
        if (!event || !before) return null;
        const after = mutate(before, event);
        if (after === before) return { before, after, event };
        const delta = seatDelta(before, after);
        if (delta !== 0) {
          tx.update(eventRef, { seatsTaken: FieldValue.increment(delta), updatedAt: after.updatedAt });
        }
        const { id, ...data } = after;
        tx.set(partRef, data);
        return { before, after, event };
      });
    },

    async claimWebhookEvent(stripeEventId) {
      const ref = webhookEvents.doc(stripeEventId);
      return db.runTransaction(async (tx) => {
        const snap = await tx.get(ref);
        const data = snap.data();
        if (data?.doneAt) return false;
        const claimedAt = data?.claimedAt instanceof Timestamp ? data.claimedAt.toMillis() : 0;
        if (Date.now() - claimedAt < CLAIM_TTL_MS) return false;
        tx.set(ref, { claimedAt: Timestamp.now(), doneAt: null });
        return true;
      });
    },

    async completeWebhookEvent(stripeEventId) {
      await webhookEvents.doc(stripeEventId).set({ doneAt: Timestamp.now() }, { merge: true });
    },

    async releaseWebhookEvent(stripeEventId) {
      await webhookEvents.doc(stripeEventId).delete();
    },
  };
}
```

## Indexes (`firestore.indexes.json`)

```json
{
  "indexes": [
    {
      "collectionGroup": "events",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "locationId", "order": "ASCENDING" },
        { "fieldPath": "status", "order": "ASCENDING" },
        { "fieldPath": "date", "order": "ASCENDING" }
      ]
    },
    {
      "collectionGroup": "participants",
      "queryScope": "COLLECTION_GROUP",
      "fields": [
        { "fieldPath": "customerId", "order": "ASCENDING" },
        { "fieldPath": "registeredAt", "order": "DESCENDING" }
      ]
    }
  ],
  "fieldOverrides": [
    {
      "collectionGroup": "participants",
      "fieldPath": "nextActionAt",
      "indexes": [
        { "order": "ASCENDING", "queryScope": "COLLECTION" },
        { "order": "ASCENDING", "queryScope": "COLLECTION_GROUP" }
      ]
    }
  ]
}
```

| Index | Serves |
|---|---|
| `events (locationId, status, date)` | Public "upcoming at this location": `status == 'published'`, `date >= todayIn(tz)` |
| `participants (customerId, registeredAt desc)` | "My registrations" on a profile page |
| `participants.nextActionAt` collection-group override | The tick's `listDue` |

## Security rules

```
match /events/{eventId} {
  allow read: if resource.data.status == 'published' && resource.data.deletedAt == null;
  allow write: if false;                       // server only, through the admin routes

  match /participants/{participantId} {
    allow read, write: if false;               // server only — contains emails and Stripe ids
  }
}
match /stripeWebhookEvents/{id} {
  allow read, write: if false;
}
```

A client query must carry the same filters the rule checks
(`where('status', '==', 'published').where('deletedAt', '==', null)`), or
Firestore rejects the whole query. Drafts, cancelled and deleted events stay
invisible to the public SDK.

## Checklist

- [ ] Indexes deployed (`firebase deploy --only firestore:indexes`) before the first tick.
- [ ] Rules deployed; the public site reads events only with the published filters.
- [ ] `getFirestore()` is initialised once with the Admin SDK (service account).
