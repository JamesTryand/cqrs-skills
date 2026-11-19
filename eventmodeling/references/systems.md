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
EMS="https://raw.githubusercontent.com/jamestryand/eventmodelschema/v6.1.1/schema"
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

**A command another system sends has no screen and no ingress** (schema 6.1.0; the user's call, hub
L14). Its `stateChange` slice names neither: the internal edge in the coordinating document is what
starts it, named once, there. The command must be `public` (the edge needs that anyway); the
generators refuse neither with a private command ("nothing starts it"), and the checker refuses one
that no internal edge reaches (check `received`). Don't put it on a stand-in screen, and don't call it
an ingress: an ingress is outside the whole, and this is one of your own systems. Generators (pocketcqrs
v0.27.0, dotnetcqrs 0.44.0) generate its command and name it in the report; what sends it is the edge,
generated (below).

## An edge's examples, and carrying it (architecture 0.2.0; hub L17)

**Give each internal edge `examples`**: given a source event's data, then the command the receiver sends
(its target id and fields, exactly). Put the fields that must not cross (an address, a name) in `given`:
the example then proves they don't. The checker runs them (check `examples`), and so do both generators.

**What crosses is the `edgeMessage`**: the source event's id, system, event and public version, and only
the fields some internal edge from that event maps. Not either runtime's own event record, so a system on
either runtime can be on either end.

**Generating it** (pocketcqrs v0.28.0, Go target; dotnetcqrs 0.45.0):
`schema import <doc> --lang go --out domain --architecture <coordinating doc> --system <id>` (dotnetcqrs:
`generate ... --architecture ... --system ...`) writes the system's edge tables: the edges into it
(`EdgesIn`) and the events it publishes (`EdgesOut`). A failing example, an edge's event or command that is
not public, a target id of several fields, or a personal field on an edge refuses the run.

**Carrying it** is the host's and surface tooling's, never the model's: pocketcqrs-extensions v0.5.0's
federation takes the tables (`PublisherConfig.Edges`, `IngestConfig.Edges`) over NATS JetStream, one user per
system, each allowed only its own stream (or only its peer's, to read). The worked case is landing-pages:
`backend/edges-e2e.sh`, and the n2 trial's `nats-trial`. dotnetcqrs (0.46.0) carries them too, with
`JamesTryand.DotnetCqrs.Edges.Nats` (`EdgePublisher`, `EdgeIngest`): the same subjects, stream, durable consumer and
command ids as federation, so either runtime can be on either end of an edge (checked both ways, through the
carrier image with its per-system users, 2026-11-19).

**Changing a public event's version** (federation v0.6.0, dotnetcqrs 0.47.0): a receiver holds a newer version than its
edges name until one names it, and skips an older one. So deploy the sender first, or keep the old version's edge in the
coordinating document until every sender has moved: a receiver deployed first, naming only the new version, skips what
an old sender still sends.
