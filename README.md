# cqrs-skills

Claude Code skills for event-sourced CQRS work:

- `cqrs-patterns`: the paradigm (one-way data flow, the boundary where messages enter and leave, where invariants live, read models, automations, live views). Read first.
- `dotnetcqrs-domain`: workflow for dotnetcqrs and hosts built on it.
- `pocketcqrs-domain`: workflow for PocketCQRS projects.

## Install

Each skill folder is linked into `~/.claude/skills`. On Windows, a junction needs no admin rights:

```powershell
$repo = '<this clone>'   # wherever cqrs-skills is checked out on this machine
foreach ($s in 'cqrs-patterns','dotnetcqrs-domain','pocketcqrs-domain') {
  New-Item -ItemType Junction -Path "$HOME\.claude\skills\$s" -Target "$repo\$s"
}
```

`pocketcqrs` also ships its own `pocketcqrs-domain` (`.claude/skills/` in that repo, embedded in
the binary as `pocketcqrs skill install`); that copy is the one its users get.

## Keeping them current

A platform release that changes what the generator honours (for example dotnetcqrs 0.27.0's
`requiredOwnership.stateField`) updates `dotnetcqrs-domain` here in step with the release.
