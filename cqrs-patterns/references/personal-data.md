# Personal data and GDPR: crib sheet

What any system built on dotnetcqrs or pocketcqrs has to get right about personal data. It covers
what the platform gives you, what the application still has to decide, and the traps already hit.
Stack-specific APIs are in `dotnetcqrs-domain` and `pocketcqrs-domain`, under "Personal data and
erasure".

**Not legal advice.** This fixes the shape of the machinery. Which data is kept, for how long, and on
what lawful basis is a business and legal call for each system, made with whoever advises it.

## What the law pushes on

| Duty | What it means here |
|---|---|
| Right to erasure (GDPR Art. 17) | A person can ask for their data to go. An append-only log can't delete, so it has to become unreadable instead. |
| Duties to keep records (payroll, tax, employment, invoices) | Art. 17 itself exempts data kept to meet a legal obligation. Much of a business system is **not** erasable on request until its retention period ends. |
| Right of access (Art. 15) | The data has to exist and be readable to answer a subject access request. Erasing too early breaks this. |
| Security of processing (Art. 32) | Encryption at rest limits what a leaked database or backup exposes. It applies even where nothing is erasable. |
| Pseudonymised is not anonymised | A hash or token of an email is still personal data. A plain hash of a low-entropy value can be reversed by dictionary attack. |

## The platform's answer: crypto-shredding

- Each data subject has their **own key**, held in a key service outside the event store. Personal
  fields are encrypted inside the event under that key.
- **Erasing a subject destroys the key.** The events stay and replay, but the personal fields can
  never be read again. The log is never rewritten.
- Applications and generated code call a **thin facade** over the key service, never the backend
  directly. Today there are two backends: Vault, and an embedded SQLite backend for development,
  demos and small single-host use. Swapping one for the other means re-encrypting, because data
  written under one can't be read by the other.
- The cross-stack rules are in `lab/cqrs-system-contracts`: `contracts/subject-erasure.md` (frozen at
  1.0), and `contracts/events-read.md` section 6 (an erased subject's data is unreadable, not
  removed).

## Modelling: do this in the event model first

1. **Mark every personal field `pii: true`** and name its `piiSubject`, a sibling field holding the
   subject's id. **When in doubt, it's PII.** That includes:
   - direct identifiers (name, email, payroll or customer numbers);
   - pay and cost figures, *and values derived from them* (an amount that is rate × hours discloses
     the rate);
   - **free text** (descriptions, notes, comments, generated messages), which can name anyone or
     anything.
2. **Every event that carries a subject's personal data must carry that subject's id.** An update
   event that only has `entryId` can't name its `piiSubject`. Add the subject id to the event. The
   decider already knows it from state.
3. **The actor is not the subject.** "Flagged by X" about Y's entry is encrypted under Y's key. If an
   event needs both, it carries two ids.
4. **Subject ids are opaque.** Never derive them from an email or name, never reuse them. A person who
   is erased and comes back is a **new** subject with a new key.
5. **One subject per partition.** The same person in two systems is two subjects. A system passes an
   erasure on as a *request*, which the receiving system's own retention check rules on.
6. **Things that are not PII:** opaque ids, business entities (customers, projects), commercial
   rates, and counts or hours needed in clear for rollups. Decide these knowingly and write the
   reason down.
7. **Erasure reasons and other governance data are stored in the clear and outlive the erasure.**
   Use codes (`asked-by-subject`), never personal data.
8. **Declare who may ask:** `dataSubjects.erasure { self, roles, via }`. This authorises
   `RequestErasure` **only**, never `EraseSubject`. "Delete my account" is a request.

## Reading personal data

- Read models hold **ciphertext envelopes**. A query route or host reveals values through the key
  service per page. A batch reveal and a reveal cache keep that cheap.
- **Erased values read as `{"$redacted": true}`.** Every client must handle it: show a single
  "Erased" marker, never `[object Object]` and never a 500. Test a screen with an erased person in it.
- **Search and filter on PII needs a blind index** (a keyed HMAC) or a `match` filter. `WHERE email
  = ...` stops working once email is encrypted. Login by email goes through the blind index.
