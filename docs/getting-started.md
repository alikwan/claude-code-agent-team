# Getting Started

From zero to your first pipeline run. Prerequisites: [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
installed, the [`gh` CLI](https://cli.github.com/) authenticated, and a git
repository you actually work on.

## The 5-minute start

**Block 1 — copy the templates into your project:**

```bash
cd your-project
npx degit alikwan/claude-code-agent-team/agents .claude/agents --force
rm .claude/agents/README.md      # docs, not an agent definition
rm -r .claude/agents/optional    # keep only if you want the optional test-engineer
npx degit alikwan/claude-code-agent-team/templates .claude/pipeline-templates --force
mkdir -p .claude/agent-memory
```

**Block 2 — generate your constitution.** Open Claude Code at the repo root
and paste the entire contents of
`.claude/pipeline-templates/generate-your-claude-md.md`. The agent will explore
your repo read-only, ask you at most 6 questions (deploy environment, branch
strategy, test database name, user-facing language…), write `CLAUDE.md`, and
self-check that every command it documented actually runs.

**Block 3 — resolve the placeholders and launch:**

```bash
# fill the table in docs/placeholders.md with your values, find-replace, then verify —
# must return nothing (${{ ... }} in the CI templates is GitHub Actions syntax, not a placeholder):
grep -rnE '\{\{[A-Z][A-Z0-9_]*\}\}' .claude/ CLAUDE.md

claude --agent team-lead   # or pick team-lead from the agent selector
```

> ⚠️ **The one thing you must not do:** never invoke `team-lead` through the
> Task/Agent tool from another session. A sub-agent cannot spawn sub-agents —
> the pipeline will stall at Stage 2. The orchestrator **is** your main
> session. (The full failure story:
> [playbook/08-field-notes.md](../playbook/08-field-notes.md), notes 1–2.)

## Set up the memory scaffolds

One directory per agent, seeded from the template:

```bash
for a in team-lead backend-dev frontend-dev qa-engineer integration-agent \
         documentation-agent security-reviewer migration-reviewer ui-ux-designer; do
  mkdir -p .claude/agent-memory/$a
  cp .claude/pipeline-templates/MEMORY.md.template .claude/agent-memory/$a/MEMORY.md
done
# kept the optional test-engineer? add it to the list above before running.
```

Commit the scaffolds. Memory is version-controlled and travels with the code —
how it works: [playbook/06-memory-system.md](../playbook/06-memory-system.md).

## Optional: permissions and CI

- Merge `.claude/pipeline-templates/settings.json.template` into your
  `.claude/settings.json` — it denies destructive database and git commands at
  the permission layer, which is where guardrails belong
  ([field note 3](../playbook/08-field-notes.md) explains why prose isn't enough).
- Copy the CI templates (they were installed by Block 1):
  `cp .claude/pipeline-templates/ci/tests.yml.template .github/workflows/tests.yml`
  — this is the **full-suite backstop** that makes targeted-only testing safe
  (see the invariant in [placeholders.md](placeholders.md)). Optionally also
  `agent-mention.yml.template` → `.github/workflows/claude-mention.yml` and
  `agent-pr-review.yml.template` → `.github/workflows/claude-pr-review.yml`
  for @-mention tasks and automatic PR review.

## Your first task — what you should see

Give the orchestrator something real but modest ("add an export button to the
invoices table"). A healthy run looks like this:

1. **A plan, then a stop.** Files to touch, approach, an API contract block if
   the change crosses backend/frontend, and an explicit request for your
   approval. *If it starts editing files immediately, your CLAUDE.md is
   missing the pipeline section — regenerate it.*
2. **Parallel building.** Two sub-agent spawns in one message (backend +
   frontend), then a commit: `feat(invoices): add export button`.
3. **A QA verdict in the exact canonical format:**

   ```
   QA APPROVED ✅
   - Tests: 4 passed, 0 failed (targeted: InvoiceExportTest)
   - Lint: clean
   - Empirical check: GET /api/invoices/export → 200, CSV headers verified
   - Conditional reviews: none required
   - Requirements: met
   ```

   A `QA REJECTED ❌` with numbered findings is also healthy — you'll see a
   fix round and a `fix(invoices): address QA feedback` commit, then QA re-runs.
4. **Integration, the version gate, then documentation.** A wiring pass
   (`INTEGRATION COMPLETE ✅`), then the orchestrator proposes a version bump
   and waits for you — Gate 2. After you approve, the documentation audit runs
   (`DOCUMENTATION AUDIT: COMPLETE ✅`) and the docs + VERSION commit lands.
5. **A PR.** Branch pushed, PR opened against your integration branch with the
   stage-by-stage verdict table in the body. The orchestrator reports the URL.

If any stage's report doesn't match the formats in
[playbook/03](../playbook/03-communication-protocol.md), the corresponding
agent file was probably edited — the verdict strings are load-bearing.

## Troubleshooting

| Symptom | Cause → fix |
| :--- | :--- |
| `Agent type 'team-lead' not found` | Templates not in `.claude/agents/`, or you're not at the repo root. |
| Orchestrator "completes" without committing/pushing | It was run backgrounded or as a sub-agent. Check `git status` yourself, finish the release steps in the main session — [field note 2](../playbook/08-field-notes.md). |
| Pipeline stalls after the plan | Sub-agent spawning unavailable (nested session, or Task tool disabled by permissions). Run `team-lead` as the main session. |
| Builders edit the same file and clobber each other | The spawn prompts are missing exclusive file lists — see resource ownership in [playbook/01](../playbook/01-pipeline.md), Stage 2. |
| QA "passes" the full test suite for 40 minutes | Set `{{FULL_SUITE_POLICY}}: targeted-only` — but only if CI runs the full suite ([the invariant](placeholders.md)). |
| Agents re-litigate known-broken tests every run | Seed [KNOWN_FAILURES.md](../templates/KNOWN_FAILURES.md.template) and reference it from QA's spawn prompt. |
| A literal placeholder token shows up in agent output | You missed one during find-replace — re-run the placeholder grep from Block 3. |

## Where to go next

- Understand what you just ran: [playbook/01 — The Pipeline](../playbook/01-pipeline.md)
- Trim the team for a small project (3 agents suffice):
  [playbook/02 — Roles](../playbook/02-roles.md)
- Schedule the standing routines: [playbook/07](../playbook/07-standing-routines.md)
  \+ [prompts/](../prompts/)
- See every generic file made concrete:
  [examples/laravel-react/](../examples/laravel-react/)
