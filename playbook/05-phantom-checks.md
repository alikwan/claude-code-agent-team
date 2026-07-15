<img src="../assets/icons/playbook.svg" width="40" height="40" alt="">

[**← playbook index**](README.md) &nbsp;·&nbsp; chapter 05 of 08

# 05 — The Phantom Checks (A–H)

Eight mechanical checks that catch features which are *defined but not
connected* — the failure mode unique enough to AI-generated code that it earned
its own pipeline stage. `integration-agent` runs these at pipeline time
(Stage 4 — Integrate) on every change; the
[recurring-patterns sweep](../prompts/recurring-patterns-sweep.md) re-runs the
same checks **codebase-wide** on a schedule, because instances that merged
before a check existed don't fix themselves.

Every check has the same shape: **name the artifact → grep for its consumers
→ treat zero hits as a finding.** The grep templates below use generic
backend/frontend layout (`{{BACKEND_DIR}}`, `{{FRONTEND_DIR}}`) and
placeholder names like `WidgetAlert` / `widgets.export` — substitute your real
artifact names, and adjust idioms to your stack *(see
[examples/laravel-react](../examples/laravel-react) for a concrete version)*.

> **Zero hits ≠ proof of guilt.** Dynamic dispatch, container bindings, string
> concatenation, and reflection all defeat grep. A zero-hit result means
> "verify by reading," not "delete on sight." But in practice, most zero-hit
> artifacts really are phantoms — read first, then act.

---

## Check A — Notification/event/job defined but never dispatched

**Catches:** a class that exists and is never instantiated, dispatched, or
broadcast. The purest phantom: tests construct it directly, so coverage looks
fine.

```bash
# For each new Notification / Event / Job class in the diff:
grep -rn "new WidgetAlert\|WidgetAlert::dispatch\|notify(.*WidgetAlert\|broadcast(new WidgetAlert" {{BACKEND_DIR}}/
```

**Zero hits means:** nothing in the system ever triggers it. Documentation may
claim the alert exists; no user will ever receive it.

**Remedy:** add the dispatch call at the triggering code path, or remove the
class. If wiring is deliberately deferred, require the `// TODO(phase-N):`
marker plus a CHANGELOG note ([chapter 04, pattern 3](04-bug-patterns.md)).

## Check B — Event broadcast but no frontend listener

**Catches:** the backend broadcasts a real-time event and no frontend code
subscribes to it — the "live" dashboard silently never updates. Run the
inverse too: a frontend subscription with no backend broadcaster listens
forever to nothing.

```bash
# Backend broadcasts 'widget.updated' — who listens?
grep -rn "widget.updated" {{FRONTEND_DIR}}/src/

# Inverse: frontend subscribes — who broadcasts?
grep -rn "widget.updated" {{BACKEND_DIR}}/
```

**Zero hits means:** one half of a real-time pair was built. Both halves pass
their own tests; the feature does not exist.

**Remedy:** add the missing half — subscription handler in the frontend, or
broadcaster in the backend — and verify the channel itself is authorized in
your channel-registration file.

## Check C — Setting/flag defined but never read

**Catches:** a configuration key seeded into settings that no runtime code
consults. The admin UI shows a toggle; flipping it does nothing.

```bash
# For each new key written by a seeder/defaults file:
grep -rn "get('widgets.auto_archive_enabled'" {{BACKEND_DIR}}/
```

**Zero hits means:** the setting has no effect. This is worse than a missing
feature — it is a *lying* control surface.

**Remedy:** add the runtime guard that reads the key at the decision point it
is supposed to govern. Match the key string exactly
([pattern 2](04-bug-patterns.md)).

## Check D — Permission defined but never enforced

**Catches:** a permission seeded into the roles system that no route
middleware or ability check references. Users see or don't see things by
accident, and the security model silently diverges from the seeded intent.

```bash
# For each new permission in the roles/permissions seeder:
grep -rn "permission:widgets.export\|can('widgets.export'" {{BACKEND_DIR}}/ routes/ {{FRONTEND_DIR}}/src/
```

**Zero hits means:** the permission is decorative. Anyone authenticated can do
the thing it was meant to gate.

**Remedy:** attach it — route middleware, controller authorization check, or
both — and gate the corresponding UI affordance on the same permission.

## Check E — Background job not registered with the worker system

