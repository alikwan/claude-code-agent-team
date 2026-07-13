[**اقرأ هذا الدليل بالعربية ← README.ar.md**](README.ar.md)

# Claude Code Agent Team — a Battle-Tested Multi-Agent Development Pipeline

![License: MIT](https://img.shields.io/badge/license-MIT-blue)
![Made for Claude Code](https://img.shields.io/badge/made%20for-Claude%20Code-d97757)
![Docs](https://img.shields.io/badge/docs-EN%20%7C%20AR-green)
![Production proven](https://img.shields.io/badge/production-200%2B%20releases-brightgreen)

> Turn Claude Code into a coordinated AI dev team: 9 specialized sub-agents, a
> 6-stage delivery pipeline, agent memory, and review protocols — extracted
> from a real production system that shipped 200+ releases this way.

A production-proven playbook for multi-agent software development in Claude
Code: 9 specialized sub-agents working as one coordinated AI dev team. A
team-lead orchestrator drives backend, frontend, QA, integration,
security-review, migration-review, UI/UX, and documentation agents through a
6-stage pipeline with commit checkpoints and verdict protocols. Agents
accumulate knowledge in a per-agent memory system, recurring bugs feed a
pattern library, and standing routine prompts (daily code review, pattern
sweep, security sweep) keep quality from drifting. Includes a CLAUDE.md
constitution template, a meta-prompt that generates one for your project, and
a complete worked Laravel + React example. Everything is documented in both
English and Arabic.

**200+ production releases · 1 developer · 9 agents · a live Arabic-first platform**

## Who this is for (and not for)

**For you if:** you run Claude Code against a real, long-lived codebase — solo
or in a small team — and you want AI-built changes to arrive planned, reviewed,
integrated, documented, and committed with discipline instead of sprayed into
your working tree.

**Not for you if:** you want a fully autonomous agent swarm (look at CrewAI /
AutoGen — different shelf), you use a non-Claude-Code toolchain (the *method*
transfers, the agent-file format won't), or your project is a weekend
throwaway where a 6-stage pipeline costs more than it protects. See
[when *not* to use the full pipeline](docs/faq.md#when-should-i-not-use-the-full-pipeline).

## Set up a multi-agent team in Claude Code

Three copy-paste steps (full walkthrough: [getting started](docs/getting-started.md)):

**1 — Copy the agent templates into your project:**

```bash
npx degit alikwan/claude-code-agent-team/agents .claude/agents --force
rm .claude/agents/README.md      # docs, not an agent definition
rm -r .claude/agents/optional    # keep only if you want the optional test-engineer
npx degit alikwan/claude-code-agent-team/templates .claude/pipeline-templates --force
```

**2 — Generate your project constitution.** Open Claude Code at your repo root
and paste the meta-prompt from
[`templates/generate-your-claude-md.md`](templates/generate-your-claude-md.md) —
it explores your repo, asks you at most 6 questions, and writes a complete
`CLAUDE.md`.

**3 — Fill the placeholders and run your first task:**

```bash
# replace the {{PLACEHOLDER}} tokens per docs/placeholders.md, then verify —
# must return nothing (${{ ... }} in CI templates is GitHub Actions syntax, not a placeholder):
grep -rnE '\{\{[A-Z][A-Z0-9_]*\}\}' .claude/ CLAUDE.md

claude --agent team-lead   # the orchestrator IS your main session
                           # (or pick team-lead from the agent selector)
```

Give it a real task. It will read your constitution, present a plan, wait for
your approval, and drive the team.

## How the 6-stage pipeline works

![Claude Code multi-agent team 6-stage development pipeline diagram: a team-lead orchestrator drives Plan, parallel Build, Review, Integrate, Version and Document, and Release stages with two human approval gates and four commit checkpoints](assets/pipeline-diagram.svg)

Two human gates (you approve the **plan**, then the **version**), four commit
checkpoints (every stage leaves inspectable history), verdicts as exact
machine-parsed strings (`QA APPROVED ✅` — never "looks mostly fine"), and one
iron rule: **only the orchestrator touches git.** Full mechanics:
[playbook/01-pipeline.md](playbook/01-pipeline.md).

## What you get

| Asset | Where | What it is |
| :--- | :--- | :--- |
| **The playbook** | [`playbook/`](playbook/) | 9 chapters: pipeline, roles, protocols, bug patterns, phantom checks, memory, routines, field notes |
| **9 agent templates** (+1 optional) | [`agents/`](agents/) | Claude Code sub-agent files, fully parameterized — copy, find-replace, run |
| **Standing review routines** | [`prompts/`](prompts/) | Daily code review, recurring-pattern sweep, 4-surface security sweep — schedule and forget |
| **Templates** | [`templates/`](templates/) | CLAUDE.md constitution + generator meta-prompt, agent memory, API contract, PR bodies, permissions, CI |
| **A complete worked example** | [`examples/laravel-react/`](examples/laravel-react/) | Every generic file made concrete for a Laravel 12 API + React SPA monorepo — including the authentic Arabic production review prompts |

## How is this different from other agent repos?

- **Not a prompt collection.** Every file interlocks: agents cite the pipeline,
  the pipeline cites the verdict protocol, CI enforces the vocabulary. It's an
  integrated methodology, not snippets.
- **Not an autonomous framework.** This is human-in-the-loop by design — two
  approval gates, git discipline, and receipts at every stage. The goal is
  trustworthy changes to a codebase you care about, not maximum autonomy.
- **Earned, not invented.** Every rule traces to something that actually broke
  in production. The [bug-pattern library](playbook/04-bug-patterns.md) came
  from ~30 real review blockers; the [field notes](playbook/08-field-notes.md)
  document our failures, including the ones this pipeline's v1.1 fixes.

## The 9 agent roles (sub-agent templates)

| Agent | Job | Key constraint |
| :--- | :--- | :--- |
| `team-lead` | Plans, orchestrates, owns **all** git, opens the PR | Cannot write files — must delegate |
| `backend-dev` / `frontend-dev` | Build in parallel against a shared API contract | Never run git |
| `qa-engineer` | Reviews the diff, runs tests + lint, exercises the flow | Read-only — judges can't hold pens |
| `security-reviewer` | 10-category security checklist (conditional) | Read-only, cheap model |
| `migration-reviewer` | Database migration safety (conditional) | Read-only, cheap model |
| `integration-agent` | Finds and fixes "phantom features" — built but never wired | Connects, never judges |
| `documentation-agent` | Blocking docs audit — CHANGELOG always | No PR until COMPLETE |
| `ui-ux-designer` | Design specs before code (optional) | RTL/Arabic-first aware |

Why each exists — and which three you can start with:
[playbook/02-roles.md](playbook/02-roles.md).

## The story

One developer, one large Arabic-first collections platform, and the discovery
that a single AI session writing code unsupervised produces exactly the bugs
you'd expect from a developer with no memory, no reviewer, and no fear.
The pipeline condensed out of fixing that — release after release, for 200+
releases. Read the honest version, including what still doesn't work:
[docs/case-study.md](docs/case-study.md).

هذه المنهجية موثّقة بالكامل بالعربية أيضًا — [ابدأ من هنا](README.ar.md).

## Repository map

```
playbook/     the methodology, 9 chapters — the "why and how"
agents/       10 sub-agent templates ({{parameterized}})
prompts/      standing routines — pasted into sessions on a schedule
templates/    files that live in YOUR repo after adaptation
docs/         using this repo: getting started, placeholders, case study, FAQ (+ Arabic twins in docs/ar/)
examples/     laravel-react/ — every generic file made concrete
```

## FAQ, contributing, license

- Common questions — cost, minimal subsets, non-Laravel stacks:
  [docs/faq.md](docs/faq.md)
- Contributions welcome — especially `examples/<your-stack>/` adaptations and
  evidence-backed bug patterns: [CONTRIBUTING.md](CONTRIBUTING.md)
- [MIT](LICENSE) © 2026 Ali Kwan
