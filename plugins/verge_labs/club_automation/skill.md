---
name: club_automation
description: >
  Use when someone asks to find, book or cancel a court (tennis, squash,
  pickleball, padel, racquetball) at the owner's club, or asks what's open
  or what's already booked. Covers looking up sports and courts, listing open
  times for a date, previewing and booking a slot, listing the owner's
  reservations, and cancelling one, through the club_automation plugin.
---

# Club courts

Invoke via `keepcmd resource invoke <connection> <verb> '<args-json>'`.
`<connection>` is the name in your resource list (for example `club`).
Arguments are one JSON object in single quotes.

Bookings are made in **your owner's name**, on their membership, whoever in
the chat asked. Your keep asks your owner before every `reserve` and
`cancel`; that's expected, not an error. Say so when you ask: "I've asked
<owner> to approve it."

## The verbs

| Verb | Does | Asks the owner |
|---|---|---|
| `list_options` | sports, locations, courts (value + label) | no |
| `list_slots` | open start times for a sport, date and length | no |
| `my_reservations` | upcoming bookings, with ids | no |
| `reserve_preview` | what a booking would be, and who's invoiced | no |
| `reserve` | books it | **yes, every time** |
| `cancel_check` | the portal's warning about cancelling (charges) | no |
| `cancel` | cancels it | **yes, every time** |

## Booking: always in this order

1. **Find the sport's value** (once; remember it for the conversation):
   `list_options` → find the label (`Squash`, `Tennis`) and its value.
   To pick a court, call `list_options` again with `sport` (and `location`)
   and read the court labels (`Court 3`).
2. **List open times:** `list_slots` with `sport`, `date` as `MM/DD/YYYY`,
   `minutes` (60 or 90, whatever the person asked; ask if they didn't),
   and `court` if they named one.
   ```
   keepcmd resource invoke club list_slots '{"sport":"38","date":"10/31/2026","minutes":60}'
   ```
   Each slot has `start` (a number), `location`, `surface` and `time`
   (`7:00pm`). Copy `start`, `location` and `surface` as they are. **Never
   compute or guess `start`.**
3. **Match exactly or ask.** Book only a slot whose `time` is the time asked
   for, on the date asked for. If it isn't open, say what's open nearby and
   let them choose. "Tomorrow evening" is not a time: offer the open ones.
4. **Preview:** `reserve_preview` with the same arguments plus `start`,
   `location`, `surface`. Read the day, time, place and invoice it shows.
5. **Book:** `reserve` with the same arguments plus `summary`: the day, time
   and court in words, as the preview showed them, and any charge, e.g.
   `"Fri Oct 31, 7:00–8:00 PM, Center Court 3, no invoice"`. The summary
   is what your owner reads when asked; it must match the slot.
6. **Confirm:** the answer says "Reservation Completed" when it worked.
   Reply in the chat with the day, time and court.

One booking per request. If someone asks for several, list them and confirm
before booking each.

## Cancelling

1. `my_reservations` → find the booking by date, time and court; take `id`.
   If more than one could match, ask which.
2. `cancel_check` with `id` → read the warning. If it mentions a charge, tell
   the person and ask whether to go ahead before you cancel.
3. `cancel` with `id` and `summary` (the booking in words, plus the warning).
4. `my_reservations` again to confirm it's gone, then say so.

## When things fail

- `auth_failed`: the keep couldn't sign in to the portal. Tell your owner
  their club email or password needs checking on the keep. Don't retry.
- `upstream_refused`: the portal said no (the slot was taken, a limit was
  reached). Relay what it said, then list open times again.
- An ask your owner declined: say they declined; don't try another way.
- Never try to reach the portal any other way than these verbs.