**Catches:** a job dispatched to a named queue that the worker configuration
does not consume. Dispatch succeeds, the job sits in the queue forever, and no
error is ever raised.

```bash
# For each job using a named queue:
grep -rn "onQueue('reports')" {{BACKEND_DIR}}/
# ...that queue must appear in the worker configuration:
grep -rn "'reports'" config/
```

**Zero hits in the worker config means:** silent failure by design — the
single most invisible bug in this list.

**Remedy:** register the queue (including any per-queue wait/timeout entries
your worker system requires). Then remember the operational twin:
`{{QUEUE_RESTART_COMMAND}}` on deploy, or workers keep running stale code
([chapter 08, note 4](08-field-notes.md)).

## Check F — Duplicate scheduled tasks

**Catches:** two schedule entries firing at the same time with the same
effect — typically a new task replacing a legacy one that nobody disabled.
Users get two reminders; ledgers get two writes.

```bash
# For each new schedule entry, look for neighbors at the same time:
grep -B2 -A2 "dailyAt('08:00')" routes/console.php   # or your schedule file
```

**Interpretation flips here:** it is *nonzero* unexpected hits that indicate
the problem — another task at the same time producing the same
notification/effect.

**Remedy:** disable the superseded entry in the same change that introduces
its replacement, and verify overlap guards (`withoutOverlapping`,
single-server) on anything mutating shared state.

## Check G — Referenced column doesn't exist

**Catches:** a relation select list or eager-load column spec naming a column
the table doesn't have. Depending on the framework this fails silently (the
attribute resolves to null) or 500s only in production paths tests don't
exercise.

```bash
# Find explicit column lists on relations:
grep -rn "with(\['" {{BACKEND_DIR}}/ | grep ":id,"
```

Then, for each hit, open the related table's migration/schema and verify every
listed column exists. Watch the accessor trap: if the model *computes* `name`
from `display_name`, the select list must include `display_name` — selecting
`name` is exactly the silent failure this check exists for.

**Remedy:** correct the column list to real schema columns, including every
column any accessor in the projection depends on.

## Check H — Frontend expects keys the API never sends

**Catches:** the consuming half of
[pattern 1](04-bug-patterns.md) — a component reading response keys the
controller doesn't emit. There is no single grep; this is a pairwise read:

```bash
# 1. Find the API calls in the changed frontend files:
grep -rn "api\.\|fetch(\|axios\." {{FRONTEND_DIR}}/src/ --include="*.tsx" --include="*.ts"
# 2. For each URL → find its route → read the controller's literal response keys.
# 3. Read the keys the component/hook consumes. Diff the two lists by hand.
```

**A mismatch means:** blank UI, `undefined` labels — never an exception.

**Remedy:** **prefer adding alias keys to the controller response** over
rewriting the frontend or adding silent fallbacks — the alias makes the
contract explicit and visible in one place. Then record the shape in the
resource's single data-access module so the next mismatch has one place to
happen.

---

## Mapping to the bug-pattern library

| [Chapter 04](04-bug-patterns.md) pattern | Phantom checks |
| :--- | :--- |
| 1 — Frontend ↔ API contract mismatch | **H** (plus **G** when the miss is a column list) |
| 2 — Enum/config-key string mismatch | **C**, **G** (exact-string halves) |
| 3 — Phantom features | **A**, **B**, **C**, **D** |
| 4 — Job/schedule wiring gaps | **E**, **F** |
| 5 — Documentation/code drift | — (owned by `documentation-agent`, Stage 5) |

## Ownership, restated

At pipeline time these checks belong to `integration-agent` — it runs them,
**fixes** the gaps directly ("connect, not rebuild"), runs `{{LINT_COMMAND}}`
on anything it touched, and reports `INTEGRATION COMPLETE ✅` or
`INTEGRATION BLOCKED ⚠️`; anything non-trivial it changed goes to the scoped
QA delta re-check before Checkpoint C ([chapter 01](01-pipeline.md), Stage 4).
QA does not duplicate these checks in Stage 3 — it owns the diff-correctness
patterns instead, and each agent's template points at the other's scope. The
codebase-wide re-run belongs to the standing sweep
([chapter 07](07-standing-routines.md)).

---

<div align="center">

[← 04 — Bug Patterns](04-bug-patterns.md) &nbsp;·&nbsp; [Playbook index](README.md) &nbsp;·&nbsp; [06 — The Memory System →](06-memory-system.md)

</div>
