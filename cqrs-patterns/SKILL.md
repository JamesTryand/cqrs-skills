---
name: cqrs-patterns
description: "The architectural rules for event-sourced CQRS systems (dotnetcqrs, pocketcqrs, and apps built on them such as timesheets): the one-way data flow, where invariants live, what read models may and may not be used for, projections, automations/process managers, eventual consistency, live views, access rules, removal, rebuilds, and model-first testing. Use BEFORE designing or changing any decider, projection, reactor/automation, query route, read model, command authorization, or a UI that shows read models; whenever tempted to poll, retry with a sleep, wait for a projection to catch up, read a projection on the write path, or add a lock; and when reviewing or auditing such code. Runtime-specific workflow is in dotnetcqrs-domain and pocketcqrs-domain; this skill is the paradigm both follow."
---

# CQRS patterns: follow the data flow

These systems are event-sourced CQRS. The rules below come from their designer (James Tryand),
and from real bugs found when code drifted from them. Each anti-pattern in
`references/anti-patterns.md` happened.

> Eventual consistency is okay, and intentional. There is an order to the data flow, and it's
> not just okay, but by design to only follow the flow of the data. There will inherently be
> latency in a given update. Rather than polling, this method calls into callbacks that can
> update state. Trying to scatter locks and updates everywhere causes a fundamentally broken
> system that's difficult to predict.

## The flow

```
command ─▶ decider (pure: own stream only) ─▶ events (the only truth)
                                                  │
                     ┌────────────────────────────┼─────────────────────────────┐
                     ▼                            ▼                             ▼
          projections (own tables)    automations / process managers    (external effects)
                     │                    react ─▶ dispatch commands
                     ▼
              read models ─▶ queries, and LIVE views (pushed to whoever is showing them)
```

Every arrow is one-way. **Latency between any two stages is expected and by design.** Nothing
downstream reaches back upstream to force a stage to be "done".

## The rules

1. **Deciders are pure.** Decide and evolve see only the command and their own stream's
   folded state. No I/O, read models, clock or randomness: time and actor arrive on the
   command.
2. **Invariants live in the stream that owns them.** If a rule must hold, the aggregate
   whose events establish it decides it. "Has this entry been deleted", "is it already on
   an invoice", "who owns it" and "one session per manager" all belong in streams. **Never
   enforce an invariant by reading a projection on the write path.** It lags, so it is
   wrong just after a create, wrong just after a revocation, and wrong whenever a projection
   is blocked or being rebuilt.
3. **A read model may be consulted for context, never as authority.** Looking up another
   aggregate's current rate to stamp onto an event is context. Stamp the value on the event
   so it never needs looking up again. If the answer must be exact, the design needs a
   stream or a process manager, not a read.
4. **Projections write only their own tables and are deterministic under replay.** No
   reading other projections' tables, the clock, or external services while applying. They
   must be idempotent under at-least-once delivery, and rebuildable by resetting their
   checkpoint and replaying.
5. **An event that removes something must remove it from every read model that lists it.**
   Declare it (`removedByEventIds` in schema ≥ 3.9.0), don't hand-write it, and give it a
   scenario. Deleting an entry, unassigning someone or revoking a grant is a removal. A row
   left behind is a stale grant or a deleted thing that still counts.
6. **Automations react to events and dispatch commands.** They are idempotent, with
   deterministic target ids derived from the triggering *event*, not from its stream. They
   never write read models. When an automation needs state (running totals, thresholds,
   recipients), it **folds its own state from the events, in order** (a process manager).
   It never queries a projection, because a projection may already be ahead of or behind
   the event being handled.
7. **Eventual consistency is accepted, not fought.** Never poll until a projection catches
   up. Never retry with a sleep. Never add a read-your-own-writes wait, or locks or
   semaphores to force write/read ordering. Optimistic concurrency on a stream is the only
   write-side concurrency control.
8. **Views stay fresh by subscription, not by polling (live views).** A screen showing a
   view holds a short-lived subscription for as long as it's shown. When a projection
   changes that view, the new value is pushed to it, and the screen updates in a callback.
   - After a command, show the **command's own outcome** (its response carries the
     events). The view follows when the push arrives.
   - Don't re-read straight after a command and treat that read as current.
9. **Access is decided per read model, by the backend.** `requiredRole`, `selfAccess` and
   `grantsAccess` scopes (schema ≥ 3.8.0) are bound to the caller, whatever params the
   request sends. Hiding a link in the UI is cosmetic and never replaces the rule. Every
   write route, hand-written ones included, runs command authorization.
10. **The model comes first, and a test must be able to fail.** Change the event model
    before the code. Write the scenario first and watch it fail. Fixtures must look like
    real data (real date formats, other people's rows present), or the test proves nothing.

## Before you write code that…

| You're about to… | Stop and do instead |
|---|---|
| read a projection inside a command handler, resolver or authorization check, to decide whether the command is allowed | put the fact in the aggregate's state (or a process manager's) and decide it there |
| `Task.Delay`, `Thread.Sleep`, `setTimeout` or a retry loop until data shows up | design for the latency: render the command's outcome, subscribe to the view, or react to the event |
| re-read a list right after a command | show the outcome; let the live view push the change |
| sum or aggregate other rows from a projection inside an automation | fold the running state from the events in the automation itself |
| add a lock, semaphore or "busy" flag to keep write and read in step | don't; if ordering matters, it belongs in one stream |
| leave a row behind on a delete or unassign event | declare the removal, and add a scenario |
| add a hand-written write route | call command authorization in it, and test a refused caller |
| test eventual state with a sleep or polling loop | await the engine's catch-up once (a test-only barrier), or subscribe |

## Runtime notes

- **pocketcqrs:** the runtime enforces decider purity (bindings removed). Projection-owned
  collections reject direct writes. Live views are built in: PocketBase realtime
  subscriptions (SSE) on projection collections. Access is PocketBase collection rules. See
  `pocketcqrs-domain`.
- **dotnetcqrs:** deciders are pure by construction. Access rules and removals are generated
  from the schema. The engine publishes a read-model change feed (`IReadModelChangeFeed`),
  the basis of generated live routes (D27). See `dotnetcqrs-domain`.

## Reference

- `references/anti-patterns.md`: each anti-pattern, its symptom, the real case where it
  happened, and the fix.
