<img src="../assets/icons/playbook.svg" width="40" height="40" alt="">

[**← playbook index**](README.md) &nbsp;·&nbsp; chapter 07 of 08

# 07 — Standing Routines

The pipeline is a *per-change* gate: it judges one diff, on one branch, at one
moment. Three classes of defect are structurally invisible to it:

- **Cross-change drift** — two individually-approved changes that disagree
  with each other (a contract updated in one PR, a consumer merged from
  another).
- **Slow rot** — docs decaying, schedules accumulating duplicates, dead flags
  piling up; no single diff is at fault.
- **The pre-pattern backlog** — everything that merged *before* a bug pattern
  was identified. New reviews check for it; the instances already in the
  codebase sit there until someone hunts them.

Standing routines are the complement: scheduled, repo-wide review passes that
run outside any feature branch. They don't replace the pipeline — they sweep
what it structurally cannot see.

## The three routines

| Routine | Purpose | Suggested cadence | Full prompt |
| :--- | :--- | :--- | :--- |
| **Daily code review** | Trace the last ~24 h of commits on `{{MAIN_BRANCH}}` for *critical correctness* bugs that slipped review — data loss, races, authz bypass, money-math errors | Daily (scheduled cloud agent) | [prompts/daily-code-review.md](../prompts/daily-code-review.md) |
| **Recurring-patterns sweep** | Re-run the [5 bug patterns](04-bug-patterns.md) + [phantom checks A–H](05-phantom-checks.md) across the whole codebase, catching pre-pattern instances | After each large merge; monthly; before major releases | [prompts/recurring-patterns-sweep.md](../prompts/recurring-patterns-sweep.md) |
| **Security sweep** | 4 parallel read-only agents, one per attack surface (authn/authz+IDOR, injection+files, webhooks+SSRF, secrets+XSS+mass assignment), full-repo | Biweekly or monthly; immediately after auth/webhook/upload/payment changes | [prompts/security-sweep.md](../prompts/security-sweep.md) |

Note the division of labor: the daily review is *depth on recency* (few
commits, full caller-chain tracing); the two sweeps are *breadth on the whole
repo* (known checklists, mechanical recipes). Together they cover the axis the
pipeline doesn't: time.

## Running them

Each prompt file has the same anatomy: an **operator header** (what it is,
constants, cadence) around a **`## PROMPT` block** that is the actual routine
— pasted verbatim, never summarized. Three deployment modes:

- **Scheduled cloud agent** (best for the daily review): the `## PROMPT` block
  *is* the routine's instruction; the agent clones `{{REPO_SLUG}}` and runs.
  Grant it `gh` permission to list and create issues/PRs — that is its entire
  output channel.
- **Manual paste into a local session** (best for the sweeps): same block,
  same behavior, plus optional access to local agent memory for
  known-exception filtering.
- **Spawned to a read-only agent**: the recurring-patterns sweep can run as a
  QA-type sub-agent since it only greps and reads; the security sweep needs a
  session that can fan out four parallel read-only children.

Whatever the mode, the routine is **read-only until the routing step** — it
analyzes, then files issues or hands confirmed criticals to the pipeline. It
never edits code in place.

## The cloud-agent dedup principle

Run the daily review as a scheduled cloud agent and you hit an architectural
constraint worth designing around rather than against: **a scheduled cloud
agent only clones the repository.** It cannot read local memory files, session
history, or anything else on your machine. So local state cannot be the
"already reported" ledger.

The solution: **GitHub itself is the deduplication store.** The "already
known" set is defined as:

> open PRs ∪ open issues labeled `daily-review` ∪ recently-closed
> `daily-review` issues

Every run rebuilds this set via `gh issue list` / `gh pr list` before
analyzing anything, and skips any finding the set already covers. Every
finding it *does* raise is written back as a labeled issue or a PR — which
makes it part of the next run's known set automatically. No separate log file,
no state that can desynchronize, and the ledger is human-readable by
construction.

Concretely, every run starts by rebuilding the set:

```bash
gh pr list    --repo {{REPO_SLUG}} --state open --limit 100 --json number,title,body
gh issue list --repo {{REPO_SLUG}} --state open   --label daily-review --limit 200 --json number,title,body
gh issue list --repo {{REPO_SLUG}} --state closed --label daily-review \
    --search "closed:>=<30 days ago>" --limit 200 --json number,title,body
```

…and every finding it raises carries the `daily-review` label plus the
location and causing commit in its body — the fields the *next* run matches
against.

Two operational corollaries: don't remove the `daily-review` label from an
issue before closing it (the routine will re-report the finding), and closing
or merging is the only "resolution" signal the routine understands.

## Severity routing

| Finding class | Route | Rationale |
| :--- | :--- | :--- |
| Normal (high/medium/low) | **Labeled GitHub issue** (`daily-review` + `severity:*`) | Triage at human pace; batch the small stuff |
| Critical (data loss, security, money, runaway sends) | **Fix through the full pipeline** on a fresh `feature/*` branch → **PR** | Even an urgent fix gets QA + integration + docs |
| Uncertain but strong signal | Issue, marked "needs verification" | Auto-fix is reserved for the *confirmed* critical |

Two hard rules ride along. The routine **never commits to
`{{MAIN_BRANCH}}` or `{{PRODUCTION_BRANCH}}` directly** — critical fixes take
the same pipeline as any feature. And a routine-produced PR is **never merged
unsupervised**: it was opened by an autonomous scheduled process; a human
reviews and merges it, always.

## Trace, don't pattern-match

The rule that keeps standing routines from becoming noise generators:
**follow the caller chain before reporting.** A grep hit is a candidate, not a
finding. For every candidate, read the actual code and trace who calls it,
what consumes its output, and what the downstream effect is
(frontend ↔ controller ↔ job ↔ schedule ↔ listener ↔ policy). Report only
findings you can support with a concrete trace or reproduction steps —
below that confidence threshold, either verify further or route to an issue
marked "needs verification," never to an automated fix.

The mirror duty: **record disproven findings** in the relevant agent's memory
([chapter 06](06-memory-system.md)). A suspicion that cost an hour to disprove
and wasn't written down will cost the same hour again next month — routines
run on schedules, and schedules don't remember.

## Feeding discoveries back

When a routine keeps finding the same *new* kind of bug, that's a pattern
being earned ([chapter 04](04-bug-patterns.md), "growing your own pattern
library"): add it to the pattern library and the relevant agent templates so
the pipeline starts catching it per-change — then the routine's job shrinks
back to what only it can see. A standing routine whose finding rate trends
toward zero is not failing; it is succeeding.

---

<div align="center">

[← 06 — The Memory System](06-memory-system.md) &nbsp;·&nbsp; [Playbook index](README.md) &nbsp;·&nbsp; [08 — Field Notes →](08-field-notes.md)

</div>
