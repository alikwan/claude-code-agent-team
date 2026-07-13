# 03 — Communication Protocol

The rules that let ten independent contexts behave like one team. This chapter
is the **single source of truth** for the verbatim report formats — the agent
templates repeat them exactly; if you change a string here, change it
everywhere (CI in this repo enforces the match).

## 1. The Git Ownership Rule

> **Only the orchestrator runs mutating `git` (add/commit/push/branch/stash/tag)
> and `gh`. Every other agent writes files only — read-only inspection
> (`git diff` / `git log` / `git show`) is permitted for reviewers.**

All commits happen at the four checkpoints (A–D). No other agent stages,
commits, stashes, pushes, tags, or opens PRs — ever. This single rule buys:
deterministic history (commit ↔ stage), zero inter-agent conflicts, and one
place to audit what changed when.

Corollaries:

- Builders finishing early don't commit "their part" — they report and exit.
- A stash is never a parking spot. If the orchestrator stashes for a context
  switch, it pops and commits before the pipeline continues.
- Force-push is forbidden unless the user explicitly requests it.

## 2. Spawning with full context

Sub-agents have **no conversation history**. They know only what their spawn
prompt contains. Every spawn prompt therefore includes, always:

1. **Branch name** — which branch the work happens on.
2. **The task** — what to build/review/fix, specifically.
3. **File lists** — which files to create/modify (builders: exclusive
   ownership; see resource rules in [chapter 01](01-pipeline.md), Stage 2).
4. **Specs and contracts** — the API contract verbatim, design specs if any.
5. **Prior-stage status** — what QA rejected, what integration flagged.
6. **The rules** — never run mutating git commands or gh; never reset any database except
   `{{TEST_DB_NAME}}`; report in the canonical format.

Ready-made spawn prompts for every agent type:
[prompts/spawn-templates.md](../prompts/spawn-templates.md). The most common
orchestration bug is an under-specified spawn — when a sub-agent goes off the
rails, fix the prompt template, not the agent.

## 3. Verdict formats (canonical)

These blocks are contracts. The orchestrator branches on them literally.

**QA — approval:**

```
QA APPROVED ✅
- Tests: <N> passed, 0 failed (targeted: <scope>)
- Lint: clean
- Empirical check: <flow exercised + result>
- Conditional reviews: <none required | security ✅ | migration ✅>
- Requirements: met
Ready for the next stage.
```

**QA — rejection:**

```
QA REJECTED ❌
Issues found:
1. [CRITICAL] <file>:<line> — <problem> → <required fix> (assigned: BACKEND|FRONTEND)
2. [WARNING]  <file>:<line> — <problem> → <suggested fix>
3. [TEST FAILURE] <test name> — <output excerpt>
CONDITIONAL REVIEWS REQUIRED: [<security|migration>]   ← only if the path-scan missed a trigger
```

**Integration:**

```
INTEGRATION COMPLETE ✅
- Layers checked: <list with ✅ / N/A per layer>
- Integration changes made: <none | file list>   ← non-empty triggers the QA delta re-check
- Lint on touched files: clean
- Targeted tests after changes: <N> passed
```

```
INTEGRATION BLOCKED ⚠️
- Reason: <what is missing or broken>
- File: <path>
- Required action: <specific fix>
- Assigned to: BACKEND | FRONTEND
```

**Security reviewer:** `SECURITY REVIEW: APPROVED ✅` or
`SECURITY REVIEW: BLOCKED 🔴`, preceded by a per-category status table and
`[CRITICAL]`/`[WARNING]` findings with `<file>:<line>`.

**Migration reviewer:** `MIGRATION REVIEW: APPROVED ✅` or
`MIGRATION REVIEW: BLOCKED 🔴`, preceded by a per-file risk table.

**Documentation:**

```
DOCUMENTATION AUDIT: COMPLETE ✅        (or: BLOCKED ⚠️ + reason + assigned agent)
| Doc file | Status | Action taken |
| ...      | Updated / Already current / N/A | ... |
Version consistency: VERSION file == CHANGELOG heading == vX.Y.Z ✅
```

**Builders (completion report):** what changed (file list), how it satisfies
the contract, **prove-it-ran evidence** (test output / HTTP response / REPL
result), and anything the next stage should know. Honest reporting is a hard
rule for every agent: failing tests are reported as failing, skipped steps as
skipped — an agent that embellishes its report poisons every stage after it.

## 4. The BLOCKED protocol

When any verifier reports BLOCKED (or QA reports REJECTED):

1. The blocking agent **names what is missing and which agent is responsible**
   (BACKEND / FRONTEND / other) — a verdict without an assignee stalls the
   pipeline.
2. The orchestrator spawns the responsible agent with the findings verbatim.
3. After the fix, the **blocking agent re-runs** before the pipeline resumes —
   the fixer never self-certifies.

## 5. Escalation — the 2-cycle rule

> If the same agent returns a negative verdict (**REJECTED or BLOCKED**) on
> the **same root cause** for **2 consecutive cycles**, the orchestrator stops
> the pipeline and reports to the user. Never loop indefinitely.

Threshold 2 is deliberate: 1 forbids legitimate iteration; 3 burns a full
cycle discovering what 2 already proved. *(v1.1 wording fix: the original rule
said only "BLOCKED", which — read literally — permitted an infinite
QA-REJECTED loop.)*

## 6. The two human gates

| Gate | When | What you approve |
| :--- | :--- | :--- |
| **Gate 1 — Plan** | End of Stage 1 | Files to touch, approach, API contract, lane choice |
| **Gate 2 — Version** | Start of Stage 5 | The semver bump (skippable via `{{AUTO_VERSION}}: true`) |

Between the gates the pipeline runs autonomously; every stage still leaves a
committed checkpoint you can inspect or roll back to. Two gates is the
deliberate balance — one per irreversibility boundary (committing to an
approach; declaring a release).

## 7. The pipeline state file (resume protocol)

Long pipelines exhaust context; sessions get interrupted. The orchestrator
maintains one small state file per task
([template](../templates/pipeline-state.md.template)) and updates it at every
stage transition:

```markdown
task: <name>            branch: feature/<name>
stage: 4-integrate      updated: <date>
checkpoints: A=<sha> B=<sha>
verdicts: QA ✅ (2 cycles) · security ✅ · migration N/A
next: integration-agent spawned, awaiting report
```

A fresh session reads this file plus `git log` on the branch and resumes from
the exact stage — no archaeology, no re-running approved stages.

## 8. Safety rails (non-negotiable, enforced twice)

Stated in every spawn prompt **and** enforced in the permission system
([templates/settings.json.template](../templates/settings.json.template)) —
prose alone has already failed us once
([chapter 08](08-field-notes.md), note 3):

- No agent ever resets, migrates-fresh, or wipes any database except
  `{{TEST_DB_NAME}}`.
- No agent deletes branches or force-pushes.
- No agent writes secrets into memory files, docs, or commit messages.