- **Delete a subject's blind index at erasure.** Kept, it still identifies someone who asked to be forgotten. The same
  address may then sign up again, as a new subject (the key service won't recreate the old key); say so in the privacy
  policy (the user, 2026-11-16).
- **Sums over encrypted values are revealed and summed in the host at read time.** Don't store a
  clear aggregate, which would leak what the encryption protects.
- **Replay and rebuild must tolerate erased subjects.** A projection that meets an unreadable field
  carries on. It doesn't halt.
- **Live views:** register the PII read models with the cache evictor, so an erasure pushes the
  redacted result to open screens.
- Hand-written code that reads cost, pay or names from a read model and assumes clear text breaks
  silently on retrofit. Find every such reader before switching encryption on.

## Erasure

- **A request is not an erasure.** `RequestErasure` records `ErasureRequested`, and nothing is
  destroyed. `EraseSubject` records `SubjectErased`, which destroys the key. `Erased` is terminal.
- **Erasure is governed, not automatic.** The opt-in retention module (dotnetcqrs
  `DotnetCqrs.Retention`, pocketcqrs `retention`) holds a request while a retention duty stands,
  blocks it under a legal hold, and reviews held requests on their due date through the outbox, not a
  timer. A system with nothing to retain wires an allow-all policy, so a request is carried out at
  once.
- **It fails closed.** The gateway refuses every `dataSubject` command until the host wires
  authorization. The generated authorization admits only the declared `RequestErasure`. Holds,
  approvals, legal holds and direct erasure are the host's own rule. **Never return true for a
  `dataSubject` command you have no rule for.**
- **Writes after erasure are refused** (`410 Gone` at the gateway). Nothing new is stored under an
  erased id.
- **Partial erasure is not built.** Retention classes (shred the erasable fields, keep the retained
  ones until expiry) need two keys per subject. Until then an erasure removes the whole person, and
  the retained/erasable split lives in the system's own documentation and policy.
- Use the stack's `AcceptedCommands(state)` to decide which erasure buttons a screen shows. Don't
  re-implement the stage rules in a UI.

## Operations

- **The key service is critical infrastructure.** A host that stores personal data refuses to start
  without it. It won't fall back to plaintext. Every deployment, compose file and runbook needs the
  key service beside the app, with **its own volume, kept apart from the app's data**.
- **A backup must never undo an erasure.** Restoring keys from before an erasure brings the person
  back. Keep the erasure ledger **off-host**, and after any restore reconcile against it (destroy
  every key the ledger says was destroyed) before serving traffic. Rehearse the real restore and
  reconcile path, not a copy of its steps. Vault: `ops/vault/vault.sh reconcile`. Embedded backend:
  `kms reconcile [ledger copies...]` (idempotent, so it can run at every start, before the facade
  serves; put the off-host copy where it reads it after a restore).
- **Embedded backend:** its master key sits inside the same trust boundary as its database. Keep the
  master key out of the database's backups: give it as `KMS_EMBEDDED_MASTER_KEY` from a host secret
  rather than a file on the data volume. A wrong key is refused at start. There's no HA, audit log or key rotation. It suits
  development, demos and small single-host use; choose deliberately before real data depends on it.
  The operator's policy (the user, 2026-11-16): a sealed copy of the master key in two places the operator
  controls; rotate it (re-wrap every subject key) only on a suspected leak or when someone who knew it leaves; build and
  test the rotate step before the first real customer.
- **Vault backend:** keys are out of the application host's reach. It adds unseal on restart, TLS,
  scoped access policy, and snapshot/DR that must honour the erasure ledger.
- **Demo and test data:** an erasure demo uses up its subject for good. Start each run from a fresh
  seed, and date seed history far enough back for a retention policy to approve.

## What each application must decide (not engineering's call)

- Which fields are **retained** and which are **erasable**, and the retention period for each class.
  Any built-in default period is a placeholder until the business confirms it.
- Who may **ask** for erasure (self, which roles), and who may **hold, approve or erase directly**.
- How long erasable data is **kept at most** (storage limitation), and what erases it then. A
  period is a source's job, outside the model: a scheduled job reads a read model as context and
  sends the governed erasure, on its own role. The date arrives on the commands (`statusSince`),
  never from a decider's clock. The privacy notice states a period only once something enforces it.
  Worked example: landing-pages `retention/` (30 days unconfirmed, 12 months otherwise).
- Whether supporting data (free-text descriptions behind invoices, expense notes) is retained for the
  invoice's period.
- The lawful basis for each kind of personal data, and what the privacy notice tells people.
- Which key-service backend real data lives on, where its store is kept, and how it is backed up.

## Starting a new system

1. Classify every field in the event model (the rules above). Write the decisions down, with the
   retained/erasable proposal marked as a proposal.
2. Add subject ids to every event that needs one, before any data exists.
3. Declare `dataSubjects.erasure`. Wire the retention module, or an allow-all policy if nothing is
   retained.
4. Run with the embedded key service locally. Put the key service in the deployment from day one.
5. Test the reads with an erased subject present: every screen, every export, every rollup.
6. Decide the backend and the backup/reconcile story before the first real person's data goes in.

## Traps already hit

| Trap | Symptom | Fix |
|---|---|---|
| A generated or default gateway allowed any command with no declared policy | Every signed-in user could erase anyone | Fail closed for `dataSubject`; authorize each command explicitly |
| Mapping a document's erasure declaration straight to `EraseSubject` | Self-service erasure skips the retention check | Declarations authorise `RequestErasure` only |
| Clients rendering the redacted marker as a value | `[object Object]` where a name was; a 500 on a pay view once anyone was erased | Map `{"$redacted": true}` to one marker everywhere; test with an erased subject |
| Update and flag events without a subject id | No `piiSubject` to encrypt under | Put the subject id on the event |
| Login and filters comparing on email | Nobody can sign in after encryption is switched on | Blind index for lookups; `match` filters for search |
| Restoring the key store's folder | Erased people readable again | Off-host ledger and reconcile after every restore |
| An erasure screen polling after its commands | Flaky, slow UI | Show the command's outcome; let the live view push (`anti-patterns.md`) |
| A deployment guide assuming one container | The app won't start: no key service | Key service as its own service and volume, next to the app |

## Background

The reasoning lives in the notebook repo's `thing/venture-overview` decisions: `0010` (why
crypto-shredding over pseudonymised side tables or rewriting the log), `0011` (Vault behind a facade),
and `0013` (erasure governed by retention; stages 1 and 2).
