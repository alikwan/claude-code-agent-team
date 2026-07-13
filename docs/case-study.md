# Case Study: How This Pipeline Was Born

*The origin story, told honestly. Identifying details about the production
system's clients, infrastructure, and market have been removed or generalized —
see the disclaimer at the end.*

## The setup

One developer. One large, real system: **the source project**, an Arabic-first debt
collection platform — Laravel backend, SPA frontend, call-center telephony
integrations, omnichannel messaging, automatic escalation workflows, legal
case management, field-visit tracking, workforce management. RTL interfaces,
Arabic notifications, exact-decimal money everywhere. The kind of system that
normally employs a team.

There was no team. There was Claude Code.

## Act I — one session, growing damage

The first months were what everyone's first months with an AI coding agent
look like: astonishing speed, then a slowly accumulating tax. One session did
everything — planned, wrote, tested (sometimes), committed (chaotically).
Three failure modes kept recurring:

1. **Contract mismatches.** The frontend confidently consumed response shapes
   the backend never produced — a wrapper here, a renamed key there, an object
   where an array should be. Everything compiled; screens rendered zeros.
2. **Phantom features.** Complete, correct, tested code that was *unreachable*:
   a notification class never dispatched, a permission never enforced, a
   setting toggle nothing read, a background job never registered with the
   worker system. A human wires as they go; an AI building from a prompt
   doesn't know the app has a sidebar.
3. **Illegible history.** Commits that mixed three concerns, docs that
   described the previous version, and no way to tell which change broke what.

The realization that reshaped everything: these aren't model failures — they're
*organization* failures. A developer with no memory, no reviewer, and no fear
produces exactly these bugs. The fix isn't a better prompt; it's a team
structure.

## Act II — inventing the team

The split came first: a **team-lead** that plans and coordinates but cannot
write files (its own tool restrictions enforce the discipline), builders that
write but never touch git, and a QA agent that judges but cannot fix — because
a reviewer that can quietly fix what it finds will quietly approve what it
fixed. All git operations concentrated in the orchestrator, at four named
commit checkpoints.

Two large workforce-module PRs became the crucible. Their review cycles
produced roughly **thirty blockers** — and reading them side by side revealed
they weren't thirty problems but *five patterns* recurring under different
names. Those five became the [bug-pattern library](../playbook/04-bug-patterns.md),
baked into the QA agent as mandatory checks and into the builders as
authoring-time rules. The phantom-feature blockers spawned a dedicated
[integration agent](../playbook/02-roles.md) with eight grep-recipe checks,
because "the code is correct" and "the feature exists" turned out to be
different claims.

Migrations and security-sensitive changes got their own conditional reviewers
after near-misses — cheap-model specialists with one checklist each, spawned
only when the diff touches their domain.

## Act III — memory, routines, and the long game

Agents kept re-learning the same lessons, so they got
[persistent memory](../playbook/06-memory-system.md): version-controlled files
read at session start, appended after work — recurring issues, known weak
spots, *disproven* findings (so the same false alarm is never chased twice).
Memory changed the economics: every review made every future review cheaper.

Then came the things a per-change pipeline can't catch — drift across changes,
rot in code merged before a pattern was known. Three
[standing routines](../playbook/07-standing-routines.md) now run on schedules:
a daily per-commit correctness review, a codebase-wide recurring-pattern sweep,
and a four-surface security sweep. Because scheduled cloud agents can't read
local memory, GitHub itself became the deduplication store — labeled issues
and open PRs are the "already known" set. Findings route by severity: ordinary
issues get labels; critical ones get a pipeline-driven fix and a PR that a
human reviews before merge.

**Two hundred–plus releases** have now shipped through this machine — features,
security fixes, refactors, incident responses — with a changelog whose entries
read like engineering post-mortems because the documentation agent blocks the
PR until they do.

## What still doesn't work (and what we'd do differently)

Honesty is the point of this document:

- **The release step resists automation.** Backgrounded orchestrators
  repeatedly stalled just before commit/push/PR — which is why the human-
  operated release is the *documented design* in v1.1, not a bug we pretend
  isn't there ([field notes](../playbook/08-field-notes.md), note 2).
- **Guardrails in prose fail.** A review sub-agent once wiped a development
  database with a destructive reset that every instruction said not to run.
  The rule moved into the permission system, where it should have been from
  day one.
- **We under-invested in memory governance.** One agent's memory grew to seven
  times its declared cap before we noticed the token tax. Caps are now
  enforced, not suggested.
- **We'd define the API contract convention on day one.** Half of Act I's pain
  traces to its absence.
- **The pipeline you're reading is v1.1**, not what we ran historically —
  preparing it for publication surfaced three real protocol holes (unreviewed
  integration changes, a version/changelog ordering bug, an escalation-rule
  wording gap). They're fixed here and marked where they appear. Extracting
  your process forces you to debug it; we recommend the exercise.

## Why it's shared

This methodology was built on top of open tools, open frameworks, and other
people's shared experience. Releasing the working system back — the agents,
the protocols, the patterns, the failures — is the honest way to pay that
debt. If it ships one stuck project, it was worth publishing.

---

*Sanitization disclaimer: "the source project" is used here as the project's codename.
Client names, domains, hostnames, internal paths, credentials, vendor
integrations that could identify the operation, and private issue/PR numbers
have been removed or replaced throughout this repository. The worked example
in [examples/laravel-react/](../examples/laravel-react/) uses a fictionalized
project ("Acme Collect") on the author's current recommended stack; the
original production system ran a different SPA frontend — the methodology
carried over unchanged, which is rather the point. Quantitative claims
(releases shipped, blocker counts) were verified against the production
repository's history before publication.*
