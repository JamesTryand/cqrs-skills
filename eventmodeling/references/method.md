# The method, step by step

Sources: Adam Dymitruk, "What is Event Modeling?" (eventmodeling.org, the seven steps); the
eventmodeling.org cheat sheet and posts on human thinking, traditional systems and user experience;
Nebulit's cheat sheet as recorded in eventmodelschema's `docs/methodology-notes.md`; the user's own
practice, hub decision `0008` (bounded state machines). "Mine" marks this skill's own suggestion.

## Why a story

People remember a long story step by step through time, and Event Modeling follows that: no
technical knowledge is needed to read one. It is weaker than Event Storming's design level at
surfacing explicit rules, which is why step 6b (the state machines) is added: it finds the rules a
story skips.

## Step 0: goal and scope (mine)

Ask what the system is for, and where the story starts and ends. "Start small" (Nebulit). Say the
goal back in one paragraph and wait for a yes.

## Step 1: brainstorm the events

Only facts that change the system's state count. Name each as a past-tense business fact in the
user's words: "Guest Checked Out", not "CheckoutProcessed" or "UpdateRoom". Model facts, not ideas
(Nebulit). Propose a list; the user corrects and adds.

## Step 2: the plot

Arrange the events into a plausible story, left to right in time. Show it as a text timeline:

```
Room Night Booked -> Guest Checked In -> Guest Checked Out -> Bill Closed
                                                           -> Room Cleaned -> Room Ready
```

Ask "is that the order?" and "what happens next?" (Nebulit's key question).

## Step 3: the storyboard

Name each screen and who uses it, and list the fields it shows. Every field shown must come from
somewhere. The schema keeps a screen as a name only, so this list lives in the conversation: the
user confirms it, and step "Completeness" checks it.

Each step on the timeline has **context** (what the person needs to know) and **measures** (what
they can do). A gap in context shows up when the flow is drawn in time order (the UX post).

## Steps 4 and 5: inputs and outputs

Put a **command** before each event: how someone (or something) changes the state. Ask: why would
anyone send it? That is its `reason`.

Put a **view** after the events a screen needs: how the information gets back out. Ask: what
question does it answer? That is its `question`. A view can show only information that already
exists, so missing data shows up here.

## Step 6: swimlanes (Conway's law)

Group the events into lanes that separate teams (or systems) can own. A lane is `team` or `system`.
If two lanes are really separate systems, with separate owners, deploys or stores, that is the cue
for `systems.md`.

## Step 6b: a state machine for every aggregate (decision 0008)

Before scenarios, each aggregate gets a table: one row per state, one column per command, every cell
filled with what happens (the event, or the refusal and its message). Every aggregate, even a
two-state one: the small ones are where gaps hide (the user, 2026-11-16).

Four checks (the hub's `statemachine_check.py` checks most of them):

1. **Totality**: every state-and-command pair has an answer.
2. **Reachability**: every state can be reached from the start.
3. **Termination**: a terminal state refuses everything, by name.
4. **Event grounding**: every event a cell names exists, and every event is produced somewhere.

A cell that says "nothing happens" is a trap: make it a named refusal the sender treats as done.

## Step 7: scenarios

One Given/When/Then per rule, each tied to exactly one command or one view (Dymitruk). Refusals are
rules too: each `error` scenario names its message. Use realistic data: real-looking ids, dates as
`yyyymmdd`, and other people's rows present in a view scenario so the scenario can fail. Ask three
times: "is there any rule we didn't cover?" (Nebulit).

## Completeness

"Every field accounted for" (Dymitruk): trace each field on each screen, command, event and view to
an origin and a destination. A field with no origin is a missing command or event; one with no
destination may be unneeded. Unknowns become hotspots, shown to the user, never guessed.

## The boundary check

Apply `cqrs-patterns`' boundary test to every message that enters or leaves: the model names the
message and its meaning, never its source. See `boundary.md`.

## The review checklist (judgement calls; generators check the rest)

Checks a generator can make go into that generator's own document check (the user, hub I8). These
stay with the person:

- Does every command have a reason, and every view a question?
- Is every event a business fact, in the user's words?
- **Left chair** (one command, many events), **right chair** (one view fed by very many events),
  **bed** (one screen, many commands), **shelf** (a slice with no scenarios): Nebulit's
  anti-patterns, to look at, not forbid.
- Is any rule enforced by reading a view? Move it into the stream that owns it.
- Does any automation need a fact that isn't on its trigger event?
- Is any message's source in the model?
- Does every hotspot have an owner who will answer it?

## Nebulit's rules worth leaning on

Start with events; model facts, not ideas; time flows left to right; every command has a reason;
every read model answers a question; don't guess: unknowns become hotspots; keep slices small; a
slice is the smallest piece of work a developer can be handed (the cheat sheet).

Two cheat sheets exist; keep them apart. The eventmodeling.org web cheat sheet draws a translation
with a command. Nebulit's PDF draws it as event, read model, event, with no command; eventmodelschema's
design notes call that diagram an error, and removed `translation` as a pattern in schema 2.0.
