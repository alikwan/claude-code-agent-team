# Contributing

Thanks for wanting to improve the playbook. This is a methodology repository —
contributions look different here than in a code repo, so please read this
short page first.

## What this repo accepts

| Contribution | Route | Bar |
| :--- | :--- | :--- |
| **A new stack example** (`examples/<your-stack>/`) | PR | The most valuable contribution this repo accepts. Follow the structure of `examples/laravel-react/` (README with mapping table + honesty note, concrete agents, CLAUDE.md). Zero placeholders, zero private data. |
| **A new bug pattern or phantom check** | [Pattern proposal issue](https://github.com/alikwan/claude-code-agent-team/issues/new?template=pattern-proposal.yml) first, then PR | Must include evidence from a real codebase. Patterns are earned, not brainstormed. |
| **An adaptation report** | [Adaptation report issue](https://github.com/alikwan/claude-code-agent-team/issues/new?template=adaptation-report.yml) | Honest negative reports are as welcome as successes. |
| **Docs fixes** (broken links, wrong instructions, typos) | PR directly | Small and obvious — just send it. |
| **Arabic translation fixes** | PR directly | Native-speaker corrections gladly taken. |
| **Agent template improvements** | Issue first | Templates encode production experience; changes need a "what breaks without this" story, not a style preference. |

**Not accepted:** rewrites of the methodology's core decisions (single git
owner, read-only judges, two human gates) without extraordinary evidence —
those decisions are the product; drive-by reorganizations; AI-generated
pattern lists without real-world provenance.

## The language policy (please read — it's unusual)

- **English is canonical.** Every normative file is English; the Arabic files
  are translations.
- Four files have full Arabic twins: `README.md`, `docs/getting-started.md`,
  `docs/case-study.md`, plus the consolidated `docs/ar/playbook-overview.ar.md`.
- **If your PR changes one of those English files**, either update the Arabic
  twin too, or tick the PR-template checkbox requesting maintainer sync (the
  `needs-ar-sync` label) — AR syncs are batched per release.
- Every Arabic file carries a header noting which English revision it was
  translated from. Don't remove it — it's how drift stays auditable.
- The three `*.ar.md` files under `examples/laravel-react/prompts/` are
  **authentic production artifacts**, not translations — they have no English
  twin by design.

## Ground rules

- **No private data, ever.** No credentials, internal hostnames, absolute
  local paths, client-identifying details, or unsanitized pastes from your
  production systems. CI scans for generic leak patterns; you are responsible
  for the specific ones. Found something already leaked? **Don't open an
  issue** — see [SECURITY.md](SECURITY.md).
- **Placeholders are a registry, not a convention.** Any new `{{TOKEN}}` must
  be added to [docs/placeholders.md](docs/placeholders.md) (CI enforces this);
  `examples/` must contain none.
- **Vocabulary is load-bearing.** Stage names, checkpoint letters, and verdict
  strings are defined in [playbook/03](playbook/03-communication-protocol.md)
  and repeated verbatim elsewhere. Never introduce a variant spelling.
- **Commits:** Conventional Commits (`feat:`, `fix:`, `docs:`, `chore:`,
  plus `security:`) — yes, we dogfood the convention the playbook teaches.

## Maintenance expectations

This repository is maintained by one person alongside the production system it
was extracted from. Responses are best-effort; PRs that follow this page get
reviewed much faster than ones that don't. The MIT license means forks are a
feature, not a betrayal.
