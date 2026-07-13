# The Playbook

The methodology itself, in nine chapters. This is the *why and how* — the
copy-paste artifacts live in [`agents/`](../agents), [`prompts/`](../prompts),
and [`templates/`](../templates).

**New here?** Read in this order: the
[case study](../docs/case-study.md) (why this exists) → chapter
[01 — the pipeline](01-pipeline.md) (the core mechanism) → the
[getting-started guide](../docs/getting-started.md) (do it). Everything else
is reference.

| # | Chapter | One line |
| :--- | :--- | :--- |
| 00 | [Overview](00-overview.md) | The operating model and the three load-bearing ideas |
| 01 | [The Pipeline](01-pipeline.md) | Six stages, four commit checkpoints, and the hotfix lane |
| 02 | [Roles](02-roles.md) | The nine agents: tools, model tiers, and why each exists |
| 03 | [Communication Protocol](03-communication-protocol.md) | Git ownership, verdict formats, BLOCKED handling, escalation, approval gates |
| 04 | [Bug Patterns](04-bug-patterns.md) | Five recurring failure modes and how to detect them mechanically |
| 05 | [Phantom Checks](05-phantom-checks.md) | Eight grep recipes that catch features that exist but were never wired |
| 06 | [The Memory System](06-memory-system.md) | How agents accumulate knowledge across sessions without bloating |
| 07 | [Standing Routines](07-standing-routines.md) | Scheduled reviews that catch what per-change gates cannot |
| 08 | [Field Notes](08-field-notes.md) | Hard-won operational lessons, stated as symptom → diagnosis → rule |

Chapters 00–03 define canonical vocabulary (stage names, checkpoint letters,
verdict strings) used verbatim by every agent template. If you rename anything,
rename it everywhere — the verdict strings are machine-parsed contracts, not
prose.
