---
name: pocketcqrs-domain
description: "Build and change domains in a PocketCQRS project: JS deciders, projections, reactors and effect functions, plus Go aggregates. Use when writing or editing files in pb_functions/, working with //@trigger or //@schema directives, adding an aggregate/command/event/read model, hot-reloading functions, dry-running candidate code, importing an event-model schema, starting or upgrading a Go program of your own (`init go`, `go generate`), or debugging why a function did not load, a projection wrote nothing, or a reload was refused."
---

# Building a domain with PocketCQRS

PocketCQRS is event sourcing + CQRS on PocketBase. Commands go to a **decider**, which
returns **events**; events are the only source of truth; **projections** fold them into
read-only collections; **reactors** turn them into further commands. Domains are written in
hot-reloadable JS (`pb_functions/*.js`) or compiled Go.

This skill is the workflow and the traps. The reference material lives in the repo and is
authoritative — read it rather than trusting a summary here:

| you need | read |
| --- | --- |
| tiers, bindings, worked examples | `docs/js-guide.md` |
| every directive, exactly | `docs/reference/directives.md` |
| Go aggregates, JS→Go graduation, your own program | `docs/go-guide.md` |
| commands, flags | `docs/reference/cli.md` |
| HTTP surface, dry-run modes | `docs/reference/gateway.md` |
| empty dir → running slice | `docs/tutorial.md`, `docs/getting-started.md` |
| a design document → generated code | `docs/schema.md` |
| personal data, erasure, retention | `docs/personal-data.md` |

## The invariant

**A decider must never reach outside itself.** No `pb`, no `Date`, no `Math.random`, no
network. It sees only the command and the folded state, and returns events. This is what makes
replay reproducible, and everything else rests on it. The runtime enforces it — a decider VM
has those bindings removed — so if a design seems to need decide-time external state, the
design is wrong, not the restriction.

The same rule in one line: **state changes become events by going through a command.** There
is deliberately no write binding in any tier. Projection-owned collections reject direct
writes (`403`, naming the command API).

## The wider rules (read `cqrs-patterns`)

The paradigm both runtimes follow (one-way flow, invariants in streams, eventual consistency
accepted, no polling, no locks, no retry-with-sleep) is in the `cqrs-patterns` skill, along
with its anti-pattern catalogue. In PocketCQRS terms:

- **Live views are built in.** Projection-owned collections are served by PocketBase's REST
  and **realtime subscriptions** (SSE) (`docs/consuming.md`). A screen subscribes to the
  collection, and gets pushed changes as the projection writes them. Don't poll a
  collection, and don't re-read after a command expecting it to be current: show the
  command's outcome and let the subscription update the view.
- **Access is the collection's auth rules.** It is enforced on the server, for REST and
  realtime alike. Hiding UI is cosmetic.
- **Invariants belong to the decider's own stream.** A reactor or effect function reads
  read models only for context. If a rule must hold, model it in a stream or a process
  manager, not as a lookup.
- **Removal:** a projection that lists something must drop its row on the event that
  removes it (unassign, delete, revoke), or the stale row keeps counting and keeps granting.

## The four tiers

| tier | trigger | writes | may call out |
| --- | --- | --- | --- |
| 1 effect | `event`, `http`, `cron` | nothing | `$http` on `event`/`cron` only |
| 2 projection | `projection` | its own collections, via returned row ops | no |
| 3 decider | `decider` | nothing (returns events) | **never** |
| 4 reactor | `reactor` | nothing (returns commands) | `$http` |

## The loop

1. **Design the slice**: what command, what events, what read model answers what question.
2. **Write the file(s)** into `--functionsDir` (default `pb_functions/`). One purpose per file.
3. **Dry-run before activating** — `pocketcqrs dryrun decider|decide|projection <file>`, or
   `POST /api/cqrs/admin/dryrun`, or the dashboard's Functions page. This runs candidate code
   against real history without appending anything.
4. **Reload.** Schema-bearing files (anything with `//@schema`) need the maintenance barrier:
   ```sh
   pocketcqrs system maintenance on
   curl -X POST .../api/cqrs/admin/reload -H "Authorization: Bearer $TOKEN"
   pocketcqrs system maintenance off
   ```
   Effects, HTTP, cron and reactors reload in **running** mode — no barrier for an ordinary
   saga or effect edit.
