<img src="../assets/icons/playbook.svg" width="40" height="40" alt="">

[**← playbook index**](README.md) &nbsp;·&nbsp; chapter 08 of 08

# 08 — Field Notes

Everything in this chapter cost us something. These are the incidents behind
the rules — written as Symptom → Diagnosis → Rule, because a rule whose origin
story is known gets followed, and a rule that reads as arbitrary gets
"optimized away" by the next clever session. We hit every one of these in
production use; several of them more than once before we learned.

## 1. The sub-agent nesting limit

**Symptom.** We spawned the orchestrator itself via the Task tool — a session
delegating to `team-lead` the way `team-lead` delegates to builders. The
pipeline planned correctly, then stalled at Stage 2, silently, every time.

**Diagnosis.** A sub-agent cannot spawn further sub-agents. Spawned as a
sub-agent, the orchestrator's Task calls simply don't work — and it has no way
to do the builders' work itself (no Write/Edit by design), so it stops.

**Rule.** **The orchestrator is always the main session.** Start it with
`claude --agent team-lead` or select it as the main-session agent — never
invoke it through the Task tool. The agent template carries this warning in
its own description because the failure mode gives no error message.

## 2. The backgrounded orchestrator stalls before release

**Symptom.** Running the orchestrator as a backgrounded task, we watched it
complete edits, QA, integration, even documentation — then stop. No commit, no
push, no PR. Not once: **five-plus times**, same shape.

**Diagnosis.** Two mechanisms, often stacked. First, a backgrounded session's
turn can end right after dispatching a child agent — nothing re-prompts it to
continue, so the pipeline's tail never runs. Second, Gate 2 (version approval)
requires *user* approval, and the orchestrator — correctly — refuses to treat
an approval relayed through another agent's prompt as genuine user consent. It
waits for a human who isn't there.

**Rule.** Get real user approval once, up front or at the gate — then **verify
git state directly and perform the release steps in the main session**
(`git log`, `git push`, `gh pr create`). Never trust a backgrounded agent's
self-report that it pushed; check the remote. The release step is
human-operated by design, not by accident ([chapter 02](02-roles.md)).

## 3. The development database wipe

**Symptom.** Mid-review, a verification sub-agent decided the cleanest way to
confirm a migration was a fresh migrate — and ran a destructive reset against
the **development** database. Wiped it. Weeks of accumulated local test data,
gone in one tool call.

**Diagnosis.** The agent's instructions said, in prose, to be careful with
databases. Prose is a suggestion to a system that pattern-matches "reset the
DB" as a normal debugging step. Nothing *mechanical* stood between the
instruction and the command.

**Rule.** **The only database any agent may reset is `{{TEST_DB_NAME}}`** —
and this rule lives in two places: stated in every spawn prompt, *and* denied
in the permission settings
([templates/settings.json.template](../templates/settings.json.template)).
Prose-only guardrails fail; this one failed us exactly once. Enforce in the
permission layer anything you cannot afford to have "interpreted."

## 4. Background workers run stale code

**Symptom.** We shipped a fix to a queued job. Deploy succeeded, code on disk
was correct, and the bug kept happening — for hours. We nearly "fixed" it a
second time.

**Diagnosis.** Long-lived worker processes load code once, at startup. A
deploy replaces files on disk; the workers keep executing the version they
loaded yesterday. The fix was live and nothing was running it.

**Rule.** **`{{QUEUE_RESTART_COMMAND}}` is part of every deploy that touches
job, queue, or listener code.** Put it in the deploy script, not in a
checklist someone remembers. When a queued-code fix "didn't take effect,"
check worker uptime before re-debugging the code.

## 5. Long single-agent runs die and lose everything

**Symptom.** A multi-hour, single-session run — build, review, docs, the
works — hit a transient network blip at hour three. Session gone, and with it
every uncommitted edit. We got to do the whole thing again.

**Diagnosis.** Work-in-flight has no crash resilience. The failure wasn't the
blip — blips are weather — it was holding hours of state in one fragile
context with no durable snapshots.

**Rule.** **Commit in small chunks — the four checkpoints (A–D) exist for
crash resilience as much as legibility.** A session that dies mid-pipeline
loses at most one stage. On transient failure, re-dispatch the stage; the
committed foundation survives. This is also why the
[pipeline state file](../templates/pipeline-state.md.template) exists: a fresh
session resumes from the last checkpoint instead of from archaeology.

## 6. Empirical verification beats reading

**Symptom.** Twice in opposite directions. Multiple "looks correct" reviews
passed code that failed on its first real execution — a render crash a single
REPL call would have caught. And once we spent hours chasing a "bug" through
code that was, on inspection, flawless.

**Diagnosis.** Reading is hypothesis; running is evidence. The phantom bug was
the sharper lesson: the payload was fine, the code was fine — the anomaly was
a propagation delay in an external service, visible *immediately* once we
instrumented the actual payload instead of theorizing about it.

**Rule.** **Exercise the flow once before declaring done** — a REPL render, a
real HTTP call, real data through the changed path. This is now a formal QA
duty (the empirical check in the `QA APPROVED ✅` block), and builders must
include prove-it-ran evidence in completion reports. Ten minutes of execution
regularly beats three hours of confident reading.

## 7. Targeted tests are only safe with a CI backstop

**Symptom.** For speed, our pipeline ran targeted tests only — QA selected
test files related to the diff, never the full suite. It worked for hundreds
of releases. Then we went to document the practice and realized *why* it had
worked: CI ran the full suite on every PR. That fact was documented nowhere.
Nothing in the playbook would have stopped an adopter (or a future us) from
copying the targeted-only rule without the backstop — inheriting the speed and
none of the safety.

**Diagnosis.** An invariant held by an unwritten dependency between two
systems. The in-pipeline rule was visible; the thing that made it sound
wasn't.

**Rule.** **The full suite must run somewhere before merge — pick where, and
write it down.** That is the `{{FULL_SUITE_POLICY}}` placeholder
([docs/placeholders.md](../docs/placeholders.md)): `targeted-only` is
permitted *only if* CI runs the complete suite on every PR; with no CI
backstop, set `full-suite-allowed` so QA runs everything in-pipeline.

## 8. Memory grows unbounded unless enforced

**Symptom.** One agent's memory index — self-capped at 200 lines in its own
header — reached nearly 1,500 lines: **7× its self-declared cap.** Every spawn
of that agent paid thousands of tokens to load months of stale, per-branch
review notes before doing any work.

**Diagnosis.** Append-only plus prose-only cap equals unbounded growth. Each
individual append was reasonable; nobody's job was to say no; the cap in the
header was a comment, not a control.

**Rule.** **The index cap is enforced — exceeding it is a maintenance
blocker.** Overflow goes to topic files; a scheduled cheap-model consolidation
chore merges duplicates and prunes stale entries
([chapter 06](06-memory-system.md)). Any self-imposed limit that nothing
checks is, in practice, not a limit.

---

## The meta-note

Reading these back, one theme repeats: **every rule that survived is a rule
that got enforced somewhere mechanical** — a permission file, a commit
checkpoint, a CI job, a canonical string an orchestrator branches on. The
rules that lived only in prose are the ones with incident stories attached.
When you adapt this playbook, port the enforcement, not just the sentences.

---

<div align="center">

[← 07 — Standing Routines](07-standing-routines.md) &nbsp;·&nbsp; [Playbook index](README.md) &nbsp;·&nbsp; [Getting started →](../docs/getting-started.md)

</div>
