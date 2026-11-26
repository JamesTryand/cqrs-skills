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
EMS="https://raw.githubusercontent.com/jamestryand/eventmodelschema/v6.2.1/schema"
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
the fields the event publishes: those its public marker lists (6.2.0), or else those some internal edge from
it maps. Not either runtime's own event record, so a system on either runtime can be on either end. Its `system`
is the sender's **document id** (coordinating document 0.3.1), never the name a view gives it: a view may call
the system anything, and the generated receiver turns that name into the id. Renaming a view's key changes
nothing on the wire; changing a document's `id` does, so treat a document id as a published name.

**Generating it** (pocketcqrs v0.28.0, Go target; dotnetcqrs 0.45.0):
`schema import <doc> --lang go --out domain --architecture <coordinating doc> --system <id>` (dotnetcqrs:
`generate ... --architecture ... --system ...`) writes the system's edge tables: the edges into it
(`EdgesIn`) and the events it publishes (`EdgesOut`). A failing example, an edge's event or command that is
not public, a target id of several fields, or a personal field on an edge refuses the run. A system linked in
several connecting documents (pocketcqrs v0.34.0, dotnetcqrs 0.53.0) gives `--architecture` once per document,
and `--system` once for all, once per document, or not at all (each document's system is the one that is this
document, by its release id or path); its edges come from all of them, and two documents giving one edge id
different ends or translations are refused.

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

## Systems in their own repositories (EventModel 6.2.0, coordinating document 0.3.0)

When systems live in separate repositories and release apart, each system's document is the authority on that
system, and a **connecting document** in a repository of its own is a view of how some of them work together. Several
views may hold at once, as long as they do not contradict each other.

- **Name a system by release**: `"release": { "system": "<its document id>", "tag": "<its release tag>" }`, beside
  `document` (a path, for systems in the same repository) and `placeholder`. Where each repository is stays out of the
  document: the checker takes the installation's own map (`--systems <file>`), which lists each system's checkout and
  document path, and the views (`"views": [...]`) the installation holds.
- **Say what leaves**: a public event's marker lists the fields it publishes,
  `"public": { "version": 1, "fields": ["memberId", "branchId"] }`. An edge may read only those, so a sender's build
  needs only its own document (generate with no coordinating document). A system named by release must list them.
- **Say what you rely on**: a receiver's `expects`, keyed by the other system's document id, with `writtenAgainst`
  (that system's release tag) and the events it uses, each with only the fields it reads. It is checked against the
  sender wherever it is stated; a system named by release states one for every edge into it. A tag behind the one in
  use is a note to re-check, not a failure. A system never expects from itself.
- **A message that means nothing to its receiver** is accepted with nothing recorded (an outcome with no events, and a
  scenario with `then: {"events": []}`), not refused, so a refusal keeps meaning "this should not have been sent".
- **How far a check reaches**: a check judges the systems it names, at `--reach short` (the default: those systems,
  reading only the documents that feed them), `full` (and everything they depend on) or `complete` (everything
  connected). With nothing named, the whole installation. A stale view (one naming an older release of a system
  another view reads newer) is checked against the newer document and ratified with a note when its links hold.

The worked example is eventmodelschema `schema/examples/links/` (two systems, each in its own repository, and one
connecting document; `npm run test:links`) and `schema/examples/reach/` (six systems in two views).
