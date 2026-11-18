# One model as several systems

When step 6's lanes turn out to be separate systems (separate owners, deploys or stores), each system
gets its own EventModel document, and a **coordinating document** says how they work together. Its
schema is eventmodelschema's `architecture.schema.json` (0.1.0, with EventModel 4.2.0).

Signs the lanes are separate systems: a different team changes them on a different schedule; they
need different stores or hosting (personal data apart from the rest, say); one could be replaced
without the other noticing. If none of these hold, keep one document with lanes.

## What a coordinating document holds

- **systems**: each is an EventModel document (by path) or a **placeholder** for one not written
  yet, which lists what it `publishes` and `accepts`.
- **edges**:
  - **internal**: an event of one system, at a version, becomes a command of another. It names the
    command's target id (`targetId`, a field of the event) and maps the command's fields from the
    event's (`fields`).
  - **external**: a message crossing the edge of the whole, in one of the three shapes: `input`
    (to a command), `output` (from an event), `call` (from an event, `answers` naming the commands
    the answer and the give-up come back as).
- **flows**: named paths through the edges ("a sign-up becomes a lead").

An edge **never names its carrier** (a queue, a topic, a URL): that is a source (`boundary.md`).

## The rules its checker enforces

1. **The receiver owns the translation.** An internal edge maps into the receiving command's fields,
   in the receiver's terms.
2. **The target id comes from the source event**, so a redelivery reaches the same stream and is
   refused as already done.
3. **Only public events and commands, at their version.** In each EventModel document, mark what
   other systems may use with `"public": {"version": 1}` (4.2.0). Anything unmarked is private and
   free to change; a public one changes only by a new version.
4. **Personal data crosses only into a system that can erase it**: one with `dataSubjects.erasure`,
   into a field marked `pii` there. Never into a placeholder.

It also checks that everything an edge or flow names exists, that flows connect, and reports each
system's coupling (edges in and out, depends on, used by). An edge from a system to itself is
refused: that belongs inside the system's own document.

## Run the checker

From an eventmodelschema checkout's `schema/` folder (`npm ci` once):

```sh
node scripts/architecture-check.js <path>/system.architecture.json          # exit 0 ok, 1 refused
```

Or pinned to a tag with nothing checked out, as landing-pages' `model/verify.sh` does:

```sh
EMS="https://raw.githubusercontent.com/jamestryand/eventmodelschema/v4.2.0/schema"
TMP="$(mktemp -d)"; mkdir -p "$TMP/scripts"
for f in package.json package-lock.json architecture.schema.json eventmodeling.schema.json scripts/architecture-check.js; do
  curl -fsSL "$EMS/$f" -o "$TMP/$f"
done
( cd "$TMP" && npm ci --silent --no-audit --no-fund >/dev/null )
node "$TMP/scripts/architecture-check.js" system.architecture.json
```

Each system's own document is still verified on its own with its generator.

## Doing it with the user

1. Agree which lanes are systems, and why (the signs above).
2. For each crossing, agree its shape: internal (which event becomes which command, the target id,
   the field mapping) or external (input, output, call).
3. Mark exactly what crosses as `public` in each document; nothing else.
4. Check personal data: does anything personal cross? Into a system that can erase it?
5. Write the coordinating document, run the checker, read its coupling report out.

## Where to look

- eventmodelschema `schema/examples/architecture/`: a library's members and lending systems, and a
  fines system not written yet (a placeholder).
- landing-pages `model/landing-pages.architecture.json`: personal data and commerce, the real case.

**Open, not settled:** in both, the command a system receives over an internal edge sits on a screen
in the receiving document, as if a person sent it. Like an outside input, no person sends it. Raise it
with the user rather than choosing.
