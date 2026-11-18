# What the generators do with a document

Both generators fail closed: a document that declares something a generator cannot apply is refused,
naming each thing, rather than generated without it. Runtime workflow after generation is in
`pocketcqrs-domain` and `dotnetcqrs-domain`.

## pocketcqrs (`pocketcqrs schema verify | import`)

- **Checks the document against the schema first** (v0.22.0 and later), and names each problem by its
  place (in a split document, by file). Older releases refused only keys they don't read, so a
  missing required key (a view scenario's `caller.subjectId`) went through.
- **Refuses keys it does not read**, naming each: today `selfAccess.via` and partitioning.
- **Reads the boundary** (v0.25.0, schema 6.0.0): an ingress-started slice's command is generated;
  an automation with a `delay`, or an `effect` (its answer and give-up), gets its commands but no
  reactor, since its source fires it; an output generates nothing, and each field it names must be on
  its trigger events. The report names each.
- **Every command and event needs an `aggregate`** (or `--aggregate id=name`); it won't guess from
  lanes.
- **Reads `public`** (4.2.0) and generates nothing from it.
- **An automation** becomes a reactor on the trigger events. It dispatches the command with the
  trigger event's fields copied **by name**, to the stream its `targetIdField` names (v0.23.0, schema
  5.0.0; before, a stream derived from the trigger's id, which could not reach an existing stream
  named by a field). Import refuses an automation without the field, or whose field a trigger event
  or the command (as `idAttribute`) lacks, and (v0.24.0) one whose trigger and command declare a field of one name
  with different types. A redelivery reaches the same stream and is skipped. It
  never reads the slice's `readModelId`, and says so.
- **Personal fields** (`pii`): the Go target (`--lang go`) encrypts them; the JS target refuses them.
- **Not carried into code**: screens, actor lanes, chapters, slice status. Hotspots go to the domain
  document.
- **Every scenario is run** against the generated code. The event types are the assertion; the
  example data is compared too, and a difference is reported as a note, not a failure ("would append
  X, but the example data differs"). An `error` scenario passes only on its own message. A command
  scenario runs the command's authorization first, over views built from its given events. An
  automation scenario's given event from another aggregate is the trigger, said in brackets; it runs
  on the stream the trigger names, and fails if its command carries another value for the field or
  it shows no trigger (v0.23.0).
- **The report** before the scenarios lists every decision taken on the document's behalf ("integer
  becomes number", "read model folds events from 2 aggregates") and what was not carried across. Read
  it out to the user.

```sh
go run github.com/jamestryand/pocketcqrs@<version> schema verify model.eventmodel.json [--lang go]
go run github.com/jamestryand/pocketcqrs@<version> schema import model.eventmodel.json --out <scratch> --force
```

## dotnetcqrs (`dotnetcqrs-codegen verify | generate`)

- **Checks the document against its copy of the schema** (4.2.0 from v0.39.0), naming each problem
  (from v0.39.1, only what to fix; before, every failing branch of the schema too).
- **Refuses** what it does not implement: partitioning, `selfAccess.via`, `partitionFrom`.
- **Reads the boundary** as pocketcqrs does (0.42.0, schema 6.0.0).
- **Reads `public`** and generates nothing from it.
- **An automation** goes to the stream its `targetIdField` names, with the same refusals and the same
  verify check as pocketcqrs (0.40.0, schema 5.0.0); the copied-field type check from 0.41.0.
- **verify** compiles the generated code in memory and runs every scenario, authorization before the
  decider, as the host's gateway does.

```sh
dotnet tool install --tool-path <dir> JamesTryand.DotnetCqrs.Codegen.Cli --version <version>
<dir>/dotnetcqrs-codegen verify --input model.eventmodel.json
```

(`dotnetcqrs-codegen generate --input <doc> --output <dir> [--host]` writes the code; see
`dotnetcqrs-domain`.)

## The hotel example, as checked on 2026-11-18

**At schema 6.0.0** (Record Bill Paid, started by the `card-processor` ingress): 15 of 15 on pocketcqrs
v0.25.0 (JS and Go) and dotnetcqrs 0.42.0.

**Again on 2026-11-18, at schema 5.0.0** (`targetIdField`: `billId` for Close Bill, `cleaningId` for
Room Ready): 13 of 13 on pocketcqrs v0.23.0 (JS and Go) and dotnetcqrs 0.40.0. Before it, Close Bill's
reactor sent to a stream made from the stay (`close-bill-<stay>`), not the bill's own; every scenario
passed regardless. The first check:

13 scenarios, 13 passed, no HAND, on: pocketcqrs v0.21.0 (released, JS); the pocketcqrs build that
became v0.22.0 (JS and `--lang go`); dotnetcqrs 0.39.0 (released). The report's decisions, all
expected: dates as `yyyymmdd` integers become numbers; the two views each fold events from two
aggregates (their projection is generated under one of them and listens to all); one hotspot, three
screens and the slices' status are not carried into code.
