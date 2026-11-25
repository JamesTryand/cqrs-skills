---
name: eventmodeling
description: "Event Modeling as a method, done with the user, ending in an EventModel document (eventmodelschema 6.2.0) that pocketcqrs or dotnetcqrs generates and verifies: brainstorm the events, plot them as a story, storyboard the screens, find the commands and views, swimlanes, a states-by-commands table per aggregate, Given/When/Then scenarios, the completeness check, the boundary check, then the JSON. Also when one model becomes several systems: public events and commands, and a coordinating document checked across them. Use when designing or changing a domain, running an event-modelling workshop, writing or reviewing an EventModel document, slices, scenarios or swimlanes, or splitting a model into systems. Architecture rules are in cqrs-patterns; generating and running code is in pocketcqrs-domain and dotnetcqrs-domain."
---

# Event Modeling, with the user

Event Modeling describes a system by an example of how its information changes over time: a story
told in events, with the commands that cause them and the views that read them. It was developed
by Adam Dymitruk (eventmodeling.org). It has three moving pieces (commands, events, views) and four
patterns (command, view, automation, translation).

**The user decides; Claude proposes.** The method is a conversation. The EventModel document is its
record, and it comes **last**: nothing becomes JSON until the user has seen the story in plain form
and corrected it.

## The workflow

Each step ends by showing the user something and waiting for a yes. One decision at a time, in the
user's words.

| step | Claude does | show the user |
|---|---|---|
| 0. Goal and scope | ask what the system is for, where it starts and ends; start small | one paragraph |
| 1. Brainstorm | propose events as past-tense business facts ("Room Night Booked") | a plain list |
| 2. Plot | order them into a story | a left-to-right text timeline |
| 3. Storyboard | name the screens, per person, with the fields each shows | screen: fields lists |
| 4-5. Inputs and outputs | a command before each event, a view after; ask "why would anyone send this?" and "what question does this view answer?" | the timeline with commands and views |
| 6. Swimlanes | propose who owns which events (teams or systems: Conway's law) | the timeline grouped by lane |
| 6b. State machines | for **every** aggregate, a states-by-commands table, every pair filled | the table and its four checks |
| 7. Scenarios | one Given/When/Then per rule, refusals included; ask three times "is there a rule we haven't covered?" | scenarios in prose |
| Completeness | trace every field to where it comes from and where it goes | the gaps; each unknown becomes a hotspot |
| Boundary | apply the boundary test to everything that enters or leaves | which messages cross, with no sources |
| Systems | if lanes are really separate systems, split the model (`references/systems.md`) | the systems and the links between them |
| JSON | write the document; run the checks below | the generator's report, read out |
| Review | the checklist in `references/method.md` | anything flagged |

The detail of each step, with the questions to ask, is in `references/method.md`. The four patterns
are in `references/patterns.md`.

## Rules for the session

- **Never invent a rule.** A missing answer is a hotspot in the document, shown to the user.
- **Show the timeline before any JSON**, and correct it in plain language.
- **Stop and ask before anything outside the document**: a schema or generator change, or any
  repository the user didn't name.
- **Read the generator's report out.** It names every decision taken on the document's behalf.

## Traps (each from a real case)

- **Writing JSON first.** The user can't correct a story they haven't seen.
- **A read model that answers no question, a command with no reason.** Give each its `question` or
  `reason`; if there isn't one, it probably shouldn't exist.
- **Modelling a source.** A scheduler, webhook, queue or provider is where a message comes from, not
  part of the model. Apply the boundary test in `cqrs-patterns` (and `references/boundary.md`).
- **Changing a generator or the schema to fit the model.** Stop and ask the user.
- **The automation's to-do view.** Draw it with the user, because it is the method. But the generated
  reactor does not read it, and must not: put the facts the next command needs **on the trigger event**,
  under the names the command uses (the reactor copies the event's fields into the command by name),
  and let the target decider refuse what doesn't hold.
- **A decider can't read the clock.** "Today" arrives on the command, as a `yyyymmdd` integer a
  decider can compare; the stream stores the deadlines it compares against.
- **No "succeed without an event".** A harmless repeat becomes a named refusal ("this guest has
  already checked out") that the sender treats as done.
- **An untagged command or event.** pocketcqrs refuses it: give every command and event its
  `aggregate`.
- **A scenario that can't fail.** A view scenario with only the caller's own rows proves nothing about
  who sees what; give it someone else's rows too. A view scenario's caller needs a `subjectId`.
- **An invariant in a view.** "Don't overbook" lives in the stream that owns the room's night, never
  in a read model (`cqrs-patterns` rule 2).

## Check the document

Every command below has been run. `<version>` is a released pocketcqrs (v0.21.0 or later).

```sh
# the scenarios, against the code pocketcqrs generates (add --lang go for personal fields)
go run github.com/jamestryand/pocketcqrs@<version> schema verify model.eventmodel.json

# the full report: what was generated and every decision taken on the document's behalf
go run github.com/jamestryand/pocketcqrs@<version> schema import model.eventmodel.json --out <scratch> --force
```

Schema 6.0.0 needs pocketcqrs v0.25.0 or dotnetcqrs 0.42.0 or later (5.0.0's `targetIdField`; 6.0.0's
ingresses with no source details, `outputs`, no `schedule` or `elapsed`). A slice with neither a screen
nor an ingress (6.1.0: another system sends its public command) needs pocketcqrs v0.27.0 or dotnetcqrs 0.44.0.
6.2.0 (a scenario accepted with nothing recorded, `expects`, the fields a public event publishes) needs
pocketcqrs v0.32.0 or dotnetcqrs 0.51.0; older releases refuse the document. A message names its sender by the
sender's document id, and a sender publishes from its own document alone, from pocketcqrs v0.33.0 and dotnetcqrs
0.52.0 (coordinating document 0.3.1).
pocketcqrs from v0.22.0 checks the document against the schema before anything else, as dotnetcqrs
does, and names each problem's place. With an older pocketcqrs, validate first from an
eventmodelschema checkout's `schema/` folder:
`npx ajv validate -s eventmodeling.schema.json -d <document> --spec=draft2020 -c ajv-formats`.
dotnetcqrs: `references/generators.md`.

## References

| file | holds |
|---|---|
| `references/method.md` | each step, the questions to ask, Nebulit's rules, the state-machine step, the review checklist |
| `references/patterns.md` | command, view, automation, translation: shape, Given/When/Then, what the schema calls each |
| `references/schema-mapping.md` | method concept to schema key, and what each has that the other doesn't |
| `references/boundary.md` | what enters and leaves the model (decision 0033), as settled, and what today's schema still lacks |
| `references/systems.md` | one model as several systems: `public`, the coordinating document, its checker |
| `references/generators.md` | what pocketcqrs and dotnetcqrs do with each part, and how scenarios are checked |
| `references/worked-example.md` | the hotel stay, step by step, ending in `examples/hotel-stay.eventmodel.json` |
| `references/lessons.md` | what real models taught, each with its source |

Real cases: landing-pages' two models and their coordinating document (`lab/landing-pages`,
`model/`). Rules this skill relies on but doesn't repeat: `cqrs-patterns`.
