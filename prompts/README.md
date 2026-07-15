<img src="../assets/icons/prompts.svg" width="56" height="56" alt="">

# Standing Routines & Spawn Prompts

[**← repository root**](../README.md) &nbsp;·&nbsp; `prompts/` — pasted into sessions, never installed

---

Prompts you **paste into a session verbatim** — usually on a schedule (standing routines) or at
spawn time (templates). Nothing here is installed into your repository; each file is consumed
as a message.

| File | Use |
| :--- | :--- |
| [daily-code-review.md](daily-code-review.md) | Scheduled cloud/cron agent — daily correctness review of recent commits |
| [recurring-patterns-sweep.md](recurring-patterns-sweep.md) | Codebase-wide sweep of the 5 bug patterns + phantom checks A–H |
| [security-sweep.md](security-sweep.md) | 4-agent parallel security review by attack surface |
| [spawn-templates.md](spawn-templates.md) | Ready-to-adapt sub-agent spawn prompts for the orchestrator |

> [!NOTE]
> By contrast, [`templates/`](../templates) holds files that **live in your repo** after
> adaptation (agent files, settings, memory seeds). Scheduling guidance:
> [playbook/07-standing-routines.md](../playbook/07-standing-routines.md).
> The authentic Arabic production versions of the three routines live in
> [`examples/laravel-react/prompts/`](../examples/laravel-react/prompts/).

---

<div align="center">

[README](../README.md) · [Playbook](../playbook/) · [Agents](../agents/) · [Templates](../templates/) · [Examples](../examples/)

</div>
