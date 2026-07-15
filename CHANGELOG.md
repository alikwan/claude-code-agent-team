# Changelog

All notable changes to this repository are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

> [!NOTE]
> This file is itself an example of the changelog convention the playbook teaches
> (see [templates/CLAUDE.md.template](templates/CLAUDE.md.template), Versioning section):
> one heading per release, reverse-chronological, and every change ships with a
> changelog entry — no exceptions.

---

## [0.2.0] — 2026-07-15

### Changed

- **Full documentation redesign.** Restructured README (EN + AR) with a centered
  identity header, quick-nav links, GitHub alert callouts, and a consistent
  footer; every section README (`playbook/`, `agents/`, `prompts/`,
  `templates/`, `examples/`) now opens with its section icon and a breadcrumb
  back to the root, and closes with cross-section navigation.
- **New original visual identity**, hand-drawn as vector art in a single
  palette (blue `#2563EB`, orange accent `#F97316`, slate neutrals): logo mark
  (orchestrator-and-builders tree), typographic hero banner with an inline
  6-stage pipeline strip, a redrawn pipeline diagram (stages, both human gates,
  commit checkpoints A–D, hotfix-lane legend — bilingual labels), five section
  icons, and a bilingual social-preview card. Replaces the AI-generated art
  from 0.1.0.
- `agents/README.md` gained a full team table (role, write access, model tier).
- Docs pages (`docs/`, `playbook/` chapters) carry breadcrumb headers and
  prev/next navigation footers.

## [0.1.1] — 2026-07-14

### Changed

- The source production project is now fully unnamed throughout the repository
  (case study, its Arabic twin, and the example's provenance note), and its git
  history was rewritten to match — the experience transfers; the project's
  identity does not.
- Increased the worked example's fictional distance from its origin: domain
  strings that were byte-identical to the source project (doc filenames, a
  module path, service names, a setting key, table names) were renamed to
  fiction-preserving equivalents.

### Fixed

- SVG assets carry explicit intrinsic `width`/`height` again — without them,
  GitHub READMEs collapse the images (icons rendered at 15px).
- Component icons moved inline into the asset table's text cells — a dedicated
  icon column gets crushed when GitHub shrinks an overflowing table.

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
- Visual identity (AI-generated vector art, Recraft V4.1 via Higgsfield, with
  provenance signatures retained): logo mark, hero banner, five component
  icons, and a bilingual social-preview card composited locally.
