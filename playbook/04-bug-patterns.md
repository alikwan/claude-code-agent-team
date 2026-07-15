<img src="../assets/icons/playbook.svg" width="40" height="40" alt="">

[**← playbook index**](README.md) &nbsp;·&nbsp; chapter 04 of 08

# 04 — The Bug-Pattern Library

Five failure modes that repeat in AI-assisted (and, honestly, human) development.
They are not hypothetical: **~30 review blockers across two large production PRs
taught us these.** Both PRs had green test suites. Every one of the thirty bugs
lived in a *seam* — the boundary between two layers that each had its own
passing tests but that nobody verified against each other.

That is the unifying theme of this chapter: **tests verify layers in isolation;
these patterns live at the points of contact.** A response-shape test passes. A
component-render test with mocked data passes. The screen is still blank in
production, because nobody asserted that the component consumes the response
the controller actually emits.

## Who checks what

Pattern ownership is deliberately split so no check is run twice and none is
run zero times:

| Patterns | Owner | When |
| :--- | :--- | :--- |
| 1 (contract mismatch), 2 (enum/config strings), 5 (doc drift) | `qa-engineer` | Stage 3 — Review, on the diff |
| 3 (phantom features), 4 (job/schedule wiring) | `integration-agent` | Stage 4 — Integrate, via [phantom checks A–F](05-phantom-checks.md) (G/H overlap QA’s patterns 1–2 and are covered there) |
| All five, codebase-wide | [recurring-patterns sweep](../prompts/recurring-patterns-sweep.md) | Standing routine ([chapter 07](07-standing-routines.md)) |

Each agent's template references the other's scope instead of duplicating it.
Builders additionally carry *authoring-time prevention* rules for all five —
the cheapest place to kill a pattern is before it exists.

---

## [PATTERN-1] Frontend ↔ API contract mismatch

**Definition.** The frontend reads a key, shape, or nesting level that the API
never emits: response envelopes (`{ data: [...] }` vs a bare array), key-name
drift (`assignee_name` vs `assignee.name`), object-vs-array shape confusion,
or a column selected under an aliased/computed name the query never produced.
The symptom is a blank widget, an `undefined` label, or a silent `NaN` — never
an exception, never a failed test.

**Why tests and casual review miss it.** Backend tests assert the response
shape against the backend's own expectation. Frontend tests render against
mocked data written by the same author who wrote the consuming code — the mock
inherits the misunderstanding. Casual review sees two individually plausible
files and no diff line where they disagree.

**Detection recipe.**

1. For each new/modified frontend file, list every API call (grep for your
   HTTP client or data-access module imports).
2. For each URL, find the route registration, then the controller/handler
   method serving it.
3. Read the **literal response construction** — the actual keys emitted, at
   the actual nesting depth. Do not trust the plan or the docstring.
4. Read the keys the frontend consumes: property accesses on the response,
   destructured fields, mapped-over arrays, typed interfaces.
5. Flag every key the frontend reads that the controller does not emit, and
   every shape assumption (array vs object, wrapper vs bare) that differs.

Common subtle variants to hunt for:

- Backend returns `{ data: [...], meta: {...} }`; frontend maps over the
  wrapper object instead of `data`.
- Backend returns `{ buckets: { "0_30": n } }`; frontend reads
  `aging["0_30"]`, skipping the wrapper.
- Backend emits `assignee_name`; frontend reads `assignee.name`.
- Backend emits `exceeded_by_days`; frontend reads `days_over`.
- Backend returns an object keyed by category; frontend expects an array of
  `{ category, count }` entries.
- Backend eager-loads a relation with a column list that omits the column a
  computed accessor needs — the SQL succeeds, the attribute resolves to null,
  and the failure surfaces only in production data.

**Fix strategy.** Read the actual controller and make the contract explicit:
**prefer adding alias keys to the controller response over silent frontend
fallbacks** (`?? row.other_name` hides the disagreement; an alias documents
it). Route all access to a resource through **one data-access module per
resource** — a single typed function per endpoint (e.g. a `useWidgetStats()`
hook over TanStack Query calling `src/api/widgets.ts`) so the response shape
is asserted in exactly one place instead of scattered across components.

