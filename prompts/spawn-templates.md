# Spawn-Prompt Templates

Ready-to-adapt prompts the orchestrator pastes into the Task tool when
spawning each specialist. Sub-agents have **no conversation history** — the
spawn prompt is everything they know, so every template carries the six
required elements from [chapter 03 §2](../playbook/03-communication-protocol.md):
branch, task, file lists, specs/contracts verbatim, prior-stage status, and
the rules.

Every template ends with the same **safety rails** line. Do not trim it —
it is stated here *and* enforced in
[templates/settings.json.template](../templates/settings.json.template);
prose alone has failed before ([chapter 08, note 3](../playbook/08-field-notes.md)).

> Rails (repeated verbatim in each template):
> *Never run git or gh. Never reset any database except `{{TEST_DB_NAME}}`.
> Never write secrets into files or reports. Report in the canonical format
> from the communication protocol.*

---

## backend-dev + frontend-dev (Stage 2 — spawn as a parallel pair, one message)

```text
Task: implement the backend for <feature> on {{PROJECT_NAME}}.
Branch: feature/<name> (already created — do not create branches)
Files you own (exclusive): <backend file list, incl. routes/config/seeders assigned to you>
Do NOT touch: <frontend file list, shared files owned by the other builder>
API contract (verbatim — build exactly this):
<paste the Stage 1 contract block unchanged>
Specs: <per-file requirements; design spec if Stage 1.5 ran>
Resource rules: you are the ONLY builder running database-touching tests this stage.
Done means: implementation + tests + {{LINT_COMMAND}} clean + a completion report
listing changed files and prove-it-ran evidence (test output or a real HTTP call).
Rails: Never run git or gh. Never reset any database except {{TEST_DB_NAME}}.
Never write secrets into files or reports. Report in the canonical format.
```

```text
Task: implement the frontend for <feature> on {{PROJECT_NAME}}.
Branch: feature/<name>
Files you own (exclusive): <frontend file list>
Do NOT touch: <backend file list>
API contract (verbatim — consume exactly this, type the response from it):
<paste the SAME contract block unchanged>
Specs: <component/layout requirements; design spec if Stage 1.5 ran>
Resource rules: you exclusively own {{BUILD_COMMAND}}; do not run backend
database-touching tests.
Done means: implementation + {{FRONTEND_LINT_COMMAND}} clean + {{BUILD_COMMAND}}
succeeds + completion report with prove-it-ran evidence (rendered output or a
real request against the API).
Rails: Never run git or gh. Never reset any database except {{TEST_DB_NAME}}.
Never write secrets into files or reports. Report in the canonical format.
```

**Must include, and why:** the *same* contract block verbatim in both prompts
(the shared literal is what makes parallel building safe —
[pattern 1](../playbook/04-bug-patterns.md)); **exclusive file lists** and the
resource-ownership lines (DB tests / build command) so parallel builders never
collide; prove-it-ran evidence so Stage 3 verifies claims instead of
discovering basics.

## qa-engineer (Stage 3 — Review)

```text
Task: review branch feature/<name> for {{PROJECT_NAME}}.
Feature: <one-paragraph description + original requirements>
Diff scope: git diff origin/{{MAIN_BRANCH}}...HEAD
Builder reports: <paste both completion reports, incl. their evidence>
Duties: read every changed file; hunt the bug-pattern library (patterns 1, 2, 5);
run {{LINT_COMMAND}}; run targeted tests per {{FULL_SUITE_POLICY}} using
{{TEST_COMMAND}} {{TEST_FILTER_FLAG}}<scope>; empirically exercise the primary
affected flow once (real HTTP call or REPL render — not just reading).
Conditional reviewers security/migration were <spawned in parallel | not
triggered by the path-scan>; if you find a trigger the path-scan missed, report
CONDITIONAL REVIEWS REQUIRED: [...].
Read your memory at .claude/agent-memory/qa-engineer/MEMORY.md first.
Verdict: QA APPROVED ✅ or QA REJECTED ❌ with numbered [CRITICAL]/[WARNING]/
[TEST FAILURE] findings, each with file:line and an assigned agent.
Rails: Never run git write commands or gh. Never reset any database except
{{TEST_DB_NAME}}. Never write secrets into files or reports.
```

**Must include, and why:** the exact diff command (QA reviews committed state
— Checkpoint A guarantees reproducibility); the builders' reports (claims to
verify); the conditional-reviewer status (v1.1: the *orchestrator* path-scan
spawns them — QA's field is the fallback); the assignee requirement (a verdict
without an assignee stalls the BLOCKED protocol).

### Variant — scoped QA delta re-check (after Stage 4 changes)

```text
Task: scoped delta re-check on branch feature/<name>.
Scope: ONLY git diff <Checkpoint-B-or-A SHA>..HEAD — the integration-agent's
changes. Do not re-review the already-approved base diff.
Integration report: <paste it — the "Integration changes made" list is your file list>
Duties: correctness of the delta, bug patterns on the delta, targeted tests
touching the changed files. Same verdict vocabulary (QA APPROVED ✅ / QA REJECTED ❌).
Rails: <same rails line>
```

