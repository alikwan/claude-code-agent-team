# templates/

Files that live in **your repository** after adaptation — unlike
[`prompts/`](../prompts), which are pasted into sessions, these are copied in,
placeholder-filled (see [docs/placeholders.md](../docs/placeholders.md)), and
committed.

| File | Copies to |
| :--- | :--- |
| `CLAUDE.md.template` | `CLAUDE.md` (or generate it with `generate-your-claude-md.md`) |
| `MEMORY.md.template` | `.claude/agent-memory/<agent>/MEMORY.md` (one per agent) |
| `settings.json.template` | `.claude/settings.json` |
| `pipeline-state.md.template`, `KNOWN_FAILURES.md.template` | per-task / repo root |
| `api-contract.md`, `pr-body.md` | reference formats used by the orchestrator |
| `ci/*.yml.template` | `.github/workflows/*.yml` |

`ci/*.yml.template` files are renamed to `.yml` when copied into
`.github/workflows/` — the `.template` suffix prevents accidental activation
here in this repository.