5. **Read the reload report.** It names what reloaded and what was *refused*, per tier. A
   refusal here is the system telling you something; do not skip past it.
6. **Verify with real traffic**, not by reading. Send the command, check the events, check the
   read model.

## A Go domain lives in its own repository (v0.20.0)

An application's Go domain never goes into pocketcqrs's source (its tutorial `task`/`order` are
the only domains there). It lives in the application's own repository, which depends on
pocketcqrs as a **module version**, never a copy, a checkout or a `replace`, so an upgrade is a
version bump. It still ships as one binary: all of pocketcqrs with the domains compiled in.
`docs/go-guide.md`, "Your own program", is the full workflow.

- **Start:** `pocketcqrs init go <module> --model <document.json>` into an empty directory. It
  writes `go.mod` (pocketcqrs required **and** as a Go `tool`, so one version pins the generator
  and the library), `generate.go`, `model/`, `main.go`, a `Dockerfile`, and the generated
  `domain/`.
- **Change the model:** edit the document, `go generate ./...`, commit `domain/`.
- **Upgrade:** `go get -tool github.com/jamestryand/pocketcqrs@vX.Y.Z`, then `go generate ./...`.
- **The generated `domain/register.go` is the wiring:** `New()`, `Register(h)` (command
  permissions, the read models' collections, deciders (protected where they carry personal
  data), projections, reactors, searches) and `Serve` (the read routes). `main.go` is
  `host.Run(host.Options{Register: d.Register, Serve: d.Serve})`, plus the program's own rules
  (for example `AuthorizeSubjectCommand`) in a `Register` that calls `d.Register` first.
- **Never edit generated files.** A rule the document leaves to you comes as a `.go.example`:
  copy it to a file of your own. A read model removed from the document leaves its old
  generated files behind: delete them.
- `--pocketcqrs <checkout>` (on `init go`, and `schema verify --lang go`) is only for trying an
  unreleased version; a released binary needs neither.
- **JS-only domains need no Go program:** they run on the stock binary, fetched by version
  (`go install github.com/jamestryand/pocketcqrs@vX.Y.Z` or a release), never built from a
  checkout. Until one must publish to another system: then the shell below.

## The maturity path: how a system grows

A system moves along one path, a step at a time, and only when a step is worth it:

1. **JavaScript functions on the stock server.** The model is imported into `pb_functions/`, plus any
   hand-written parts, and the stock `pocketcqrs` binary runs them. The fastest way to prove a domain.
2. **A Go program around the stock server.** `host.Run`, as the stock `main.go` does, plus what is this
   program's own; the JavaScript runs unchanged (below).
3. **Its JavaScript ported to Go.** The aggregates, projections and hand-written parts move from `pb_functions/`
   to generated and hand-written Go, one at a time (`docs/go-guide.md`, "Converting a domain from JS to Go"),
   until the program is a Go domain like any other (`init go`'s shape: `domain/`, generated from the document,
   and its edge tables generated too, so the hand-written publishing table below goes).
4. **dotnetcqrs**, when the system needs what it brings (its own skill, `dotnetcqrs-domain`).

**Step 2 is a shell around the stock server.** `host.Run` still loads `--functionsDir`, so a
program that is only `host.Run` plus its own wiring runs the same JavaScript, unchanged. Take this step when a
system needs something the stock server cannot do, most often **publishing its public events to another system**
over pocketcqrs-extensions' `federation` (the stock server cannot). Do this rather than building a relay beside the
server: a second program reading the server's events is another messaging layer to run and reason about.

- `main.go`: `host.Run(host.Options{Serve: ...})`, with the publisher registered in `Serve` (only the server
  publishes, never a one-shot command of the same binary). No domain package is compiled in: a Go decider may not
  take a name a JavaScript one has.
- **The publishing table is hand-written** while the rules are JavaScript: the generator writes `edges.go` only
  into a Go wiring package. Keep it to what the document says (`[]edges.Out`: the stored event type, the document
  id, the event, its public version and its listed fields), and **test it against the document**:
  `emschema.PublishedEdges(doc)` must equal the table, and each `EventType` must be a type a JavaScript decider
  stores (`type: 'HealthCheckPassed'`), so a model change not made in the table fails the build.
