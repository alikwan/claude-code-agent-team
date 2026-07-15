<img src="../assets/icons/templates.svg" width="56" height="56" alt="">

# Templates

[**← repository root**](../README.md) &nbsp;·&nbsp; `templates/` — files that live in **your** repository after adaptation

---

Unlike [`prompts/`](../prompts), which are pasted into sessions, these are copied in,
placeholder-filled (see [docs/placeholders.md](../docs/placeholders.md)), and committed.

| File | Copies to |
| :--- | :--- |
| `CLAUDE.md.template` | `CLAUDE.md` (or generate it with [`generate-your-claude-md.md`](generate-your-claude-md.md)) |
| `MEMORY.md.template` | `.claude/agent-memory/<agent>/MEMORY.md` (one per agent) |
| `settings.json.template` | `.claude/settings.json` |
| `pipeline-state.md.template`, `KNOWN_FAILURES.md.template` | per-task / repo root |
| [`api-contract.md`](api-contract.md), [`pr-body.md`](pr-body.md) | reference formats used by the orchestrator |
| `ci/*.yml.template` | `.github/workflows/*.yml` |

> [!NOTE]
> `ci/*.yml.template` files are renamed to `.yml` when copied into `.github/workflows/` — the
> `.template` suffix prevents accidental activation here in this repository.

---

<div align="center">

[README](../README.md) · [Playbook](../playbook/) · [Agents](../agents/) · [Prompts](../prompts/) · [Examples](../examples/)

</div>