**Authoring-time prevention.** This is what the Stage 1 API contract exists
for: the orchestrator writes the exact request/response shape and pastes it
**verbatim into both builder prompts** ([chapter 01](01-pipeline.md)). The
frontend builder types the response interface directly from the contract; the
backend builder shapes the response from the same block. Both include one real
HTTP call in their prove-it-ran evidence.

---

## [PATTERN-2] Enum / config-key string mismatch

**Definition.** Code references a string that is supposed to exist somewhere
else — an enum case, a settings key, a status value, a column name — and it
doesn't. The membership check silently returns false, the setting read
silently returns the default, the column silently resolves to null. Nothing
throws.

**Why tests and casual review miss it.** String literals look
self-evidently correct; reviewers read `'partially_broken'` and see a
plausible status, not a value that must be verified against an enum defined
three directories away. Tests that construct their own fixtures use the same
(wrong) literal on both sides and pass.

**Detection recipe.** The rule is absolute: **every string literal that claims
to reference a source of truth must be grepped against that source of truth.**

- For each membership check against an enum-like list, open the enum/constant
  definition and verify **every** string in the list is an actual case.
- For each runtime settings read (`Setting.get("group.key")` or equivalent),
  verify the key exists in the seeder/defaults file — exact string, not a
  paraphrase.
- For each settings *group* fetch, verify keys are actually registered under
  that group name and that the group follows your project's naming convention.
- For each column referenced in a select/eager-load list, verify it exists in
  the schema (this overlaps with [check G](05-phantom-checks.md)).

**Fix strategy.** Correct the literal at the point of use, or add the missing
case/key at the source of truth — whichever the plan intended. When ambiguous,
that ambiguity is itself a `[CRITICAL]` finding: the two sides were built from
different understandings.

**Authoring-time prevention.** Builders grep every new key they introduce
(`grep -rn "your_new_key"`) to confirm it is *read* somewhere before reporting
complete. Where the language allows, replace string unions with real enum
types so the compiler does this check for free.

---

## [PATTERN-3] Phantom features (defined but never wired)

**Definition.** The feature exists as a class, column, setting, or permission
— and nothing dispatches, reads, enforces, or writes it. A notification class
with zero dispatch sites. A feature flag no runtime code checks. A permission
seeded but never attached to a route or ability check. Documentation says the
feature works; nothing does.

**Why tests and casual review miss it.** Unit tests exercise the artifact
directly — `new LateArrivalAlert(...)` renders fine in a test — so coverage
looks healthy. Review sees a well-written class and approves it. Absence of a
call site is invisible in a diff, because the missing line was never written.

**Detection recipe.** This is the core of the integration stage. The full
grep-template set — one per artifact type — is
[chapter 05, checks A–D](05-phantom-checks.md). The shape is always the same:
take the artifact's name, grep the codebase for consumption sites, and treat
**zero hits as a finding**, not as silence.

**Fix strategy.** Wire it or remove it. If the wiring is intentionally
deferred to a future change, that intent must be visible: a
`// TODO(phase-N):` comment at the definition **and** a note in the CHANGELOG,
so QA and the sweep know it is deliberate rather than forgotten.

**Authoring-time prevention.** Builders treat "defined" and "wired" as one
task: the same change that adds a notification class adds its dispatch call;
the same change that seeds a flag adds the runtime guard that reads it. Their
completion report names the wiring site for every new artifact.

---

## [PATTERN-4] Background-job / schedule wiring gaps

**Definition.** Asynchronous plumbing that fails silently: a job dispatched to
a queue the worker system is not configured to consume (dispatched, queued,
never executed); two scheduled tasks firing at the same time producing
duplicate side effects (users get the reminder twice); a scheduled task
without an overlap guard stacking concurrent runs; or workers still executing
**stale code after deploy** because long-lived worker processes never reload.

**Why tests and casual review miss it.** Tests run jobs synchronously or with
a fake queue — the worker configuration is never in the loop. Schedule
collisions require reading the *whole* schedule file, not the diff. And stale
workers are invisible in any code artifact at all: the code is correct; the
process running it is old.

**Detection recipe.**

- For each job dispatched to a named queue, grep the worker configuration for
  that queue name. Absent = dispatched but never consumed.
- For each new scheduled task, grep the schedule file for **other** tasks at
  the same time producing the same notification/effect — especially when the
  new task replaces a legacy one that nobody disabled.
