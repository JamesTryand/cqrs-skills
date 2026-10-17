# Anti-patterns, with real cases

Each was found in dotnetcqrs, pocketcqrs or an application built on them, most of them in one
audit on 2026-10-09. The cases are told in neutral terms (records, members, managers, invoices);
the lesson in each is what carried over.

## Invariant read from a projection on the write path

- **Symptom:** works in demos. Fails just after a create (a false refusal), just after a
  revocation (a stale permission), or when a projection is blocked or rebuilt.
- **Cases:**
  - Ownership checks read a records projection: a test's delete was refused with 403,
    because the record it had just created wasn't projected yet.
  - Double-invoicing was prevented only by a `locked` column.
  - "One delegated session per manager" was checked against a projection.
  - A deleted record stayed `Exists = true` in its decider, and only a read-model removal
    hid that.
- **Fix:** the owning stream decides: a `Deleted`, `ReservedFor` or `Owner` field in state
  (`requiredOwnership.stateField`, schema ≥ 3.15.0), or a process manager that reserves across
  aggregates through events.

## Rows that outlive the event that removed them

- **Symptom:** a deleted thing still counts; a revoked grant still works.
- **Cases:**
  - A member removed from a project kept access to its documents.
  - Deleted records still counted in pay and invoice totals, and a removed project lead kept
    every lead's grant.
- **Fix:** declare `removedByEventIds`, and add a scenario that seeds the removal and expects
  the row gone. Existing data needs a one-off rebuild of that projection.

## Access enforced only when the client sends the param

- **Symptom:** the UI is fine; a direct API call without the param returns everyone's data.
- **Case:** members could read every colleague's records and pay, decrypted, by leaving out
  the member-id param.
- **Fix:** declare access per read model (`selfAccess`, `grantsAccess`), bound to the caller
  and generated. Test with the param omitted, and with other people's rows seeded.

## Hand-written route skips command authorization

- **Symptom:** a refused caller gets 400 or 500 where the generic gateway gives 403.
- **Case:** four hand-written routes (staging and finalising an invoice, releasing a pay
  period, notifying a flagged member) only checked that the caller was signed in.
- **Fix:** every literal route calls the same authorization. Add a test that a refused role
  gets 403.

## Polling or re-reading until a projection catches up

- **Symptom:** timers, "settle" loops, and flicker or stale values after a save.
- **Cases:**
  - An erasure screen polled 8 × 500 ms after commands.
  - An import screen polled every 2 s, and retried forever on error.
  - A settings screen reloaded stale values over the form after a save.
  - A proposed "refresh until changed" helper (rejected).
- **Fix:** live views (subscription and push), and render the command's own outcome.

## Retry-with-sleep to paper over a design

- **Symptom:** `Thread.Sleep` or `Task.Delay` in a retry loop on a production path.
- **Cases:**
  - A marker file was polled for import status and retried on a Windows sharing violation.
  - The external-caller consumer slept between attempts, blocking every consumer behind it.
    Fixed in dotnetcqrs: the consumer records into an outbox, and a worker beside the engine
    makes the calls.
- **Fix:** model the process as an aggregate with events and a projection. Let the engine's
  redelivery, or an outbox, be the retry.
- **Done for the marker:**
  - The import is an aggregate whose start is a create command, so the stream refuses a
    second run.
  - The status is a projection behind a live view.
  - The commands carry a role no user holds, so only the host dispatches them.
  - Watch for one thing: "running" must cover a finish that is recorded but not yet
    projected, or the status flickers to "interrupted" during the projection's lag.

## Automation reading moving projections

- **Symptom:** results depend on timing, and a replay gives different answers.
- **Case:** an over-allotment flag summed the records projection as it stood *now*, so two
  records created together could both be flagged. Thresholds and recipients were read at
  processing time.
- **Fix:** a process manager that folds its own running totals from the events in order.

## Engine-wide "write then wait" in a live host

- **Symptom:** a batch job freezes every projection and automation while it runs.
- **Case:** a legacy import ran under `RunExclusivelyAsync`, catching up after every row so
  it could read back what it had just written.
- **Fix:** carry the needed context in memory or on events. Keep `RunOnceAsync` for tests
  and offline tools.

## Tests that can't fail

- **Symptom:** green tests, broken behaviour.
- **Cases:**
  - "A member sees their own records" scenarios seeded only the caller's own row, so they
    passed whether or not anything was scoped.
  - A date fixture used `2026-09-04`, but the API returns `2026-09-04T00:00:00Z`, which hid
    a blank date field.
  - Verify ran only the decider, so an error scenario about a scope refusal was reported as
    accepted. Both stacks now run the command's authorization first (dotnetcqrs 0.31.0,
    pocketcqrs 0.15.0).
- **Fix:** seed other people's rows, use real data shapes, and run the new test against the
  old code to see it fail.

## Rollups that double-count

- **Symptom:** a count drifts upward after a crash or a blocked projection.
- **Case:** generated `count`/`sum` rollups incremented without an applied-position guard,
  and apply and checkpoint were not atomic. Fixed in dotnetcqrs (`6c9b75a`): a contributions
  table makes a rollup idempotent.
- **Fix:** an idempotent rollup (position guard or contributions), or apply and checkpoint in
  one transaction.
