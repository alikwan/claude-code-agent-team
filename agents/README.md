# Agent Templates

The nine agent definitions that make up the pipeline team — plus the optional
tenth (`optional/test-engineer.md`) — generalized from the
production originals that shipped 200+ releases. Each file is a complete,
drop-in Claude Code agent: YAML frontmatter (name, description, tools, model
tier) plus the role's workflow, critical rules, and verbatim report formats.

Every template follows the same canonical anatomy — Role, Scope, Workflow,
Critical rules, Report format, Persistent memory — explained in
[playbook/02-roles.md](../playbook/02-roles.md). The verdict strings inside
these files are contracts; they must match
[playbook/03-communication-protocol.md](../playbook/03-communication-protocol.md)
exactly.

**Install:** copy the files you need into your project's `.claude/agents/`
(including `optional/test-engineer.md` if you carry test debt), then run the
one-pass placeholder find-and-replace described in
[docs/placeholders.md](../docs/placeholders.md). Verify with
`grep -rnE '\{\{[A-Z][A-Z0-9_]*\}\}' .claude/agents/` — it must return nothing.

Run `team-lead` as your **main session** (`claude --agent team-lead`) — never
via the Task tool. All other agents are spawned by the orchestrator.
