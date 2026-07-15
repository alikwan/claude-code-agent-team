[**← repository root**](../README.md) &nbsp;·&nbsp; `docs/`

# Placeholder Registry

Every generic file in this repository (`agents/`, `prompts/`, `templates/`,
`playbook/`) is parameterized with placeholders in `{{UPPER_SNAKE}}` form.
Adopting the pipeline is a one-pass find-and-replace of this table — nothing
else needs editing.

Two rules keep this system honest (both enforced by CI):

1. **Every placeholder used anywhere must be registered in this table.**
   Do not invent unregistered tokens.
2. **`examples/` contains zero placeholders** — it exists to show concrete
   values. The only exception is the mapping table in
   [examples/laravel-react/README.md](../examples/laravel-react/README.md).

## The registry

| Placeholder | Meaning | Example value (Laravel + React) |
| :--- | :--- | :--- |
| `{{PROJECT_NAME}}` | Your project's name, as agents should refer to it | `Acme Collect` |
| `{{PROJECT_DESCRIPTION}}` | One-line description of what the system does | `a subscription billing platform` |
| `{{REPO_SLUG}}` | GitHub `owner/repo` used in `gh` commands | `your-org/acme-collect` |
| `{{MAIN_BRANCH}}` | The integration branch PRs target and feature branches fork from | `develop` |
| `{{PRODUCTION_BRANCH}}` | The branch that represents production | `main` |
| `{{BACKEND_DIR}}` | Backend root directory (repo-relative; `.` if not a monorepo) | `backend` |
| `{{FRONTEND_DIR}}` | Frontend root directory (repo-relative; `.` if not a monorepo) | `frontend` |
| `{{BACKEND_STACK}}` | Backend framework + language version | `Laravel 12 (PHP 8.3)` |
| `{{FRONTEND_STACK}}` | Frontend framework + tooling | `React 19 + TypeScript + Vite` |
| `{{DEV_COMMAND_PREFIX}}` | Prefix for running backend commands in dev (empty if native) | `docker exec app` |
| `{{TEST_COMMAND}}` | Backend test runner invocation | `php artisan test` |
| `{{TEST_FILTER_FLAG}}` | The flag that scopes the test runner to named tests | `--filter=` |
| `{{FRONTEND_TEST_COMMAND}}` | Frontend test runner invocation | `npm --prefix frontend run test` |
| `{{FULL_SUITE_POLICY}}` | `targeted-only` or `full-suite-allowed` — see the invariant below | `targeted-only` |
| `{{LINT_COMMAND}}` | Backend lint/format command | `composer exec pint` |
| `{{FRONTEND_LINT_COMMAND}}` | Frontend lint command | `npm --prefix frontend run lint` |
| `{{BUILD_COMMAND}}` | Frontend production build command | `npm --prefix frontend run build` |
| `{{CACHE_CLEAR_COMMAND}}` | Command(s) that clear framework caches after config/route changes | `docker exec app php artisan optimize:clear` |
| `{{QUEUE_RESTART_COMMAND}}` | Command that restarts background workers so they load new code | `docker exec app php artisan horizon:terminate` |
| `{{TEST_DB_NAME}}` | The **only** database any agent is ever allowed to reset | `acme_test` |
| `{{PRIMARY_LANGUAGE}}` | The language of all user-facing text, logs, and notifications | `Arabic` |
| `{{MODEL_STRONG}}` | Model tier for builders, QA, and integration (your strongest) | *(your platform's strongest model)* |
| `{{MODEL_FAST}}` | Model tier for documentation/security/migration reviewers (cheaper) | *(a fast, cheaper model)* |
| `{{AUTO_VERSION}}` | `true` to skip the human version-approval gate in Stage 5 | `false` |

> **The full-suite invariant.** `{{FULL_SUITE_POLICY}}: targeted-only` is safe
> **only if CI runs the complete test suite on every PR** before merge. If you
> have no CI backstop, set `full-suite-allowed` so QA runs everything
> in-pipeline. The full suite must run *somewhere* before merge — pick where.

## Setup checklist

1. Copy the directories you need into your project (see
   [getting-started](getting-started.md)).
2. Fill the **Example value** column above with your own values.
3. Find-and-replace across the copied files. The portable way (identical on
   macOS and Linux, and immune to `/` or `&` in your values):

   ```bash
   grep -rl '{{PROJECT_NAME}}' .claude/ | xargs perl -pi -e 's/\Q{{PROJECT_NAME}}\E/Acme Collect/g'
   ```

   (If you prefer `sed`: it's `sed -i ''` on macOS but `sed -i` on GNU/Linux,
   and you must escape any `/` or `&` inside replacement values — the `perl`
   form avoids both traps.)

4. **Empty values leave stray spaces.** If a placeholder resolves to empty
   (commonly `{{DEV_COMMAND_PREFIX}}` for native, non-container dev), patterns
   like `{{DEV_COMMAND_PREFIX}} {{TEST_COMMAND}}` are left with a leading
   space — notably inside `.claude/settings.json` permission patterns, where a
   leading space means the rule **never matches**. Sweep them:

   ```bash
   grep -rn '( \| \{2\}' .claude/settings.json && echo "fix the spaces above"
   ```

5. Verify no placeholder was missed — this must return **no output**
   (`${{ ... }}` in the CI templates is GitHub Actions syntax, not a
   placeholder, and is correctly ignored by this pattern):

   ```bash
   grep -rnE '\{\{[A-Z][A-Z0-9_]*\}\}' .claude/ CLAUDE.md
   ```

That final grep is the same consistency gate this repository runs on itself in
CI — adopt it as a habit.

---

<div align="center">

[README](../README.md) · [Playbook](../playbook/) · [Agents](../agents/) · [FAQ](faq.md) · [Getting started](getting-started.md)

</div>
