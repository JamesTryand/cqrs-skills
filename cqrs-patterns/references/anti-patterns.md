# Anti-patterns, with real cases

Each was found in dotnetcqrs or timesheets, mostly on 2026-10-09. The full audit is in
timesheets' `build-plan/07-paradigm-audit.md`, with decisions D21–D27 in
`build-plan/00-decisions-and-blockers.md`.

## Invariant read from a projection on the write path

- **Symptom:** works in demos. Fails just after a create (a false refusal), just after a
  revocation (a stale permission), or when a projection is blocked or rebuilt.
- **Cases:**
  - Ownership checks read `timeEntries`: a test's delete was refused with 403, because the
    entry it had just logged wasn't projected yet.
  - Double-invoicing was prevented only by a `locked` column.
  - "One impersonation session per manager" was checked against a projection.
  - A deleted entry stayed `Exists = true` in its decider, and only a read-model removal
    hid that.
- **Fix:** the owning stream decides: a `Deleted`, `ReservedFor` or `Owner` field in state,
  or a process manager that reserves across aggregates through events.

## Rows that outlive the event that removed them

- **Symptom:** a deleted thing still counts; a revoked grant still works.
- **Cases:**
  - D23: an unassigned staff member kept access to the project's documents.
  - D26: deleted time and expense entries still counted in payroll and invoicing, and an
    unassigned PM kept every PM grant.
- **Fix:** declare `removedByEventIds`, and add a scenario that seeds the removal and expects
  the row gone. Existing data needs a one-off rebuild of that projection.

## Access enforced only when the client sends the param

- **Symptom:** the UI is fine; a direct API call without the param returns everyone's data.
- **Case:** D21. Staff could read every colleague's time entries and pay, decrypted, by
  leaving out `staffId`.
- **Fix:** declare access per read model (`selfAccess`, `grantsAccess`), bound to the caller
  and generated. Test with the param omitted, and with other people's rows seeded.

## Hand-written route skips command authorization

- **Symptom:** a staff caller gets 400 or 500 where the generic gateway gives 403.
- **Case:** S1. StageInvoice, FinaliseInvoice, ReleasePayrollPeriod and NotifyStaffOfFlag only
  checked that the caller was signed in.
- **Fix:** every literal route calls the same authorization. Add a test that a refused role
  gets 403.

## Polling or re-reading until a projection catches up

- **Symptom:** timers, "settle" loops, and flicker or stale values after a save.
- **Cases:**
  - The Erasure screen polled 8 × 500 ms after commands.
  - The Import screen polled every 2 s, and retried forever on error.
  - Settings reloaded stale values over the form after a save.
  - A proposed "refresh until changed" helper (rejected).
- **Fix:** live views (subscription and push), and render the command's own outcome.

## Retry-with-sleep to paper over a design

- **Symptom:** `Thread.Sleep` or `Task.Delay` in a retry loop on a production path.
- **Cases:**
  - A marker file was polled for import status and retried on a Windows sharing violation.
  - The ext-caller consumer sleeps between attempts, and blocks every consumer behind it.
- **Fix:** model the process as an aggregate with events and a projection. Let the engine's
  redelivery be the retry.
- **Done for the marker (timesheets L5):**
  - `legacy_import` is an aggregate whose Start is a create command, so the stream refuses
    a second run.
  - The status is a projection behind a live view.
  - The commands carry a role no user holds, so only the host dispatches them.
  - Watch for one thing: "running" must cover a finish that is recorded but not yet
    projected, or the status flickers to "interrupted" during the projection's lag.

## Automation reading moving projections

- **Symptom:** results depend on timing, and a replay gives different answers.
- **Case:** the over-allotment flag summed `timeEntries` as it stood *now*, so two entries
  logged together could both be flagged. Thresholds and recipients were read at processing
  time.
- **Fix:** a process manager that folds its own running totals from the events in order.

## Engine-wide "write then wait" in a live host

- **Symptom:** a batch job freezes every projection and automation while it runs.
- **Case:** the legacy import ran under `RunExclusivelyAsync`, catching up after every row
  so it could read back what it had just written.
- **Fix:** carry the needed context in memory or on events. Keep `RunOnceAsync` for tests
  and offline tools.

## Tests that can't fail

- **Symptom:** green tests, broken behaviour.
- **Cases:**
  - "Staff sees own entries" scenarios seeded only the caller's own row, so they passed
    whether or not anything was scoped.
  - A date fixture used `2026-09-04`, but the API returns `2026-09-04T00:00:00Z`, which hid
    a blank date field.
- **Fix:** seed other people's rows, use real data shapes, and run the new test against the
  old code to see it fail.

## Rollups that double-count

- **Symptom:** a count drifts upward after a crash or a blocked projection.
- **Case:** generated `count`/`sum` rollups increment without an applied-position guard, and
  apply and checkpoint are not atomic.
- **Fix:** an idempotent rollup (position guard), or apply and checkpoint in one transaction.
