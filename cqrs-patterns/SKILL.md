---
name: cqrs-patterns
description: "The architectural rules for event-sourced CQRS systems (dotnetcqrs, pocketcqrs, and the applications built on them): the one-way data flow, where invariants live, what read models may and may not be used for, projections, automations/process managers, eventual consistency, live views, access rules, removal, rebuilds, model-first testing, and personal data (GDPR: PII fields, erasure, the key service). Use BEFORE designing or changing any decider, projection, reactor/automation, query route, read model, command authorization, or a UI that shows read models; whenever tempted to poll, retry with a sleep, wait for a projection to catch up, read a projection on the write path, or add a lock; whenever modelling a timer, schedule, webhook, ingress or call to an outside system (the boundary: messages enter and leave, sources are not in the model); and when reviewing or auditing such code. Runtime-specific workflow is in dotnetcqrs-domain and pocketcqrs-domain; this skill is the paradigm both follow."
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
  sources (people, schedulers, webhooks, outside systems): NOT in the model
     │ messages enter as commands                          ▲ messages leave
─────┼─────────────────────────── the domain graph ────────┼──────────────────────
     ▼                                                     │
command ─▶ decider (pure: own stream only) ─▶ events (the only truth)
                                                  │
                     ┌────────────────────────────┼─────────────────────────────┐
                     ▼                            ▼                             ▼
          projections (own tables)    automations / process managers    calls out (ext-caller);
                     │                    react ─▶ dispatch commands    the answer comes back
                     ▼                                                  in as a command
              read models ─▶ queries, and LIVE views (pushed to whoever is showing them)
```

Every arrow is one-way. **Latency between any two stages is expected and by design.** Nothing
downstream reaches back upstream to force a stage to be "done".

## The boundary: messages enter and leave; sources are not in the model

The domain model is a directed graph. Messages cross its boundary in **three shapes only**:

| shape | what it is | in the model |
|---|---|---|
| input → output | a call out and its answer | an automation's call (ext-caller); the answer and the give-up both come back in as **commands** |
| input → unit | an outside input: a message enters, nothing goes back | a command the outside sends ("a period has ended", "payment arrived") |
| unit → output | an outside output: a message leaves, nothing comes back | a publish or notification after an event |

- **The model names the message and its meaning, never its source.** A scheduler, webhook, queue,
  person or provider is a *source*. A schedule is modelled as the domain idea ("renewal is due")
  and the command it sends, never as a cron job, timer service, URL, signature scheme or credential.
- **The test:** does it say where a message comes from, or how it is carried? Then it is **not** in
  the model and not in a generator. Sources are configured by surface tooling (`sctl`,
  `lab/surface-tooling`). Does it say that a message enters or leaves, and what it means? Then it is.
- **Never change a generator's or the schema's syntax or meaning to carry a source.** If something
  seems to need it, stop and ask the user. (Done wrongly several times; the user's decision is
  `0033` in the venture-overview hub.)
- **Time arrives on the command.** A decider can't read the clock, so a scheduled check carries
  "today" (and any outside answer) as command fields; the stream records deadlines it needs to
  compare (for example, when grace ends), so the guard compares two values it already has.
- **Decided by the user (2026-11-14):** a delay's length ("grace lasts 7 days") is in the model;
  firing it is a source's job. Delays are not cancelled by events: the decider's guards make a late
  message harmless. Scenarios have no "time passes" step: send a command with a later date. A
  message that leaves with no answer is a shape of its own, not an effect with an optional answer.
  Approved 2026-11-15 but not built: the schema drops those two and moves `schedule.cron`/`timeZone`
  and an ingress's `verification`/`deliveryIdField` out to surface tooling. Until then, don't use any
  of them.
- **Two documents, joined by message ids (the user, 2026-11-15).** The event model holds only
  model parts. Surface tooling's own description holds the sources (schedule, webhook adapter,
  signature check, host, secrets) and names the model's messages by id, never the reverse.

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
   checkpoint and replaying. **Each projection has exactly one writer.** The core flow is single
   and provably correct; a second runner added for convenience (a rebuild tool with its own
   engine, say) can apply the same events twice. A rebuild goes through the one writer.
   Peripheral consumers that write no projection (an external caller) may run separately,
   even as a pool, as long as each request is handled by exactly one of them.
5. **An event that removes something must remove it from every read model that lists it.**
   Declare it (`removedByEventIds` in schema ≥ 3.9.0), don't hand-write it, and give it a
   scenario. Deleting an entry, unassigning someone or revoking a grant is a removal. A row
   left behind is a stale grant or a deleted thing that still counts.
6. **Automations react to events and dispatch commands.** They are idempotent. Their
   target is the stream a field of the triggering event names, and the model says which
   field (`targetIdField`, eventmodelschema 5.0.0; the command declares it as its identity):
   the same stream a command sent directly with that id reaches, never a random id or one
   made up from the event's own id. They never write read models. When an automation needs state (running totals, thresholds,
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
   write route, hand-written ones included, runs command authorization. A command that
   declares no permission is open to every signed-in caller, so where anyone can sign up,
   every command says who may send it (`requiredRole` at least), and each machine sender
   has its own role, not a superuser's. Both platforms enforce the declarations
   (pocketcqrs ≥ 0.18.0, dotnetcqrs generated hosts ≥ 0.36.0).
10. **The model comes first, and a test must be able to fail.** Change the event model
    before the code. Write the scenario first and watch it fail. Fixtures must look like
    real data (real date formats, other people's rows present), or the test proves nothing.
    Verify runs a command scenario's authorization before its decider, over read models
    built from its given events, and an error scenario passes only on its own message: give
    a command `scope` its `message` (schema 4.0.0) so a refusal scenario can name it.

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
| add a command with no `requiredRole` (or other permission) | declare who may send it; test a caller without the role is refused |
| give a scheduled job or intake a superuser token | give it its own account and role, and a long-lived token kept on its host |
| test eventual state with a sleep or polling loop | await the engine's catch-up once (a test-only barrier), or subscribe |
| put a cron expression, webhook URL, signature scheme or timer service into the model | model the command the source sends; configure the source in surface tooling |
| change a generator or the schema so it can carry a timer, schedule, ingress or effect | stop: apply the boundary test, and ask the user |
| read the clock in a decider to see whether something is due | carry "today" on the command; store the deadline in the stream |

## Runtime notes

- **pocketcqrs:** the runtime enforces decider purity (bindings removed). Projection-owned
  collections reject direct writes. Live views are built in: PocketBase realtime
  subscriptions (SSE) on projection collections. Access is PocketBase collection rules. See
  `pocketcqrs-domain`.
- **dotnetcqrs:** deciders are pure by construction. Access rules and removals are generated
  from the schema. The engine publishes a read-model change feed (`IReadModelChangeFeed`),
  the basis of generated live routes. See `dotnetcqrs-domain`.

- **Both:** an application's domain lives in the application's own repository, never in the
  platform's source. It depends on the platform as a released version (a Go module and tool;
  NuGet packages and a `dotnet tool`), with one version pinning the generator and the
  libraries. The generated code is committed and never edited. A checkout is only for trying
  an unreleased platform version.

## Reference

- `references/anti-patterns.md`: each anti-pattern, its symptom, the real case where it
  happened, and the fix. Includes the boundary ones (a source modelled in the graph; a generator
  changed to carry a source).
- `references/personal-data.md`: the GDPR crib sheet. How to classify fields as PII, read
  encrypted and erased data, govern erasure, and run the key service, plus what each application
  still has to decide. Read it before modelling any field about a person, and before shipping
  anything that stores personal data.
