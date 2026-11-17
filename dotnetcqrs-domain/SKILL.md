---
name: dotnetcqrs-domain
description: "Work on dotnetcqrs or a host built on it: EventModel-first changes, `dotnetcqrs-codegen generate --host` / `verify`, a generated host in its own repository (NuGet package references, the generator pinned as a `dotnet tool`), merging generated code into a hand-finished host, read-model access rules (requiredRole, selfAccess, grantsAccess), removedByEventIds, command authorization (scope actorField and message) on literal routes, the consumer engine and its read-model change feed / live routes, the outbox, governed erasure (DotnetCqrs.Retention), rebuilding projections, and packaging/publishing a release. Use when editing deciders, projections, query routes, CommandAuthorization, Program.cs wiring, or the codegen itself. Read cqrs-patterns first for the paradigm."
---

# Working with dotnetcqrs

**Read `cqrs-patterns` first.** This skill covers the workflow; that one covers the rules. When
the two seem to disagree, the paradigm wins and this file is out of date.

Where the repos live differs per machine; look them up rather than assuming a path (the
notebook's `reference/environments` `reference/machine-paths.md`, or ask).

## Where things are

| What | Where |
|---|---|
| Runtime: engine, stores, gateway | `src/DotnetCqrs`, `src/DotnetCqrs.Abstractions`, `src/DotnetCqrs.Host` |
| PII, erasure (request + erase) | `src/DotnetCqrs.Crypto` |
| Governed erasure (holds, reviews, retention) | `src/DotnetCqrs.Retention` (opt-in package, since 0.31.0) |
| Postgres backend | `src/DotnetCqrs.Postgres` |
| Generator and verify harness | `src/DotnetCqrs.Codegen` (`Generation/`, `Mapping/`, `Verification/HarnessProgram.txt`) |
| Pinned schema | `src/DotnetCqrs.Codegen/Schema/eventmodeling.schema.json`, vendored from the eventmodelschema repo (`schema/`); the csproj comment names the commit |
| Tests | `test/DotnetCqrs.Tests` (Postgres/NATS tests skip without their services) |

## The change loop (model first)

1. **Change the EventModel** (the host's own document, kept beside its planning).
   - Edit it with Python using `newline=''`, and check the result round-trips through
     `json.dumps(indent=2, ensure_ascii=False)` byte for byte.
   - Schema-validate with `npx ajv validate -s eventmodeling.schema.json -d <model>
     --spec=draft2020 -c ajv-formats` from the schema repo's `schema/`.
2. **Write the scenarios first, and see them fail.** Run
   `dotnet tool run dotnetcqrs-codegen verify --input <model>` in the host's repository (the
   pinned tool; inside dotnetcqrs itself, `dotnet run --project src/DotnetCqrs.Codegen.Cli --
   verify ...`) and record PASS/FAIL before and after. Existing failures stay the same set.
   - A stateView scenario can name its `caller` (schema 3.8.0); a command scenario names
     `when.actor`.
   - Seed other people's rows, or the scenario proves nothing.
3. **Generate into a scratch folder, never over a hand-finished host:** `dotnet tool run
   dotnetcqrs-codegen generate --host --input <model> --output <scratch> --project-name <Name>`.
4. **Diff each generated file against the host's.**
   - Take the generated file wholesale only when the hand version adds nothing.
   - Otherwise merge just the generated block, and mark hand logic `[HAND, reason]`.
   - The generated query route selects declared columns only, so a column the projection
     writes but the model doesn't declare **silently disappears**. Compare table columns
     against the generated `Columns` list before taking a route.
5. **Test through the real commands, over HTTP.** Run the new test against the previous
   commit and see it fail (a detached `git worktree` at `HEAD`).

## A host lives in its own repository (0.37.0)

A generated host is an application: it lives in its own repository and depends on dotnetcqrs
as **NuGet packages**, never its source or a checkout (`docs/tutorial.md`, "Running it as a
host").

- **Start:** `dotnet new tool-manifest`, then `dotnet tool install
  JamesTryand.DotnetCqrs.Codegen.Cli --version <v>`, then `dotnet tool run dotnetcqrs-codegen
  generate --host ...`.
- **What it writes:** the `.csproj` references `JamesTryand.DotnetCqrs`, `.Host`, `.Codegen`,
  `.Crypto`, `.Postgres` and `.Telemetry.Nats` at the generator's own version. The tool
  manifest `dotnet tool` finds (the nearest at or above the project, or a new
  `.config/dotnet-tools.json`) gets the generator pinned at that same version; other tools in
  it are kept. One version pins the generator and the libraries together.
- **Upgrade:** `dotnet tool update JamesTryand.DotnetCqrs.Codegen.Cli --version <new>`, then
  generate again: the `.csproj` and the manifest move together. `--dotnetcqrs-version` names
  another published version.
- **`--dotnetcqrs-project <checkout>/src/DotnetCqrs/DotnetCqrs.csproj`** writes project
  references instead, and no manifest: only for trying an unreleased version, and for
  dotnetcqrs's own tests. A host committed with project references depends on dotnetcqrs's
  source; switch it back before committing.
- Releases before 0.37.0 required `--dotnetcqrs-project` for `--host`.

## What verify runs

- **Command scenarios:** the generated `CommandAuthorization` first, then the decider, as the
  gateway does (since 0.31.0). The read models the command's authorization reads (a
  `requiredOwnership.via`, a `scope`) are built from the scenario's given events. A scenario
  names an actor, not a role, so it holds the command's required roles and no bypass role:
  only the checks about who the actor is can refuse.
- **An error scenario passes only on its own message.** A scope refuses with its `message`
  (schema 4.0.0), or "not authorized" when it declares none, which no scenario should rely on.
- **Host verification** (`--verify` against `HostDeciders`) runs the host's deciders and
  projections, but not yet the authorization (a known gap).
- Generated refusals name the aggregate in words: "loan request does not exist", not
  "loanRequest does not exist" (0.31.0).

## Declarations the generator honours

- **`requiredRole`** on read models and commands.
- **`selfAccess {subjectField, param}` and scope `grantsAccess`** (schema 3.8.0).
  - Holders of `requiredRole` see every row. Anyone else sees only their own rows, plus
    the rows a granting scope admits for them.
  - Bound params ignore the value sent.
  - A route with these rules needs `resolveOwnRole` **and** `resolveSubjectId` wired; it
    fails closed (403) without them.
- **`removedByEventIds`** (schema 3.9.0) deletes the row the event's stream keys. It must
  also be in `builtFromEventIds`.
- **`refuseWhen`** (3.10.0/3.11.0) and **`outcomes`** (3.12.0, `repeated` 3.13.0) are generated
  into `Decide`. **`rowKeyField`** (3.14.0) keys a read model's row by an event field.
- **`requiredOwnership.stateField`** (schema 3.15.0): the aggregate decides who owns it from
  its own state.
  - Use it for anything an entity's own stream knows (the member who made a record). `via`
    reads a projection before the command, and lags.
  - The generated `Decide` compares the field with `cmd.Actor` after the existence check, and
    throws `CommandForbiddenException`. The gateway answers it with 403, and a hand-written route
    must catch it the same way.
  - The actor is whatever the host passes (an impersonating manager passed as the effective
    actor acts as the owner).
  - `bypassRoles` exists only with `via`, because a decision knows its actor but not their role.
- **Command `scope`** (schema 4.0.0): `memberOfVia.actorField` is required, the membership
  read model's field compared with the actor. An optional `message` is the refusal's wording:
  the generated `AuthorizeAsync` still returns `bool`, but throws `CommandForbiddenException`
  with that message when the scope refuses, and the gateway answers 403 with it. A literal
  route that calls `AuthorizeAsync` must catch it too.
- **`rowsPerListItem {listField, groupBy}`** (schema 3.16.0): a row per group of a stream's list
  items, keyed `<stream>|<group>`.
  - Each event carrying the list replaces it (the events must carry the whole list). The stream's
    other fields are merged into a companion `<table>_streams` table and copied onto every row;
    the rows are rebuilt after each event, so a rebuild replays to the same rows.
  - An item without the group subfield is in no row. There is no fallback to a stream-level value.
  - **`sumOfItems`** sums a subfield over the row's items. It is null unless every item carries
    the subfield, so amounts not yet in one currency are never added.
  - A table made before the projection was generated has no hidden `_stream` column. Add it in
    the host's `InitAsync`, and rebuild once.
- **`projection: "hand"`** (schema 3.17.0): a read model built by a hand-written projection, for
  a join no declaration expresses. The route is still generated. Model-only verify reports its
  view scenarios HAND. The host's `--verify` must name the projection in
  `HostDeciders.Projections` (`["late-orders"] = typeof(LateOrdersProjection)`), and then runs
  it. Without that it fails, saying why. A host may also name its own projection for a
  generated read model.
- **`byEvent {values}`** (3.16.0): a field set to a literal per event, such as a status. An event
  it doesn't name leaves the field alone. It works on any read model.
- **A command's authorization declarations combine:** every one it declares must hold.
- **`generate --host` enforces them (0.36.0)** when the document declares any command permission:
  `MapCqrsGateway(authorize: ...)` calls the generated `AuthorizeAsync`. Without a declaration
  `Program.cs` is unchanged and every command is open to any caller.
  - Role and subject come from the claims named by `Cqrs:RoleClaim` (default `role`) and
    `Cqrs:SubjectClaim` (default `ClaimTypes.NameIdentifier`).
  - The scaffold has no sign-in, so a declared command answers 403 until the host adds one.
  - Command names match whatever their case; `dataSubject` commands keep their refusal.
- **`dataSubjects.erasure`** (3.2.0, 3.7.1): who may send `RequestErasure` (`self`, `roles`, `self`
  through `via`). The generated `AuthorizeAsync` refuses every other `dataSubject` command, and all of
  them without the declaration; a host allows holds, approvals or direct erasure in its own delegate.
- **The generator refuses** schema features it doesn't implement (ingresses,
  partitioning, `selfAccess.via`, timers, effects) rather than ignoring them.

