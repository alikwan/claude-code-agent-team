# Security Policy

This is a documentation and template repository — there is no runtime code to
exploit. Its security surface is nonetheless real, and reports are taken
seriously in three specific areas:

## 1. Leaked or insufficiently sanitized data

The content of this repository was extracted from a real production system and
passed through a multi-stage sanitization gate (an automated token sweep plus
human review in both English and Arabic). If you nonetheless find anything that
looks like a real credential, hostname, internal path, client-identifying
detail, or personal data:

**Do NOT open a public issue.** Use GitHub's
[private vulnerability reporting](https://github.com/alikwan/claude-code-agent-team/security/advisories/new) on this
repository. A public issue titled "credential visible in example prompt" is
itself a disclosure.

Such reports are treated as the highest-priority category and will be handled
(removed, history-rewritten if necessary, and acknowledged) as fast as
possible.

## 2. Flaws in the shipped templates

The files under `templates/ci/` and `templates/settings.json.template` are
meant to be copied into other people's repositories. A weakness there (an
overly broad permission, a workflow that leaks secrets into logs, a missing
`deny` rule for a destructive command) multiplies across every adopter.
Report these via private vulnerability reporting as well, or a public issue if
the impact is clearly theoretical.

## 3. Prompt-injection considerations in agent templates

The agent templates instruct AI agents that read repository content and run
shell commands. If you identify a realistic prompt-injection path enabled by
the *wording of these templates* (for example, an instruction that tells an
agent to obey text found inside reviewed files), open an issue or a private
report describing the scenario. Hardening the templates against instruction
smuggling is in scope for this project.

## Out of scope

- Vulnerabilities in Claude Code itself, GitHub Actions, or any third-party
  tool referenced here — report those to their respective vendors.
- Hypothetical misuse of the methodology (e.g., "an agent could be told to do
  X") that does not stem from the shipped template wording.

## Supported versions

Only the latest release on the default branch is supported. There are no
backports.
