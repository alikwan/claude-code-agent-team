<img src="../assets/icons/agents.svg" width="56" height="56" alt="">

# Agent Templates

[**← repository root**](../README.md) &nbsp;·&nbsp; `agents/` — the nine sub-agent definitions (+1 optional)

---

The nine agent definitions that make up the pipeline team — plus the optional tenth
(`optional/test-engineer.md`) — generalized from the production originals that shipped 200+
releases. Each file is a complete, drop-in Claude Code agent: YAML frontmatter (name,
description, tools, model tier) plus the role's workflow, critical rules, and verbatim report
formats.

Every template follows the same canonical anatomy — **Role, Scope, Workflow, Critical rules,
Report format, Persistent memory** — explained in [playbook/02-roles.md](../playbook/02-roles.md).

> [!IMPORTANT]
> The verdict strings inside these files are contracts; they must match
> [playbook/03-communication-protocol.md](../playbook/03-communication-protocol.md) exactly.

## The team

| File | Role | Writes? | Model tier |
| :--- | :--- | :--- | :--- |
| [`team-lead.md`](team-lead.md) | Orchestrator — plans, delegates, owns all git | git only | strong |
| [`backend-dev.md`](backend-dev.md) | Backend builder | ✅ | strong |
| [`frontend-dev.md`](frontend-dev.md) | Frontend builder | ✅ | strong |
| [`qa-engineer.md`](qa-engineer.md) | Reviews the diff, runs tests + lint | read-only | strong |
| [`integration-agent.md`](integration-agent.md) | Wires phantom features | ✅ | strong |
| [`security-reviewer.md`](security-reviewer.md) | 10-category security checklist (conditional) | read-only | fast |
| [`migration-reviewer.md`](migration-reviewer.md) | Migration safety (conditional) | read-only | fast |
| [`documentation-agent.md`](documentation-agent.md) | Blocking docs audit | docs only | fast |
| [`ui-ux-designer.md`](ui-ux-designer.md) | Design specs before code (optional) | specs only | strong |
| [`optional/test-engineer.md`](optional/test-engineer.md) | Test-debt remediation (optional) | tests only | strong |

## Install

Copy the files you need into your project's `.claude/agents/` (including
`optional/test-engineer.md` if you carry test debt), then run the one-pass placeholder
find-and-replace described in [docs/placeholders.md](../docs/placeholders.md). Verify:

```bash
grep -rnE '\{\{[A-Z][A-Z0-9_]*\}\}' .claude/agents/   # must return nothing
```

> [!WARNING]
> Run `team-lead` as your **main session** (`claude --agent team-lead`) — never via the Task
> tool. A sub-agent cannot spawn sub-agents; the pipeline will stall at Stage 2. All other
> agents are spawned by the orchestrator.

---

<div align="center">

[README](../README.md) · [Playbook](../playbook/) · [Prompts](../prompts/) · [Templates](../templates/) · [Examples](../examples/)

</div>