- Build the image's importer and program from the version `go.mod` pins (`go tool pocketcqrs` / `go build
  github.com/jamestryand/pocketcqrs`), not a checkout, so the code that writes `pb_functions` and the code that runs
  them are one release.
- **What crosses from JavaScript:** a field a JavaScript decider leaves unset is stored, and published, as `null`.
  A receiver treats a `null` target field as no target and drops the message, which is usually what you want.
- On first start the publisher sends the whole history of each published event (its checkpoint starts at zero),
  so check what a receiver does with the old ones before deploying.

This shell is pocketcqrs mechanics only. dotnetcqrs hosts are their own programs from the start.

## Traps that have actually cost time

Each of these produced a real defect in this project.

- **`schema verify --functions <dir>` passes a slice whose file is missing.** A scenario whose decider or
  projection file does not exist yet is shown `–` ("the host has no decider") and verify still exits 0
  (v0.29.0). So "write the scenario and watch it fail" does not happen before `schema import`: import, then
  break the generated rule (drop the refusal, drop a projected field) and see verify exit 1, then restore it.
  Read the summary line: `0 skipped` is the pass you want.
- **`schema verify` checks event types, not their data.** A scenario whose event has the right type but a wrong
  field passes, with "would append X — but the example data differs: ..." under its ✓ (v0.29.0, by design). Two
  real defects passed that way: an automation copying its trigger's `status` into the event it started, and events
  missing fields the command never carried (the generated decider copies only what the command names). Treat any
  "example data differs" line as a failure: grep for it in your build. And an automation copies every same-named
  trigger field into its command, `status` too: give the automation slice an `outcomes` entry that sets its event's
  `status`.
- **A projection returning plain objects writes nothing, forever.** `project()` must return
  **row ops** — `{ upsert: { key, fields } }` or `{ delete: <keyval> }` — not the row. Anything
  else is counted and logged as discarded, but the collection just stays empty. If a projection
  "runs fine" and the table is empty, this is why.
- **One purpose per file.** Projection, decider and reactor files must be single-purpose; a
  file mixing tiers is **refused at load** with `must be single-purpose`. (It used to load
  happily and silently discard the extra trigger — the refusal exists because that left a cron
  job that simply never ran.) Split them.
- **Reactor target ids come from the source event.** Delivery is at-least-once, so a
  redelivery must dispatch the *same* command to the *same* id, where the runtime skips a
  reaction already applied. A generated reactor sends to `event.data[<field>]`, the field the
  model names (`targetIdField`, v0.23.0 and eventmodelschema 5.0.0), and declares it with
  `//@target <field>`; the event must carry the value, or the reaction is dropped, said. Never
  a random or time-based id.
- **`//@trigger reactor`, not `react`.** And the function is `reactTo(event)`.
- **A JS decider cannot take a name a Go decider already has.** The reload refuses it and says
  so. Under `--tutorial`, `task` and `order` are taken.
- **Events are append-only and additive.** Add properties, never remove or retype. A shape
  change is a *new event type* plus an upcast transform. Collections are create/extend-only
  too: never drop, retype or rename.
- **`//@handles` must cover what the decider actually folds**, or the dry run and reload fail
  the decider. That is the gate working.
- **Nothing is registered by default.** PocketCQRS ships empty; `--tutorial` opts into the
  example `task`/`order` domains and their collections. Do not assume they exist.
- **A command with no permission is open to every signed-in caller.** Since v0.18.0 the stock
  server enforces a decider's `//@authorize <Command> <policy JSON>` lines (written by
  `schema import` from the model's `requiredRole`/`fieldGatedRole`/`requiredOwnership.via`/
  `scope`); a command without one is not checked, and `users` allows self-registration. Declare
  who may send a command in the model; a superuser passes every policy.

## Who may send a command (v0.18.0)

- The policy travels with the decider file as `//@authorize` lines, so a reload changes it (the
  report counts `commandPolicies`). A hand-written decider may declare one too; a malformed line or
  a command not in that file's `//@commands` is a load error.
- The caller's role is the auth record's `--cqrsRoleField` (default `role`), the same field read
  models' rules read; the server stops a user setting it on themselves. Refused → `403` with the
  policy's message.
