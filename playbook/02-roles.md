# 02 — Roles: The Nine Agents

Each agent is a Markdown file with YAML frontmatter, installed in your
project's `.claude/agents/`. Each has exactly one job, the minimum tools to do
it, and a fixed report format. The templates live in [`agents/`](../agents);
this chapter explains the design.

## The team table

| Agent | Stage | Tools | Model tier | Verdict vocabulary | Spawned by |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `team-lead` | 1, 6 (drives all) | Read, Bash, Grep, Glob, Task — **no Write/Edit** | strong | — | **you** (main session) |
| `ui-ux-designer` | 1.5 (optional) | Read, Write, Edit, Grep, Glob, Bash | strong | design spec | orchestrator |
| `backend-dev` | 2 | Read, Write, Edit, Bash, Grep, Glob | strong | completion report + evidence | orchestrator |
| `frontend-dev` | 2 | Read, Write, Edit, Bash, Grep, Glob | strong | completion report + evidence | orchestrator |
| `qa-engineer` | 3 (+4 delta re-check) | Read, Bash, Grep, Glob — **read-only + shell** | strong | `QA APPROVED ✅` / `QA REJECTED ❌` | orchestrator |
| `security-reviewer` | 3 (conditional) | Read, Grep, Glob, Bash — read-only | fast | `SECURITY REVIEW: APPROVED ✅` / `BLOCKED 🔴` | orchestrator (path-scan or QA flag) |
| `migration-reviewer` | 3 (conditional) | Read, Grep, Glob, Bash — read-only | fast | `MIGRATION REVIEW: APPROVED ✅` / `BLOCKED 🔴` | orchestrator (migrations in diff) |
| `integration-agent` | 4 | Read, Write, Edit, Bash, Grep, Glob | strong | `INTEGRATION COMPLETE ✅` / `BLOCKED ⚠️` | orchestrator |
| `documentation-agent` | 5 | Read, Write, Edit, Bash, Grep, Glob | fast | `DOCUMENTATION AUDIT: COMPLETE ✅` / `BLOCKED ⚠️` | orchestrator |

Plus one optional extension: [`test-engineer`](../agents/optional/test-engineer.md)
(see below).

## Why the tool restrictions are the design

Tool allowlists aren't security theater — they are what makes each role's
output trustworthy:

- **`team-lead` cannot Write/Edit.** An orchestrator that can "just quickly
  fix" things stops delegating, and the pipeline collapses back into one
  overloaded session. Its constraint *is* its discipline: it can look (Read),
  verify (Bash: git/gh/grep), and direct (Task) — nothing else.
- **`qa-engineer` and both reviewers are read-only.** A judge that can modify
  the code it judges will eventually approve its own unreviewed fixes. They
  keep Bash for *running* things (tests, lint, an HTTP call) — observing is
  not modifying.
- **`integration-agent` can write but cannot judge.** The mirror image of QA.
  It connects features to the system, and its non-trivial changes go back to
  a scoped QA delta re-check ([chapter 01](01-pipeline.md), Stage 4).
- **Builders have full file tools but never git.** Committing is the
  orchestrator's job — see the Git Ownership Rule in
  [chapter 03](03-communication-protocol.md).

## Model tiering: pay for judgment, save on procedure

Two tiers, set via `{{MODEL_STRONG}}` / `{{MODEL_FAST}}` (never pin concrete
model IDs in the templates — models are deprecated faster than playbooks rot):

- **Strong** — the orchestrator, builders, QA, integration, UI/UX. These roles
  make open-ended judgment calls on arbitrary code.
- **Fast/cheap** — documentation, security-reviewer, migration-reviewer. These
  roles execute well-defined checklists over a scoped diff. A cheaper model
  with a tight checklist and a fresh, single-purpose context outperforms an
  expensive model with a diluted one — which is also why security/migration
  are separate agents instead of extra QA duties.

## The canonical anatomy of an agent file

