---
name: dotnetcqrs-domain
description: "Work on dotnetcqrs (C:\\dev\\sandbox\\dotnetcqrs) or a host built on it (e.g. workstuff/timesheets): EventModel-first changes, `dotnetcqrs-codegen generate --host` / `verify`, merging generated code into a hand-finished host, read-model access rules (requiredRole, selfAccess, grantsAccess), removedByEventIds, command authorization on literal routes, the consumer engine and its read-model change feed / live routes, rebuilding projections, and packaging/publishing a release. Use when editing deciders, projections, query routes, CommandAuthorization, Program.cs wiring, or the codegen itself. Read cqrs-patterns first for the paradigm."
---

# Working with dotnetcqrs

**Read `cqrs-patterns` first.** This skill covers the workflow; that one covers the rules. When
the two seem to disagree, the paradigm wins and this file is out of date.

## Where things are

| What | Where |
|---|---|
| Runtime: engine, stores, gateway | `src/DotnetCqrs`, `src/DotnetCqrs.Abstractions`, `src/DotnetCqrs.Host` |
| PII, erasure | `src/DotnetCqrs.Crypto` |
| Generator and verify harness | `src/DotnetCqrs.Codegen` (`Generation/`, `Mapping/`, `Verification/HarnessProgram.txt`) |
| Pinned schema | `src/DotnetCqrs.Codegen/Schema/eventmodeling.schema.json`, vendored from `C:\dev\sandbox\pcplaypit\eventmodelschema` |
| Tests | `test/DotnetCqrs.Tests` (Postgres/NATS tests skip without their services) |

## The change loop (model first)

1. **Change the EventModel** (for timesheets: `artifacts/eventmodel/timesheets.json` in the
   notebook worktree).
   - Edit it with Python using `newline=''`, and check the result round-trips through
     `json.dumps(indent=2, ensure_ascii=False)` byte for byte.
   - Schema-validate with `npx ajv validate -s eventmodeling.schema.json -d <model>
     --spec=draft2020 -c ajv-formats` from the schema repo's `schema/`.
2. **Write the scenarios first, and see them fail.** Run
   `dotnet run --project src/DotnetCqrs.Codegen.Cli -- verify --input <model>` and record
   PASS/FAIL before and after. Existing failures stay the same set.
   - A stateView scenario can name its `caller` (schema 3.8.0).
   - Seed other people's rows, or the scenario proves nothing.
3. **Generate into a scratch folder, never over the host:** `generate --host --input <model>
   --output <scratch> --project-name <Name> --dotnetcqrs-project <path>/DotnetCqrs.csproj`.
4. **Diff each generated file against the host's.**
   - Take the generated file wholesale only when the hand version adds nothing.
   - Otherwise merge just the generated block, and mark hand logic `[HAND, reason]`.
   - The generated query route selects declared columns only, so a column the projection
     writes but the model doesn't declare **silently disappears**. Compare table columns
     against the generated `Columns` list before taking a route.
5. **Test through the real commands, over HTTP.** Run the new test against the previous
   commit and see it fail (a detached `git worktree` at `HEAD`).

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
- **`requiredOwnership.stateField`** (schema 3.15.0, dotnetcqrs 0.27.0): the aggregate decides
  who owns it from its own state.
  - Use it for anything an entity's own stream knows (e.g. `staffId` on an entry). `via` reads a
    projection before the command, and lags.
  - The generated `Decide` compares the field with `cmd.Actor` after the existence check, and
    throws `CommandForbiddenException`. The gateway answers it with 403, and a hand-written route
    must catch it the same way.
  - The actor is whatever the host passes: timesheets passes the effective actor, so an
    impersonating Manager acts as the owner.
  - `bypassRoles` exists only with `via`, because a decision knows its actor but not their role.
  - Scenarios must name `when.actor`. The verify harness passes it, and since 0.27.0 the
    generated-code harness does too.
- **The generator refuses** schema features it doesn't implement (ingresses,
  partitioning, dataSubjects, `selfAccess.via`, timers, effects) rather than ignoring them.

## Known gaps (don't rely on them; see timesheets' `07-paradigm-audit.md`)

- **Generated command authorization returns early on `requiredRole`.** Ownership, scope or
  fieldGated on the same command are never checked.
- **`via` ownership and scope checks read projections at command time.** That is
  lag-sensitive. For ownership use `requiredOwnership.stateField` (above): timesheets' four
  entry update/delete commands moved to it in W5/D37. A cross-aggregate scope (a PM's projects)
  still reads a projection, as context.
- **`count`/`sum`/`groupBy` rollups aren't idempotent** under redelivery.
- **The reactor target id is per source stream,** so a second trigger is lost.
- **`RunOnceAsync`/`RunExclusivelyAsync` are for tests and offline tools only.** Never use
  them on a live request path.
- **The change feed is in-process only.** Other nodes see new events on the 1-second tick.

## Hand-written routes in a host

A literal `MapPost("/api/cqrs/{aggregate}/{id}/{Command}")` replaces the generic gateway for
that command, **including its authorization**. Call
`CommandAuthorization.AuthorizeAsync(user, aggregate, command, key, payload, store,
EffectiveActor.OwnRole, EffectiveActor.Own, ct)` and return 403. Test that a refused role
gets 403 (S1 was four routes without this).

## The engine and live views (D27)

- **The engine is nudged by every commit,** with a 1-second fallback tick. Each consumer
  catches up in registration order; a blocked consumer retries the same event forever.
- **`IReadModelChangeFeed`** (implemented by `ConsumerEngine`): after a projection
  checkpoints a batch, subscribers to its tables get `ReadModelChanged(table, position)`.
  It is one notification per table per batch, and callbacks must not block.
- **Live routes (0.24.0):** `Map{X}LiveRoute` sits beside `Map{X}Route` and serves
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
  - Timesheets has `LiveQueryService` (fetch-based SSE, because `EventSource` can't send
    `Authorization`).
  - A view needs a **projection** to be live. A value folded from a stream per request
    has no change feed, so declare it as a read model (Settings in L4).
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
existing rows stay until it replays. With the host stopped, run
`--rebuild-projections name[,name…]` (timesheets). It clears the tables and resets the
checkpoints, and accepts projections only, never automations. The write guard must be
installed on the store first (`InstallWriteGuardAsync`), or the guard triggers fail.

## Packaging and publishing (the user's call each time)

1. **Commit the feature,** then "Bump local pack-fallback version to X" in
   `Directory.Build.props`.
2. **Pack a local prerelease for the host:** `dotnet pack -c Release
   -p:Version=X-local.N -o artifacts/local-feed`. Point the host at it on a **branch**,
   so `master` never references a package that only exists on this machine.
3. **Publish, only with the user's go-ahead:**
   - an annotated tag `vX`, pushed to both remotes (`origin`, `github`);
   - the tag runs the nuget.org workflow, and the packages are visible after about 5
     minutes;
   - restore with `--no-http-cache` if the version isn't found.
4. **Switch the branch to X,** re-run host tests, `--verify`, e2e and `compose-check`, then
   merge to `master`.
5. **The schema repo:** the user pushes it themselves (interactive GitHub sign-in).

## Machine notes (Windows, this workstation)

- **Line endings:** files mix LF and CRLF. Edit with the Edit tool, or Python using
  `newline=''` that keeps each file's own endings. Git Bash `sed -i` strips CR, and Python
  text mode adds it. Check `git diff --stat` for whole-file churn before committing.
- **Search:** `rg` is not on the Bash tool's PATH; use the Grep tool.
- **Long paths:** remove scratch worktrees with `node -e "require('fs').rmSync(...)"`.
