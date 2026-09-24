# UI

Structure, states and interaction rules only. Build every screen from the host's
own primitives (button, dialog, table, badge, toast) and tokens; never import
another app's classes. All strings are keys under `events.*` in every locale the
host ships.

## Customer surfaces

### Event list / card

| Element | Source | States |
|---|---|---|
| Name, date, local time range, location | event; format time with `event.timeZone`, label the zone if it differs from the viewer's | None |
| Price | `formatMoney(ticketPrice[currency], currency, locale)` for the selected currency; "Free" when `isFreeEvent`; "Free, {deposit} refundable deposit" when a deposit exists | None |
| Seats | `capacity - seatsTaken` | "Sold out" at 0; "Last {n} seats" under a threshold |
| Status badge | `status` | cancelled events are not listed |

The currency picker lists `offeredCurrencies(event)`, not the site's global
list. An event priced only in EUR shows EUR, whatever the site default is.

### Registration dialog

Fields in order: full name, email, phone (optional), tickets (1..min(10, seats
left)), currency (only when more than one is offered), payment method (only
when a balance ledger exists and the customer is signed in and the event is
paid).

| Case | Button | Below the button |
|---|---|---|
| Free, no deposit | "Register" | None |
| Paid | "Pay {total}" | "You'll be redirected to secure checkout" |
| Deposit, `hold_now` | "Hold {deposit total} and register" | Disclosure: nothing charged now; released at check-in; captured on no-show or cancel after {deadline} |
| Deposit, `save_card` | "Save card and register" | Disclosure: hold of {deposit total} placed on {holdDate}; same release/capture rules |

The client can predict `hold_now` vs `save_card` from `eventWindowUtc` and the
policy for the copy; the server decides for real. Map every error code to a
message; `no_spots` refreshes the seat count, field errors focus the field.

### Registered / manage page

Reached from the Checkout success URL and from the manage link in every email.
Polls `GET .../status` every 2 s for up to 30 s while `payment.status` is
`PENDING`: the webhook usually lands within seconds of the redirect.

| Participant state | Shows | Actions |
|---|---|---|
| `PENDING` | "Confirming your payment..." then "Not completed" after polling | "Pay now", then resume |
| `CONFIRMED`, deposit `NONE` | Ticket summary | Cancel (with the refund consequence spelled out) |
| deposit `CARD_SAVED` | "Hold will be placed on {holdDate}" | Cancel |
| deposit `HELD` | "{amount} held, released when you check in" | Cancel (before/after deadline wording) |
| deposit `HOLD_FAILED` / `HOLD_REQUIRES_ACTION` | Warning: "We couldn't hold your deposit. Seat released on {cutoff}" | "Use another card", then resume |
| deposit `RELEASED` / `CAPTURED` | Outcome and amount | None |
| `CANCELLED`, `REFUND_PENDING`, `REFUNDED`, `EXPIRED` | Outcome, refund amount if any | None |

Cancel always goes through a confirm dialog that states the money outcome with
numbers: "You'll be refunded $25.00" / "Your $20.00 deposit will be kept".

## Staff check-in screen

The screen that makes deposits fair: without it every attendee is a no-show.

```
 +- Event name, 18:00 to 21:00, 14 / 20 seats, 9 arrived ------------ [search] -+
 | Name ^        Tickets  Paid / deposit          Attendance        Actions     |
 | Ada Lovelace  2        Deposit $40 held        o not arrived     [Arrived v] |
 | Alan Turing   1        Paid $25                * arrived 1/1                 |
 | Grace Hopper  3        Deposit hold failed !   o not arrived     [Arrived v] |
 +------------------------------------------------------------------------------+
```

| Rule | Why |
|---|---|
| Search by name, email or phone; large tap targets | It is used at a door, on a tablet, with a queue. |
| "Arrived" with a count picker when `ticketCount > 1` | Partial arrival captures only the missing tickets. |
| A partial count can be raised until settlement; a full count releases the hold at once, so the picker confirms before sending it | A released hold cannot be re-placed; a partial one is not captured before `settleDueAt`. |
| "Waive deposit" requires a reason, behind a confirm | It is money the venue gives up; the reason is the audit trail. |
| Show `settleDueAt` in the header ("no-shows charged at 23:00") | Staff know how long they have to finish check-ins. |
| Deposit column shows `lastError` in plain words | "Card declined, seat released at 18:00 tomorrow" beats a status code. |
| Hide Stripe ids; link to the Dashboard payment for admins only | Staff need outcomes, not processor internals. |

## Admin event form

Fields: name and description (per locale), type, image, date, start, end,
time zone (default: the location's zone), capacity, ticket price per currency
(empty = not offered; all empty = free), deposit per ticket per currency (only
enabled while every price is empty or 0), cancel deadline (hours), status
(draft/published).

| Rule | Why |
|---|---|
| Empty price input means "not offered", never 0 | A pre-filled `0` for each currency is exactly how a paid event became free in an unpriced currency. |
| Deposit section disabled on paid events, with the reason shown | Server refuses it (`paid_event`). |
| Capacity cannot go below seats taken | Server refuses it (`below_seats_taken`); show the current count next to the field. |
| No "Cancelled" in the status select | Cancelling is a separate, confirmed action that refunds everyone. |
| Participant list per event: status, amount + currency, deposit, attendance, cancelled by/at | Operators must see money state, not just names. |
| CSV export of the participant list | Door lists, accounting, insurance. |