## Known gaps (don't rely on them)

- **`via` ownership and scope checks read projections at command time.** That is
  lag-sensitive. For ownership use `requiredOwnership.stateField` (above). A cross-aggregate
  scope (a lead's projects) still reads a projection, as context.
- **A reaction redelivered after a crash** is skipped since dotnetcqrs `0ace646` (released in 0.38.0,
  2026-11-17): `ReactorConsumer` stamps a commandId derived from the reactor, the cause and the
  index, and `DeciderRegistry` skips a command whose id is already in the target stream. Before
  that, an update onto a stream with no create-once rule (auto-ship) could be applied twice;
  on an older release, rely on the target refusing a redelivery. A dotnetcqrs gateway still
  ignores `Idempotency-Key`, so a follow-up sent to one over HTTP relies on its target.
- **`RunOnceAsync`/`RunExclusivelyAsync` are for tests and offline tools only.** Never use
  them on a live request path.
- **Other nodes' commits:** on Postgres every append sends `pg_notify` and the store `LISTEN`s, so a
  reader node's engine wakes at once (`IPollSource.SubscribeToCommits`). A SQLite reader on a
  LiteFS replica still sees new events on its 1-second tick: nothing to listen to.
- **A hand query over a generated read model can't be verified.** Cover it with HTTP tests. A
  hand *projection* can be (`projection: "hand"`, above).
- **`rowsPerListItem` refuses** rollups, pii fields, search indexes and `rowKeyField` with it.

## The outbox (0.31.0)

`IOutbox` (`DotnetCqrs.Abstractions.Outbox`, implemented by the event stores, SQLite and
Postgres) is a general due-time outbox: a consumer records work as it handles an event
(`EnqueueAsync(consumer, key, payload, due)`; the same key twice does nothing, so a redelivery
is harmless), and an `OutboxWorker` beside the engine does it once due. Nothing slow or failing
holds up the engine, a retry survives a restart, and work due on a date waits for that date.
The retention reviews use it (`ErasureGovernor`): its worker must run beside the engine.
External calls (ExtCaller) have their own outbox, `IExtCallOutbox`: the consumer records, and
`consumer.Worker.RunAsync(ct)` must run beside the engine to make the calls.

**Who calls, and when, is not in the model.** ExtCaller's configuration and any scheduler that
sends a command are *sources*, configured by surface tooling, never in the EventModel document and
never by changing the codegen. The model has only the commands they send. See the boundary
section in `cqrs-patterns`.

## Personal data and erasure

- `DotnetCqrs.Crypto`: the `dataSubject` aggregate with `RequestErasure` (records a request,
  erases nothing) and `EraseSubject` (records the erasure; `SubjectKeyDestroyer` destroys the
  key). `DataSubject.AcceptedCommands(state)` says which a subject would act on, for a screen;
  don't repeat stage rules in a UI.
- `DotnetCqrs.Retention` (opt-in, 0.31.0): holds, reviews and retention policies decide a
  request. Wire it with `RegisterGovernedDataSubjects` and `RegisterErasureRetention`; the
  governor's reviews run through the outbox. See `docs/retention.md`.
- The gateway refuses every `dataSubject` command until a host wires `authorize` (fail closed);
  the wiring `generate --host` writes (0.36.0) keeps that refusal.
  The generated `CommandAuthorization` admits only `RequestErasure`, as the document's
  `dataSubjects.erasure` says; never return true for a `dataSubject` command you have no rule for.

## Hand-written routes in a host

A literal `MapPost("/api/cqrs/{aggregate}/{id}/{Command}")` replaces the generic gateway for
that command, **including its authorization**. Call
`CommandAuthorization.AuthorizeAsync(user, aggregate, command, key, payload, store,
resolveOwnRole, resolveOwnSubjectId, ct)`, return 403 on `false`, and catch
`CommandForbiddenException` as 403 with its message. Test that a refused role gets 403.

## The engine and live views

- **The engine is nudged by every commit,** with a 1-second fallback tick. Each consumer
  catches up in registration order; a blocked consumer retries the same event forever.
  **Order is when, not a guarantee:** a consumer reads to the head as it is when it runs,
  so a later one can be ahead of an earlier one. Never decide from another consumer's output.
- **`IReadModelChangeFeed`** (implemented by `ConsumerEngine`): after a projection
  checkpoints a batch, subscribers to its tables get `ReadModelChanged(table, position)`.
  It is one notification per table per batch, and callbacks must not block.
- **Live routes:** `Map{X}LiveRoute` sits beside `Map{X}Route` and serves
  `GET {prefix}/{collection}/live` as SSE.
  - Both routes call one generated `QueryAsync`, so the stream applies the same access
    rules and refusals.
  - A stream sends `event: view` with `{"position", "rows"}`: first the current result,
    then a push per change to the view's table or its scopes' via-tables.
  - It re-runs with the viewer's own rules, and pushes only a changed result. Re-runs are
    coalesced.
  - It ends on disconnect, or with `event: expired` when the `exp` claim expires.
  - The runtime is `LiveView` in `DotnetCqrs.Host`.
- **Wiring a host:**
  - Register the engine as `IReadModelChangeFeed`.
  - Map each live route with **the same hooks and `.RequireAuthorization()`** as its
    query route. It is a separate builder, so it doesn't inherit them.
  - Pass the PII read models to `RegisterPiiCacheEvictorAsync(..., viewTables:)`, so
    erasure pushes the redacted result. That works through `IChangesViews`, which lets a
    non-projection consumer report the tables it changed.
- **In the UI:** show a command's outcome, and let the view update when its push arrives.
  Never poll.
  - A browser `EventSource` can't send `Authorization`; use fetch-based SSE.
  - A view needs a **projection** to be live. A value folded from a stream per request
    has no change feed, so declare it as a read model.
- **Granularity is per table per batch:** a projection publishes for all its tables on
  every batch it processes, even when the batch's events don't touch them.
  - Viewers re-run and usually find nothing changed, so nothing is pushed.
  - That also hides missing wiring from end-to-end tests: a push may arrive "by luck"
    from an unrelated batch. Prove a specific publisher (like the evictor's
    `viewTables`) with a unit test on the feed.
- **e2e:** `networkidle` never comes while a live view is open. Wait for each request's
  answer instead; a live view is answered when its response headers arrive.

## Rebuilding a projection

When a projection's rules change for events it has already applied (a new removal, say),
existing rows stay until it replays. With the host stopped, run the host's rebuild command for
the named projections. It clears the tables and resets the checkpoints, and accepts projections
only, never automations. The write guard must be installed on the store first
(`InstallWriteGuardAsync`), or the guard triggers fail.

## Packaging and publishing (the user's call each time)

1. **Commit the feature,** then "Bump local pack-fallback version to X" in
   `Directory.Build.props`.
2. **Pack a local prerelease for the host:** `dotnet pack -c Release
   -p:Version=X-local.N -o artifacts/local-feed`. Point the host at it on a **branch**,
   so `master` never references a package that only exists on this machine.
3. **Publish, only with the user's go-ahead:**
   - an annotated tag `vX`, pushed to both remotes (`origin`, `github`);
   - the tag runs the nuget.org workflow on the `github` mirror, and the packages are
     visible after about 5 minutes (check the flat-container API, e.g.
     `https://api.nuget.org/v3-flatcontainer/jamestryand.dotnetcqrs/index.json`);
   - restore with `--no-http-cache` if the version isn't found.
4. **Switch the branch to X,** re-run host tests, `--verify` and e2e, then merge to `master`.

## Machine notes (Windows)

- **Line endings:** files mix LF and CRLF, per file. Edit with the Edit tool, or Python using
  `newline=''` (or bytes) that keeps each file's own endings. Git Bash `sed -i` strips CR, and
  Python text mode adds it, and Git Bash's `grep -c $'\r'` reports 0 on a CRLF file: count with
  Python. Check `git diff --stat` for whole-file churn before committing.
- **Tests:** the suite takes minutes on a small machine; run one suite at a time. Postgres tests
  need `DOTNETCQRS_PG` (any reachable Postgres 14+; `ops/postgres/verify.sh` stands one up), or
  they skip, and a skipped count means Postgres was never exercised.
- **Search:** use the Grep tool where `rg` isn't on the shell's PATH.
- **Long paths:** remove scratch worktrees with `node -e "require('fs').rmSync(...)"`.
