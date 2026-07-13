# prompts/

Prompts you **paste into a session verbatim** — usually on a schedule
(standing routines) or at spawn time (templates). Nothing here is installed
into your repository; each file is consumed as a message.

| File | Use |
| :--- | :--- |
| [daily-code-review.md](daily-code-review.md) | Scheduled cloud/cron agent — daily correctness review of recent commits |
| [recurring-patterns-sweep.md](recurring-patterns-sweep.md) | Codebase-wide sweep of the 5 bug patterns + phantom checks A–H |
| [security-sweep.md](security-sweep.md) | 4-agent parallel security review by attack surface |
| [spawn-templates.md](spawn-templates.md) | Ready-to-adapt sub-agent spawn prompts for the orchestrator |

By contrast, [`templates/`](../templates) holds files that **live in your
repo** after adaptation (agent files, settings, memory seeds).
