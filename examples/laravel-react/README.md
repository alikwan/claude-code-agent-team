# Laravel + React — the complete worked example

This directory is the generic pipeline made concrete: the templates from
[`agents/`](../../agents), [`prompts/`](../../prompts), and
[`templates/`](../../templates) adapted for one real stack, with every
placeholder resolved to an actual value. If the playbook chapters are the
theory, this directory is what a project looks like *after* adoption.

> **Provenance & honesty note.** Adapted from the production methodology
> artifacts of a real Arabic-first collections platform (deliberately
> unnamed) that shipped 200+ releases through this pipeline.
> Details that could identify the operator's clients or infrastructure were
> removed or replaced; the frontend was converted to the author's current
> recommended stack (React) — the original production system ran a different
> SPA framework, and the methodology carried over unchanged, which is rather
> the point.

The concrete values describe **Acme Collect**, a fictionalized stand-in with
the same shape as the original: an Arabic-first debt-collection platform,
built as a monorepo (`backend/` Laravel 12 API + `frontend/` React 19 SPA).

## What's in this directory

| File | What it is |
| :--- | :--- |
| [`CLAUDE.md`](CLAUDE.md) | The project constitution — the file every agent reads first |
| [`docs-structure.md`](docs-structure.md) | Documentation governance tree + doc-update map, as practiced |
| [`memory-example.md`](memory-example.md) | An annotated QA agent memory excerpt showing what earns a place |
| [`prompts/daily-code-review.ar.md`](prompts/daily-code-review.ar.md) | The authentic Arabic daily correctness-review prompt, stack-transposed |
| [`prompts/recurring-patterns-sweep.ar.md`](prompts/recurring-patterns-sweep.ar.md) | The authentic Arabic 5-pattern full-codebase sweep |
| [`prompts/security-sweep.ar.md`](prompts/security-sweep.ar.md) | The authentic Arabic 4-surface security sweep |

The three Arabic prompts are field artifacts — the original operational voice,
sanitized. Generic English adaptations live in [`../../prompts/`](../../prompts).

## Placeholder → concrete mapping

Every placeholder registered in [`docs/placeholders.md`](../../docs/placeholders.md)
with its Acme Collect value. This table is the **only** place in `examples/`
where `{{...}}` tokens may appear (CI enforces this).

| Placeholder | Acme Collect value |
| :--- | :--- |
| `{{PROJECT_NAME}}` | `Acme Collect` |
| `{{PROJECT_DESCRIPTION}}` | `an Arabic-first debt-collection platform` |
| `{{REPO_SLUG}}` | `your-org/acme-collect` |
| `{{MAIN_BRANCH}}` | `develop` |
| `{{PRODUCTION_BRANCH}}` | `main` |
| `{{BACKEND_DIR}}` | `backend` |
| `{{FRONTEND_DIR}}` | `frontend` |
| `{{BACKEND_STACK}}` | `Laravel 12 (PHP 8.3)` |
| `{{FRONTEND_STACK}}` | `React 19 + TypeScript + Vite` |
| `{{DEV_COMMAND_PREFIX}}` | `docker exec app` |
| `{{TEST_COMMAND}}` | `php artisan test` |
| `{{TEST_FILTER_FLAG}}` | `--filter=` |
| `{{FRONTEND_TEST_COMMAND}}` | `npm --prefix frontend run test` |
| `{{FULL_SUITE_POLICY}}` | `targeted-only` (CI runs the full suite on every PR) |
| `{{LINT_COMMAND}}` | `composer exec pint` |
| `{{FRONTEND_LINT_COMMAND}}` | `npm --prefix frontend run lint` |
| `{{BUILD_COMMAND}}` | `npm --prefix frontend run build` |
| `{{CACHE_CLEAR_COMMAND}}` | `docker exec app php artisan optimize:clear` |
| `{{QUEUE_RESTART_COMMAND}}` | `docker exec app php artisan horizon:terminate` |
| `{{TEST_DB_NAME}}` | `acme_test` |
| `{{PRIMARY_LANGUAGE}}` | `Arabic` |
| `{{MODEL_STRONG}}` | your platform's strongest model tier (never pin a model ID) |
| `{{MODEL_FAST}}` | a fast, cheaper model tier (never pin a model ID) |
| `{{AUTO_VERSION}}` | `false` (the human approves every version bump) |

## The 5 most instructive generic → concrete transformations

1. **Test invocation gains its environment prefix.**
   `{{DEV_COMMAND_PREFIX}} {{TEST_COMMAND}}` → `docker exec app php artisan test`.
   The prefix is a separate placeholder because production runs the same
   command *without* it — agents must never hardcode the container name inside
   application scripts.

2. **The `{{PRIMARY_LANGUAGE}}`-first rule becomes a real design system.**
   The generic rule ("all user-facing text in the primary language") expands
   into concrete Arabic-first obligations: `dir="rtl"` at the app root,
   Tailwind logical utilities (`ms-*`/`me-*`, `text-start`) instead of
   left/right, `rtl:`/`ltr:` modifiers only for genuinely directional cases,
   and Arabic activity-log and error messages. One placeholder, a page of
   consequences — see [`CLAUDE.md`](CLAUDE.md) §8.

3. **The fetch-layer rule becomes TanStack Query custom hooks.**
   "Frontend data access goes through one named layer" → one custom hook per
   resource (`usePayments()`, `useDebtors()`) wrapping TanStack Query, with
   response keys matched against the *actual* controller output — the
   pipeline's most common cross-boundary bug lives exactly here.

4. **`{{TEST_DB_NAME}}` becomes a hard safety rail, not documentation.**
   `acme_test` is the only database any agent may migrate-fresh or reset —
   stated in every spawn prompt *and* enforced in the permission settings.
   Prose alone failed the original project once.

5. **`{{QUEUE_RESTART_COMMAND}}` encodes an operational scar.**
   `docker exec app php artisan horizon:terminate` exists as a first-class
   placeholder because queue workers cache loaded code: a deployed fix that
   changes job/listener behavior silently does not run until workers restart.
   The original project rediscovered this enough times to promote the command
   into the constitution.

---

*Verified against Claude Code — 2026-07. Agent frontmatter, tool allowlists,
and sub-agent spawning behavior match the platform as of that date; re-verify
this directory after major Claude Code platform updates.*