Every template follows the same section order. Keep it when you author your
own agents — predictable structure is what makes 10 agents maintainable:

```markdown
---
name: agent-name
description: One paragraph. What it does, when to use it, who spawns it.
tools: Read, Bash, Grep, Glob        # minimum viable set
model: {{MODEL_STRONG}}              # tier placeholder or omit to inherit
---

# Role            — who this agent is, in two sentences
# Scope           — directories/concerns it owns (and explicitly does NOT own)
# Workflow        — numbered steps, in execution order
# Critical rules  — the numbered non-negotiables
# Report format   — the verbatim verdict blocks it must emit
# Persistent memory — where its MEMORY.md lives, read-at-start/append-at-end
```

## Role-by-role rationale

**`team-lead`** — plans, defines the API contract, spawns everyone, owns git,
opens the PR. Exists because *coordination is a full-time job*: interleaving
planning with implementation in one context measurably degrades both.
⚠️ Run it as your **main session** — never spawn it via the Task tool
(sub-agents cannot spawn sub-agents; see [chapter 08](08-field-notes.md), note 1).

**`backend-dev` / `frontend-dev`** — the builders. Split not by technology
fashion but because they can then run **in parallel** against the shared API
contract, halving wall-clock time on full-stack features. Each carries
authoring-time rules that pre-empt the [bug patterns](04-bug-patterns.md)
(match key names to the actual controller response, wire every feature
end-to-end, sync env examples).

**`qa-engineer`** — the last line of defense, and deliberately the most
paranoid file in the repo. Owns the diff-correctness patterns, targeted tests,
lint, and one empirical exercise of the changed flow. Its two-verdict
vocabulary is binary on purpose: there is no "approved with reservations."

**`security-reviewer` / `migration-reviewer`** — conditional specialists.
Migrations and security-sensitive surfaces are the two change classes where a
miss costs more than every other class combined (data loss; breach). They are
cheap, scoped, and spawned only when triggered — cost scales with risk.
Their operating principle: *when in doubt, flag it — a false positive costs a
conversation; a false negative costs production data or a breach.*

**`integration-agent`** — exists because of a failure mode unique to
AI-generated code: features that are complete, correct, tested — and
unreachable. No navigation entry, permission never seeded, event dispatched
but never listened to. Human developers wire as they go; sub-agents building
from a prompt don't know the system has a sidebar. The
[phantom checks](05-phantom-checks.md) are this agent's checklist.

**`documentation-agent`** — docs drift is [bug pattern #5](04-bug-patterns.md),
and the only known fix is making documentation a *blocking pipeline stage*
rather than a promise. CHANGELOG is always required; the rest follows the
project's doc-update map.

**`ui-ux-designer`** — optional by design. When present, design intent is
specified before implementation rather than reverse-engineered after.

## Scaling the team down (and up)

**Minimum viable subset — 3 agents:** orchestrator (you + the team-lead
playbook), one builder, `qa-engineer`. This preserves the two properties that
matter most — plan-before-code and independent review — and fits weekend-sized
projects. Add integration once your system has enough wiring surface to have
phantom features; add the conditional reviewers when you have a database and
an attack surface; add documentation when you have users.

**Optional 10th — `test-engineer`:** in the base team, builders write tests
for *new* code and QA observes results — **nobody owns accumulated test
debt**. Our production memory carries months-old failing tests marked
"pre-existing, not a regression" precisely because no role was responsible for
fixing them. If that's your situation, add the
[test-engineer template](../agents/optional/test-engineer.md) and give it the
[KNOWN_FAILURES ledger](../templates/KNOWN_FAILURES.md.template).

**Deliberately absent** (we rejected these; reasoning in the
[FAQ](../docs/faq.md)): a release/deploy agent — field data shows the release
step is exactly where automation stalls, so the human-operated release is the
documented design, not a gap; a dependency-update agent (it's the hotfix lane
plus your CI); a second PR-review agent (that's CI's job).