- Verify overlap/single-server guards on any schedule that mutates shared
  state.
- For idempotency-flag jobs (`reminder_sent`, `processed_at`), verify the flag
  is unique to this job — a shared flag means the legacy and new jobs race to
  claim it, or both fire.
- After any deploy touching job code: confirm `{{QUEUE_RESTART_COMMAND}}` ran.
  This is an operational check, not a code check — see
  [chapter 08, note 4](08-field-notes.md).

**Fix strategy.** Register the queue; disable the superseded schedule in the
same change that introduces its replacement; add the missing guard; make
worker restart part of the deploy procedure rather than tribal knowledge.

**Authoring-time prevention.** The builder that introduces a job registers its
queue in the same change and states in its completion report which schedule
entries it checked for collisions.

---

## [PATTERN-5] Documentation / code drift

**Definition.** The constitution, CHANGELOG, README, or reference docs mention
class names, setting keys, channel names, commands, or counts that don't match
the code. The classic cause: the plan used placeholder names, implementation
diverged, and the docs were written from the plan.

**Why tests and casual review miss it.** No test executes documentation.
Review naturally focuses on code files; a doc paragraph asserting
`ReminderDigestJob` exists reads as background noise, not as a claim requiring
verification. Drift also tends to appear in only *one* of several doc files,
so spot-checking the wrong one gives false confidence.

**Detection recipe.**

- For each class/job/event name mentioned in governed docs, verify a file with
  that exact name exists at the documented path.
- For each documented setting key, grep the seeder for the exact string.
- For each documented channel/route/command, verify against the registration
  file.
- Check **all** governed doc files in the same pass (CHANGELOG, constitution,
  README, reference docs) — drift hides in whichever one you skipped.

**Fix strategy.** Fix docs to match code (usual) or flag the code as having
diverged from the approved plan (rarer, and a bigger conversation). Never
"fix" by deleting the doc claim without checking which side is right.

**Authoring-time prevention.** Two mechanisms in the pipeline exist for this
pattern alone: builders name new artifacts by their **exact final names** in
completion reports (never plan placeholders), and `documentation-agent` runs
as a blocking stage — Stage 5 — with the approved version passed in, so docs
are audited against the code as merged, not as planned.

---

## Labeling findings: the `[PATTERN-N]` convention

Every finding matching a library pattern is labeled so patterns stay visible
in reports and countable over time — the label is what turns anecdotes into a
library:

```text
[PATTERN-1: frontend↔API mismatch] AssigneeWorkloadCard.tsx:114 reads
  `assignee.name` but ReportController::workload() emits `assignee_name`.
  → Add alias key `name` to the controller response.

[PATTERN-3: phantom feature] LateArrivalAlert exists at
  app/Notifications/LateArrivalAlert.php but grep -rn "LateArrivalAlert"
  across the backend shows ZERO dispatch sites. → Wire or remove.
```

Inside a `QA REJECTED ❌` report these carry the normal severity prefix as
well (`[CRITICAL]` / `[WARNING]`); the pattern tag is additive.

## Growing your own pattern library

These five are *ours* — earned, not brainstormed. Yours will differ. The rule
that keeps a library honest:

1. **Patterns are earned from real review blockers.** A candidate qualifies
   when the same root cause has produced findings in at least two separate
   changes. One-offs go to agent memory, not the library.
2. **Record it through the pattern-proposal flow** (this repo's
   [pattern-proposal issue template](../.github/ISSUE_TEMPLATE/pattern-proposal.yml)
   shows the required fields: definition, why tests miss it, detection recipe,
   two real occurrences).
3. **Install it in three places:** the QA agent's memory
   ([chapter 06](06-memory-system.md)), this chapter (as `[PATTERN-6]`, with
   its detection recipe), and the
   [recurring-patterns sweep prompt](../prompts/recurring-patterns-sweep.md)
   so existing instances get hunted codebase-wide, not just in future diffs.

A pattern library that only grows in someone's head protects exactly one
reviewer. Written down with a detection recipe, it protects every session that
ever runs.

---

<div align="center">

[← 03 — Communication Protocol](03-communication-protocol.md) &nbsp;·&nbsp; [Playbook index](README.md) &nbsp;·&nbsp; [05 — Phantom Checks →](05-phantom-checks.md)

</div>
