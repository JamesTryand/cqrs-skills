# What enters and leaves the model

The rule is `cqrs-patterns`' boundary section; read it there, so the two can't drift. This file is how
the method's patterns sit against it, and what the schema does and doesn't carry yet.

## The rule, in one line

The model is a directed graph. Messages cross its edge in three shapes only, and the model names the
message and its meaning, **never its source** (where it comes from, or how it is carried). Sources
(schedulers, webhooks, queues, URLs, signatures, credentials) are configured by surface tooling
(`sctl`), never in the model or a generator. If something seems to need a source in the model, stop
and ask the user (hub decision `0033`).

| shape | in the model |
|---|---|
| a call out and its answer | an automation's call; the answer and the give-up both come back in as commands |
| an outside input (a message enters, nothing goes back) | a command no screen and no person sends ("payment arrived", "a period has ended") |
| an outside output (a message leaves, nothing comes back) | a message that leaves after an event |

## Settled (the user, 2026-11-14 and 15, hub `QUESTIONS.md` I1-I8)

- **A delay's length is model data** ("grace lasts 7 days"); only firing the timer is a source's job.
- **No cancelling a delay by events**: the decider's own checks make a late message harmless.
- **No "time passes" step in a scenario**: commands carry today's date, so a scenario sends a later
  date. A decider never reads the clock, in tests either.
- **A message that leaves with no answer is a shape of its own.** A call out keeps its required
  give-up answer.
- **A schedule's `cron` and time zone, and an ingress's `verification` and `deliveryIdField`, move out
  of the document**: they are source details.
- **Two documents, joined by message ids**: the event model says what crosses and what it means;
  surface tooling's description says how and where, and refers to the model's messages by id, never
  the reverse.
- **No new lint tool**: checks a generator can make go into its own document check.

## In the schema (6.0.0) and the generators (pocketcqrs v0.25.0, dotnetcqrs 0.42.0)

- **An outside input** is an `ingress` (a name, a description, a swimlane) on a `stateChange` slice.
  Its command is generated like any other; the ingress itself generates nothing. Something started by
  time is one too: its command carries today's date. The hotel example's "payment arrived" is the
  `card-processor` ingress.
- **An outside output** is an `outputs` entry: the events after which it leaves, and the fields that
  leave (personal data shows). Nothing is generated; the generators check each field is on every
  trigger event. landing-pages' confirmation mails are outputs.
- **A call out** is an automation's `effect`; **a wait** is its `delay` (the length only). Their
  commands are generated, the answer and give-up for a call, but no reactor: the source fires them.
- **Never put an outside input on a stand-in screen** as if a person sent it (the user, hub L5).
  landing-pages' commerce model did; corrected 2026-11-18 (hub L10).

## How the method's patterns land

- **A translation from another of our own lanes**: an automation across lanes, fully in the graph. It
  generates today.
- **A translation from another of our own systems** (its own document): a link in a coordinating
  document (`systems.md`).
- **A message from outside**: an outside input (above). Don't store the sender's payload as a foreign
  event: personal data lands in the log, and anyone who can reach the sender's address writes to the
  store (eventmodelschema design notes, 3.4.0).
- **A call out and its answer** (the method's own example is a payment call at checkout): an
  automation's call; the answer and the give-up come back as commands.

## Time arrives on the command

A subscription's renewal is due on a date. The model doesn't say "a timer fires at midnight" (that
is a source); it says what the command means: "Record Period Check (customer c-1, today 20271108)".
The stream already holds the date the subscription is paid until, so the decider compares two values
it has and refuses or records. Whoever sends the command, and when, is surface tooling's business.
landing-pages' commerce model does exactly this (`RecordPeriodCheck`).
