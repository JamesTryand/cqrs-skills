# Lessons from real models

What real models taught, each with where it came from. Kept apart from the method so it can grow. The
landing-pages references are to `lab/landing-pages` (`model/`) and the notebook's
`project/landing-pages` (`STATE-MACHINES.md`, `UPSTREAM.md`).

- **No "succeed without an event".** A harmless repeat (a second notification for the same payment, a
  second check-out) is a named refusal the sender treats as done, never a silent success: otherwise
  the log can't tell "done" from "ignored". (landing-pages `STATE-MACHINES.md`.)
- **Dates are whole numbers, `yyyymmdd`** (`20271204`), so "later than" is a comparison a decider can
  make. Generators report "integer becomes number"; that is expected. (`STATE-MACHINES.md`.)
- **A decider can't read the clock, so "today" is on the command**, with any outside answer; the
  stream records the deadlines it compares against (`graceEnds`, worked out by the sender from the
  grace setting). (`STATE-MACHINES.md`, the entitlement notes; `boundary.md` here.)
- **A command that both starts a stream and acts on an existing one is an upsert**: pocketcqrs treats
  a command with both kinds of scenario that way. (`STATE-MACHINES.md`: `GrantSubscription`,
  `GrantPerpetual`.)
- **A view scenario's caller needs a `subjectId`**, even one whose role sees every row. Six
  scenarios in landing-pages' personal-data model gave only a role; pocketcqrs didn't check, dotnetcqrs
  refused the document. Found 2026-11-17, fixed 2026-11-18; pocketcqrs checks the schema from v0.22.0.
- **An automation's command gets the trigger event's fields by name.** In the hotel example, a field
  named differently on the command (`closedOn` against the event's `checkedOutOn`) arrived empty. Name
  the command's fields as the trigger names them, and put every fact it needs on the trigger.
  (`worked-example.md`, 2026-11-18.)
- **An outcome's `set` takes literal values only.** To put a value from the stream's state on a new
  event, the command carries it. (The hotel example's check-out.)
- **Personal fields and an outcome's `set` don't mix on pocketcqrs' Go target** (`UPSTREAM.md` L-008,
  filed with pocketcqrs, not fixed yet): a command with `pii` fields whose slice `set`s a field fails
  its scenario. Worked around in landing-pages: a freshly captured contact has no status, and the
  rules read "no status" as captured.
- **Commands sent by an automation or an outside system were drawn on an operator screen** in
  landing-pages' commerce model (`GrantSubscription`, `GrantPerpetual`, `RecordRenewalPaid`,
  `RevokeEntitlement`, and the processor's messages), while `STATE-MACHINES.md` says automations and
  the processor send them. The user's answer (hub L10): split by sender. Our own automation: model an
  automation, which pocketcqrs generates. An outside system: an outside input with no screen
  (`boundary.md`), which waits for the schema change. The commerce model is to be corrected the next
  time it is worked on.
- **Two systems, one coordinating document.** landing-pages' personal data and commerce are separate
  systems (personal data is encrypted and erasable, commerce knows a person only by id). Their
  coordinating document passes the checker: the contact's email never crosses. (`systems.md`;
  landing-pages `model/landing-pages.architecture.json`, 2026-11-18.)
