# Worked example: a hotel stay

After Adam Dymitruk's hotel in "What is Event Modeling?", so anyone who knows the method recognises
it. The finished document is `examples/hotel-stay.eventmodel.json`. Each step below is what a session
shows the user before going on.

## 0. Goal and scope

"A small hotel: a guest books a room for a night, checks in and out; the bill is closed when they
leave; housekeeping cleans the room, and front desk can let it again. Payment is out of scope for
now." Starts at a booking, ends at a room ready to let.

## 1. Brainstorm

Room Night Booked, Guest Checked In, Guest Checked Out, Bill Closed, Room Cleaned, Room Ready.
(Considered and left out with the user: "Bill Paid", see the outside input below.)

## 2. Plot

```
Room Night Booked -> Guest Checked In -> Guest Checked Out -> Bill Closed
                                                           -> Room Cleaned -> Room Ready
```

## 3. Storyboard

| screen | who | shows |
|---|---|---|
| Booking | a guest, or the front desk | room, night, guest |
| Front Desk | a clerk | each room's status; check in (stay, room, guest, bill), check out (stay, room, bill, date) |
| Housekeeping | a housekeeper | rooms to clean, with the check-out date; mark a room cleaned |

## 4-5. Commands and views

| command | reason |
|---|---|
| Book Room Night | a guest wants a room for a night |
| Check In Guest | the guest has arrived for a booked night |
| Check Out Guest | the guest is leaving; the room needs cleaning and the bill closing |
| Close Bill (sent by an automation) | a stay has ended, so nothing more can be charged to it |
| Mark Room Cleaned | a housekeeper has finished a room |
| Mark Room Ready (sent by an automation) | a cleaned room may be let again |

| view | question |
|---|---|
| Room Status | which rooms are occupied, waiting for cleaning, or ready to let? |
| Rooms To Clean | which rooms does housekeeping still have to clean? |

## 6. Swimlanes

Two teams: **Front Desk** (bookings, stays, bills, room readiness) and **Housekeeping** (cleaning).
Housekeeping's "Room Cleaned" becoming front desk's "Room Ready" is the translation: front desk's own
fact, in its own words.

## 6b. A state machine for every aggregate

`stay` (one stream per stay):

| state | Check In Guest | Check Out Guest |
|---|---|---|
| (none) | Guest Checked In; becomes CheckedIn | refused: "stay does not exist" |
| CheckedIn | refused: "stay already exists" | Guest Checked Out; becomes CheckedOut |
| CheckedOut (terminal) | refused: "stay already exists" | refused: "this guest has already checked out" |

`roomNight` (one stream per room and night, id `<room>-<night>`):

| state | Book Room Night |
|---|---|
| (none) | Room Night Booked; becomes Booked |
| Booked (terminal) | refused: "room night already exists" |

`bill`, `cleaning` and `roomRelease` each have one command that starts the stream and refuses a
second ("bill already exists", and so on): a two-row table each, written out with the user all the
same.

Checks: every pair answered; every state reachable; each terminal state refuses everything by name;
every event named is produced. Checking out twice is a named refusal the desk treats as done, not a
silent success.

**Where "don't overbook" lives.** Dymitruk's point with booking. The guard is in the stream that owns
the room's night: one stream per room and night, so its existing is the guard, and no view is read
(`cqrs-patterns` rule 2). A multi-night booking is one command per night here; a single command for a
range would need a hand-written rule, because `refuseWhen` sees one stream and tests one value.

**A rule this example doesn't enforce, said to the user:** check-in doesn't check that the night was
booked. That fact is in another stream (`roomNight`), and a decider sees only its own. If it must
hold, the booking and the stay become one stream, or something that folds both decides. Here it is a
known gap, not a guess.

## 7. Scenarios (12)

- Book: a free night is booked; a booked night can't be booked again ("room night already exists").
- Check in: a guest checks in.
- Check out: a checked-in guest checks out; twice is refused; an unknown stay is refused.
- Room Status: with two guests in and one checked out, the checked-out room is ToClean (the other
  room's rows are there, so the scenario could fail).
- Close the bill (automation): a check-out closes the stay's bill.
- Rooms To Clean: a checked-out room is listed; a cleaned room leaves the list while the other stays.
- Mark cleaned: a housekeeper marks a room cleaned.
- Room ready (translation): housekeeping's cleaning becomes front desk's room ready.

## Completeness

Every field traced. Two findings, both fixed in the model with the user:

- **The bill's date.** Bill Closed first had a `closedOn` field that nothing supplied: the automation
  copies the check-out event's fields into the command by name, and the check-out event has
  `checkedOutOn`. The bill now records `checkedOutOn`: its fields are named as the trigger names them.
- **The room on check-out.** Guest Checked Out needs the room and the bill for the views and the
  automation that follow. An outcome's `set` takes literal values only, so the command carries them
  (the front-desk screen has them).

## Boundary: the outside input

How payment for a closed bill arrives is an **outside input**: the card processor says a payment went
through. No person and no screen sends it (decision 0033). As a command it is simply:

```json
"record-bill-paid": {
  "name": "Record Bill Paid",
  "aggregate": "bill",
  "reason": "The card processor says the guest's payment went through.",
  "requiredRole": "payments",
  "fields": [
    { "name": "billId", "type": "string" },
    { "name": "amount", "type": "integer" },
    { "name": "paidOn", "type": "integer" }
  ]
}
```

Its sender has its own role (`payments`), not a superuser's. The processor, its webhook and its
signature are its source, configured in surface tooling, never here.

**Today's schema** can only put this command on a slice through an `ingress`, which the schema requires
to carry two source details (`verification`, `deliveryIdField`); both move out of the schema (the
user, hub I5; approved, not built yet), and both generators refuse ingresses. So the document leaves
the slice out, and keeps the question as a hotspot on the close-bill slice. It is never put on a
stand-in screen as if a clerk sent it.

## Systems?

Front desk and housekeeping are teams in one hotel, sharing one store: one document, two lanes. If
housekeeping became an outside company's system, it would get its own document; Room Cleaned would be
marked `public`, and the translation would become an internal edge in a coordinating document
(`systems.md`).

## The document, checked

The JSON is written last: `examples/hotel-stay.eventmodel.json`. Its results are in `generators.md`:
12 of 12 scenarios pass on pocketcqrs (JS and Go) and dotnetcqrs, with no hand-written code.