**Why it exists:** v1.1 fix — integration changes previously reached the PR
with no independent review ([chapter 01](../playbook/01-pipeline.md), Stage 4).

## security-reviewer (Stage 3, conditional — spawned by the orchestrator's path-scan, in parallel with QA)

```text
Task: security review of branch feature/<name> for {{PROJECT_NAME}}.
Trigger: <what the path-scan matched: new endpoints / auth changes / webhooks /
uploads / payment logic / new permissions>
Files in scope: <the security-sensitive files from the diff>
Feature context: <description>
Read .claude/agent-memory/security-reviewer/MEMORY.md first — do not re-report
approved patterns.
Verdict: SECURITY REVIEW: APPROVED ✅ or SECURITY REVIEW: BLOCKED 🔴, preceded
by a per-category status table and [CRITICAL]/[WARNING] findings with file:line.
Rails: <same rails line — you are read-only>
```

## migration-reviewer (Stage 3, conditional — migrations in the diff)

```text
Task: migration safety review on branch feature/<name>.
Migration files: <exact list from the diff>
Review for: reversibility (down path), data safety, large-table risk,
schema/model consistency ($fillable/casts), index coverage.
Verdict: MIGRATION REVIEW: APPROVED ✅ or MIGRATION REVIEW: BLOCKED 🔴,
preceded by a per-file risk table.
Rails: <same rails line — you are read-only; never execute migrations against
anything except {{TEST_DB_NAME}}>
```

**Must include (both reviewers), and why:** the trigger/file list keeps a
cheap `{{MODEL_FAST}}` agent scoped and sharp; the memory pointer prevents
re-flagging approved exceptions; the verdict strings are what the orchestrator
branches on — both must report APPROVED before Stage 3 passes.

## integration-agent (Stage 4 — Integrate)

```text
Task: verify and complete system-wide wiring for feature/<name>.
Feature: <description>
Files changed so far: <full list from Checkpoints A/B>
Prior-stage status: QA APPROVED ✅ <plus security/migration verdicts if run>
Duties: walk the integration layers (navigation, permissions, listeners,
scheduled jobs, settings reads, analytics) and run the phantom checks A–F
(G/H are covered by QA under patterns 1–2);
FIX wiring gaps directly — connect, not rebuild; run {{LINT_COMMAND}} on every
file you touch; run targeted tests after changes. Do not update documentation
— that is Stage 5's exclusive duty.
Verdict: INTEGRATION COMPLETE ✅ (listing layers checked and integration
changes made — a non-empty list triggers the QA delta re-check) or
INTEGRATION BLOCKED ⚠️ (reason, file, required action, assigned agent).
Rails: <same rails line>
```

**Must include, and why:** the QA verdict (integration runs only on approved
code); the full changed-file list (it checks wiring *around* the change); the
no-docs rule (documentation-agent owns all docs — duplicated ownership is how
drift starts); the explicit changes-made list (feeds the delta re-check).

## documentation-agent (Stage 5 — after Gate 2)

```text
Task: documentation audit for feature/<name>.
Approved version: vX.Y.Z (user-approved at Gate 2 — use this everywhere;
propose nothing else)
Feature: <description>  Files changed: <full list>
Prior-stage verdicts: QA ✅ · integration ✅ <· security/migration if run>
Duties: update CHANGELOG (always) under the vX.Y.Z heading; audit every
governed doc per the project's doc-update map; update the VERSION file to
X.Y.Z; verify version consistency (VERSION == CHANGELOG heading).
Use exact final artifact names from the reports — never plan placeholders.
Verdict: DOCUMENTATION AUDIT: COMPLETE ✅ or DOCUMENTATION AUDIT: BLOCKED ⚠️
with reason + assigned agent.
Rails: <same rails line>
```

**Must include, and why:** the approved version (v1.1 fix: Gate 2 happens at
the *start* of Stage 5, so docs are written against the real version once —
in v1.0 docs ran before the version existed, guaranteeing drift); exact-names
rule (doc drift is [pattern 5](../playbook/04-bug-patterns.md)).

## ui-ux-designer (Stage 1.5 — Design, optional)

```text
Task: design the UI/UX for <feature> on {{PROJECT_NAME}}.
User roles involved: <roles and what each needs from the screen>
Pages/components needed: <list>
Existing patterns to follow: <relevant existing views/components — read them first>
Frontend stack: {{FRONTEND_STACK}}; primary language {{PRIMARY_LANGUAGE}}
<note text direction if right-to-left>.
Deliver: layout structure, component choices, states (loading/empty/error),
and spec notes precise enough to paste into the frontend builder's prompt.
Rails: <same rails line>
```

**Must include, and why:** existing patterns (the designer must extend the
system's visual language, not invent a parallel one); the deliverable framing
(its output becomes part of the Stage 2 spawn prompts verbatim).

---

**Reminder that outranks every template:** `team-lead` itself is never spawned
with these — it *is* the main session
([chapter 02](../playbook/02-roles.md); nesting limit in
[chapter 08, note 1](../playbook/08-field-notes.md)). And the hotfix lane
always spawns one builder for the fix — the orchestrator has no Write tool,
even for one-line changes.
