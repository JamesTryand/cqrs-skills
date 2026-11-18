# The four patterns

Each pattern is a slice: "the smallest possible work that can be handed over to a developer for
implementation" (eventmodeling.org cheat sheet). The schema has three slice patterns; translation is
an automation across lanes. Examples are from the hotel stay (`worked-example.md`).

## Command (schema: `pattern: "stateChange"`)

trigger, command, event(s). A command is an intention to change the system; it can be refused. The
trigger is a person's screen (`screenId`), or a message from outside (see `boundary.md`).

```
Given  Guest Checked In (stay s-1)
When   Check Out Guest (stay s-1, on 20271204)
Then   Guest Checked Out
```

Scenario kinds: `stateChange` (Then: events) and `error` (Then: a refusal and its message). Which
events go together, which are alternatives, and what they set is the slice's `outcomes`.

## View (schema: `pattern: "stateView"`)

event(s), view. "The views are passive and cannot reject an event after it's been stored" (Dymitruk).
A view shows only information that already exists, so a missing field shows up here.

```
Given  Guest Checked In (r101), Guest Checked In (r102), Guest Checked Out (r101)
When   Room Status for r101, asked by a front-desk clerk
Then   r101: ToClean
```

Scenario kind: `stateView`, with the `caller` it reads as (`subjectId`, and `role` if it matters).

## Automation (schema: `pattern: "automation"`)

event(s), to-do view, automated trigger, command, event(s). A processor works through a to-do list:
"Given: A view of the tasks to do, When This command is launched for each item, Then These events are
expected back" (Dymitruk). No business logic in the processor (the cheat sheet).

```
Given  Guest Checked Out (stay s-1, bill b-1)
When   Close Bill (bill b-1)
Then   Bill Closed
```

**Against the generators** (cqrs-patterns rule 6): the generated reactor reacts to the trigger
events and never reads the to-do view, because a view lags the write side. Draw the view with the
user (it is the method); then put the facts the command needs on the trigger event, and let the
target decider refuse what doesn't hold. The generators copy the trigger event's fields into the
command **by name**. **Say where the command goes** (schema 5.0.0, required): `targetIdField` names
a field every trigger event carries, whose value is the target stream's id (here `billId`), and the
command declares the same field with `idAttribute: true`. Verify fails a scenario whose command
carries another value, so write the trigger's value into the `when`. Something new for each trigger
event (a notification per comment) needs that new thing's id on the trigger. A redelivery reaches the
same stream, where the runtime skips it. Give `readModelId` only if a person also looks at the view.

An automation started by time is a command that carries "today" (see `boundary.md`); the length of
a delay is model data (the user, hub I1).

## Translation (schema: an automation whose trigger is in another lane)

events (one system), view, automated trigger, command, events (the other system). Turn outside
information into our own terms; use it "whenever you have to tell another system that something
happened" (the cheat sheet). Dymitruk's example is GPS points becoming "guest left the hotel".

```
Given  Room Cleaned (housekeeping)
When   Mark Room Ready
Then   Room Ready (front desk)
```

eventmodelschema's design notes call this a "Bridge": a classification, not a field. If the other
lane is really another system, with its own document, the translation becomes a link in a
coordinating document instead (`systems.md`).
