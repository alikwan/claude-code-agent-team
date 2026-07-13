# Changelog

All notable changes to this repository are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

> This file is itself an example of the changelog convention the playbook teaches
> (see [templates/CLAUDE.md.template](templates/CLAUDE.md.template), Versioning section):
> one heading per release, reverse-chronological, and every change ships with a
> changelog entry — no exceptions.

---

## [0.1.0] — 2026-07-13

### Added

- Initial public release: the complete multi-agent development methodology
  extracted from a production system (200+ releases shipped through it).
- `playbook/` — 9 methodology chapters: operating model, the 6-stage pipeline
  (v1.1), agent roles, communication protocol, the recurring-bug pattern
  library, phantom-feature checks, the agent memory system, standing review
  routines, and field notes from production.
- `agents/` — 9 generic Claude Code sub-agent templates (+1 optional
  test-engineer), fully placeholder-parameterized.
- `prompts/` — 3 standing review routines (daily code review,
  recurring-patterns sweep, security sweep) + sub-agent spawn templates.
- `templates/` — CLAUDE.md constitution template, a meta-prompt that generates
  a project-specific CLAUDE.md, agent memory scaffold, API contract format,
  PR body templates, known-failures ledger, pipeline state file, permissions
  settings template, and CI workflow templates.
- `examples/laravel-react/` — a complete worked adaptation for a
  Laravel 12 API + React SPA stack, including the three authentic Arabic
  production review prompts (sanitized).
- Bilingual documentation: full Arabic twins of the README, getting-started
  guide, methodology overview, and case study.
- Repository self-CI: sensitive-token sweep, placeholder-registry consistency,
  markdown lint, EN/AR parity, agent frontmatter lint, link checker, and
  release consistency checks.
