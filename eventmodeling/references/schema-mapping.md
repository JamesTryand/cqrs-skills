# The method in the schema (eventmodelschema 6.1.0)

## Concept to schema

| method | schema | note |
|---|---|---|
| event (fact) | `events.<id>`: `name`, `swimlaneId`, `aggregate`, `fields` | `aggregate` is optional in the schema, required by pocketcqrs |
| command (intention) | `commands.<id>`: `reason`, `aggregate`, `fields`, `refuseWhen`, who may send it | `reason` is optional so a draft still saves |
| view | `readModels.<id>`: `question`, `builtFromEventIds`, `fields` (derivations), `removedByEventIds` | `question` optional, as above |
| screen | `screens.<id>`: a name (and `actorLaneId`) | no fields: the storyboard's field lists live in the conversation |
| processor | `automations.<id>` (a name) and an `automation` slice | |
| command pattern | slice `pattern: "stateChange"`: `screenId` (or `ingressId`, or neither), `commandId`, `eventIds`, `outcomes` | neither (6.1.0): another system sends it over an internal edge; the command must be `public` |
| outside input | `ingresses.<id>`: a name, description, swimlane; a `stateChange` slice's `ingressId` | no source details (6.0.0); something started by time is one too, its command carrying the date |
| outside output | `outputs.<id>`: `swimlaneId`, `triggerEventIds`, `fields` (what leaves) | 6.0.0; nothing is generated |
| call out | an automation's `effect` (`swimlaneId`, `gaveUp`): its answer is the slice's command | the call is a source's; the commands are generated |
| view pattern | slice `pattern: "stateView"`: `screenId`, `readModelId` | |
| automation pattern | slice `pattern: "automation"`: `automationId`, `triggerEventIds`, `targetIdField` (5.0.0), `commandId`, `resultEventIds`; optionally `delay` (its length) or `effect` | `readModelId` optional; generators don't read it |
| translation | an automation whose trigger events are in another lane | removed as a pattern in schema 2.0 |
| swimlanes | `swimlanes[]`, `kind: "team"` or `"system"` | who owns the events |
| people on the storyboard | `actorLanes` | notation only |
| timeline order | the order of `slices[]` | left to right |
| chapters | `chapters`, a slice's `chapterId` | |
| Given/When/Then | a slice's `scenarios`: `stateChange`, `error`, `stateView` | one command or one view each |
| unknown | `hotspots.<id>`: `note`, `target` (an element, a slice, or an ordering of two slices) | |
| board state | a slice's `status` | not carried into code |
| part of a system's contract | `public: {"version": n}` on an event or command (4.2.0) | see `systems.md` |

## Shapes worth knowing

- **The event that starts a stream** has a field with `idAttribute: true`. A command whose event
  starts a stream is refused with "<aggregate> already exists" if the stream exists; any other
  command is refused with "<aggregate> does not exist" if it doesn't. Both are generated.
- **A status** is a field on the events, given its value by the slice's `outcomes` (`set`, literal
  values only), and checked by `refuseWhen` (`{"state": "status", "equals": "CheckedOut"}`). A
  command can't copy a value from state onto an event with `set`: the command carries it.
- **One stream, many rows**: `rowKeyField` names the field (on every event the view is built from or
  removed by) that picks the row, in place of the stream (3.14.0).
- **A row that goes away**: `removedByEventIds`, and the event must be in `builtFromEventIds` too
  (3.9.0). An event that removes something must remove it from every view that lists it
  (`cqrs-patterns` rule 5).
- **A field set by which event fired**: derivation `byEvent` with `values` (3.16.0).
- **Who may send**: `requiredRole` on every command where anyone can sign up; `requiredOwnership`,
  `scope` for finer rules. **Who may read**: `requiredRole`, `selfAccess`, `grantsAccess` on a view.
- **Personal data**: `pii` on a field, `piiSubject`, and `dataSubjects.erasure` (see `cqrs-patterns`'
  `references/personal-data.md`).

## The method has it, the schema doesn't

- A screen's fields and wireframe: a screen is a name. The completeness check is done in the
  conversation.
- Translation as its own pattern: an automation across lanes.
- The methodology rules (a reason, a question, the anti-patterns): judgement, in `method.md`'s
  checklist. What a generator can check, it checks.
- Starting an automation on a field's value: `triggerEventIds` names events only, and there is no
  filter on purpose (hub L15, 2026-11-18). Split the event into business events, one per meaning,
  each with its own command and automation.

## The schema has it, the method doesn't

Typed fields, `aggregate`, `outcomes`, `refuseWhen`, authorization, derivations (`toggle`, `count`,
`sum`, `groupBy`, `byEvent`, `sumOfItems`), `rowKeyField`, `rowsPerListItem`, `projection: "hand"`,
personal data, partitioning, `public`. These are the developer's detail; add them only after the user
has agreed the story.
