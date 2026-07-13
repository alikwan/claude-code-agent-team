# Generate Your CLAUDE.md — the meta-prompt

Writing a project constitution by hand is the biggest single setup cost of
this pipeline. This prompt makes Claude Code do it: paste everything below the
divider into a Claude Code session **at your repository root**, and it will
explore the repo, ask you only what it cannot infer, and emit a `CLAUDE.md`
that follows the section order of
[CLAUDE.md.template](CLAUDE.md.template) — with every placeholder resolved to
a real, verified value.

Fill in any stack hints you already know before pasting; blank lines are fine.

---

**Stack hints** (optional — pre-seed anything you already know; the agent
detects the rest):

- Backend stack & version:
- Frontend stack & tooling:
- Package manager(s):
- Dev environment (native / containerized / other):
- CI system:
- Anything unusual about this repo:

You are generating this repository's `CLAUDE.md` — the project constitution
that every developer and AI agent must read before writing code. Its structure
must follow the section order of the pipeline's `CLAUDE.md.template`
(if `.claude/pipeline-templates/CLAUDE.md.template` exists in this repo,
mirror it; otherwise the section list below is the complete spec):
(1) header + project identity, (2) project philosophy, (3) tech stack,
(4) development commands, (5) architecture overview, (6) git workflow,
(7) the agent pipeline, (8) code style, (9) documentation governance,
(10) versioning.

Work in four phases. Do not skip or reorder them.

## Phase 1 — Explore (READ-ONLY)

Do not create, modify, or delete anything in this phase. Detect the following
from the repository itself, preferring evidence over inference:

1. **Stack & versions** — from manifests (`composer.json`, `package.json`,
   `pyproject.toml`, `go.mod`, `Gemfile`, `Cargo.toml`, ...).
2. **Package manager** — from lockfiles (`package-lock.json` vs `yarn.lock`
   vs `pnpm-lock.yaml`, etc.).
3. **Test, lint, and build commands** — from manifest scripts,
   `Makefile`/`justfile`, and above all CI workflow files
   (`.github/workflows/`, `.gitlab-ci.yml`). CI is the strongest evidence of
   what actually runs.
4. **Directory layout** — top two levels; identify the backend root, frontend
   root, docs tree, migrations, and seeders.
5. **Existing conventions** — commit style from `git log --oneline -30`,
   branch names from `git branch -a`, any existing `CLAUDE.md`, `CONTRIBUTING`,
   or `README` rules, lint configs, `.editorconfig`.
6. **Runtime surface** — databases, queues, schedulers, websockets — from
   config files and `.env.example`. Never read real `.env` values, and never
   copy secrets into anything you write.

Finish the phase by listing what you could **not** determine — that list
drives Phase 2.

## Phase 2 — Interview (at most 6 questions)

Ask me AT MOST six questions, in a single batch, covering only what Phase 1
could not infer. Candidate topics, in priority order:

1. **Deploy environment** — where does production run; is dev containerized?
2. **Branch strategy** — the integration branch PRs target vs the branch that
   represents production.
3. **User-facing language** — the language of all UI text, logs, and
   notifications.
4. **Versioning appetite** — SemVer with a `VERSION` file? Should version
   bumps be auto-approved or human-gated?
5. **Database names** — especially the **test database**, the only one agents
   will ever be allowed to reset.
6. **Background workers** — queue system, and the command that restarts
   workers so they load new code.

Never ask about something already answered by the stack hints above or by the
repository itself.

## Phase 3 — Emit CLAUDE.md

Write `CLAUDE.md` at the repository root, following the ten-section order
listed at the top of this prompt. Rules:

- **All placeholders resolved** — the file must contain real values only; no
  `{{...}}` token may remain and no guidance comments may survive.
- **Every command copy-paste runnable from the repo root** — bake in any
  container prefix; agents execute these lines verbatim.
- **Do not invent facts.** If a section is genuinely not applicable (e.g. no
  frontend), say so in one line rather than fabricating content.
- Keep it lean: link deep material into `docs/` instead of inlining it.

## Phase 4 — Self-check (do not skip)

1. **Run every command you documented.** For destructive or long-running
   commands, run `--help` or a dry-run variant instead. If a documented
   command fails, fix the documentation — the corrected command goes into
   `CLAUDE.md`, not into a footnote.
2. **Verify no placeholder survived:** `grep -rn '{{' CLAUDE.md` must return
   nothing.
3. **Re-read the final file top to bottom once**, as a brand-new agent would:
   every section must be actionable without asking a human.

When all checks pass, show me a short summary of what you detected versus what
I told you, then remind me to commit the result:
`git add CLAUDE.md && git commit -m "docs: add project constitution"`.