- Commands a reactor dispatches are not checked: the check is on the gateway.
- **A machine caller** (a scheduled job, an intake) is a `users` account holding its own role, not a
  superuser: a superuser token lasts 24 hours. Mint a long-lived token with
  `POST /api/collections/users/impersonate/<id>` `{"duration": <seconds>}` as a superuser; resetting
  the account's password kills every token it had (that is the rotation).

## Personal data and erasure

The model is the same as dotnetcqrs's, with the same command and event names: read the
`dotnetcqrs-domain` skill's "Personal data and erasure" section for the shape, and
`docs/personal-data.md` here for the Go API. In short:

- **`field.pii` is crypto-shredded:** encrypted under the person's own key, and erasing them destroys
  the key. Reads then return `{"$redacted":true}`, and new personal data for that id gets `410`.
- **The core `dataSubject` aggregate** (package `pii`): `RequestErasure` records that someone asked,
  and erases nothing. `EraseSubject` erases. `pii.AcceptedCommands(state)` says what a subject
  would act on, for a screen.
- **Governance is opt-in** (package `retention`, `retention.Wire`): a retention policy holds or approves
  a request, a legal hold blocks it, and reviews are due-dated outbox entries, not a timer.
- **Who may ask can be declared** in the document, `dataSubjects.erasure {self, roles, via}` (v0.17.0):
  `authorize.BuildErasureRequest(doc)` into `Authorizer.ErasureRequest`.
- **It fails closed.** The `Authorizer` refuses every `dataSubject` command except a `RequestErasure`
  the declaration admits. With no `Authorize` hook, the gateway refuses them all, and the stock
  server's own hook (v0.18.0) keeps that refusal. Holding, approving,
  legal holds and erasing directly are the host's own rule, checked in `Config.Authorize` before the
  `Authorizer`. Never return true for a `dataSubject` command you have no rule for.

## Calling a third party

`$http` reaches **event and cron functions, and reactors** — never deciders or projections, and
not `//@trigger http` functions (request-driven concurrency is unbounded). It is off unless the
server was started with `--cqrsAllowOutboundHTTP`, and every destination must be named with a
repeatable `--cqrsOutboundHost`. An empty allow-list permits nothing.

One attempt, no retry: an uncaught failure dead-letters and the checkpoint advances. The 3s
call deadline is **per call, not per function** — two sequential slow calls exhaust the 5s
function budget. Full detail in `docs/js-guide.md`.

**Who calls, and when, is not in the model.** `extcaller`'s rules (which events, which third
party, how the answer maps to a command) and any scheduler that sends a command are *sources*,
configured by surface tooling, never in the EventModel document and never by changing the
generator. The model has only the commands they send. See the boundary section in
`cqrs-patterns`.

## When something does not work

- **Import refused: "not a valid EventModeling document"** → since v0.22.0 import checks the
  document against the schema before anything else, and names each problem by its place in the
  document (in a split document, by file). A key the schema requires and the document leaves out
  (a view scenario's `caller.subjectId`, say) is refused there. Fix the document; never the check.
- **File ignored entirely** → directives must be in the **leading comment lines**; anything
  after the first non-comment line is not parsed. A file with no directives is skipped (logged).
- **Reload refused the decider** → read the report's reason: a name collision with a built-in,
  or `//@handles` not covering what it folds, or a failed contract probe.
- **Collection missing** → `//@schema` only materialises through a reload **in maintenance
  mode**. In running mode the schema tier is skipped, and the report says
  `"skipped: not in maintenance"`.
- **Effect function failed** → it dead-lettered rather than blocking the log.
  `pocketcqrs deadletter list`, fix the code, `pocketcqrs deadletter retry all` re-delivers
  through the *current* code.
- **A projection is stuck** → projections do **not** dead-letter; they block at the failing
  event deliberately. Fix the code and `pocketcqrs projection rebuild <name>`.

## Verifying your work

Do not report a domain as working because the code looks right. Send the command, assert the
events came back, and assert the read model reached its **final** state — a row that merely
exists may be half-projected if two events feed it. The project's own smoke suite
(`go test -tags=smoke ./smoke/`) and browser probes (`pocketcqrs-dashboard/probe/`) are worked
examples of doing this properly.
