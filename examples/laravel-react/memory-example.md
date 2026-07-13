# Agent Memory — annotated example (qa-engineer)

A fictionalized but structurally faithful excerpt of a QA agent's persistent
memory file (`.claude/agent-memory/qa-engineer/MEMORY.md`) as it looks after
months of production use. Italic lines are editorial annotations explaining
*why each entry earns its place* — they are not part of the memory file.

---

```markdown
# QA Engineer — Persistent Memory

This file is loaded into every QA agent session. Keep it concise (max 200
lines). Add notes about recurring issues, known weak spots, and patterns
discovered across reviews. Never store secrets, tokens, or credentials here.
```

## Recurring Issues to Always Check

**1. Form Requests created but controllers still validate inline** — a pattern
seen repeatedly: `StorePaymentRequest` exists in `backend/app/Http/Requests/`
but `PaymentController::store()` still calls `$request->validate()` inline.
Always verify the controller type-hints the Form Request class.

*Earns its place: a builder habit the pipeline keeps reproducing — the class
gets created to satisfy the constitution, the wiring gets forgotten. Cheap to
check, caught three times before it was recorded.*

**2. Response-envelope key drift between `PaymentController` and the
`usePayments` hook** — the controller returns `{data: {payments: [...]}}`
while the hook's TypeScript type claims `{data: Payment[]}`. Types are written
by the frontend builder from the *contract*; verify them against the
controller's **actual** JSON output, not the contract prose.

*Earns its place: the single most common cross-boundary bug in this codebase.
Naming the exact pair that drifted last time gives the next QA run a concrete
first place to look.*

**3. Scheduled-job duplicates sharing an idempotency flag** — a reminder job
and an artisan command both keyed on `reminder_sent_at`; when both ran on the
same day, debtors received duplicate messages. When reviewing any new
scheduled entry in `backend/routes/console.php`, check whether another
schedule or command writes the same flag.

*Earns its place: invisible in any single diff — only a reviewer with memory
of the first incident connects a new schedule entry to an old flag.*

## Known Weak Spots

**1. `ContractArchiveTest::it_archives_settled_contracts` — PRE-EXISTING
FAILURE, not a regression.** Broken since the contract-schema redesign
(commit `3f9c2ab`, two releases before the current work); the test was never
updated. Do NOT block unrelated PRs on it; do not re-diagnose it from scratch.

*Earns its place: commit provenance is what makes "pre-existing" a verifiable
claim instead of an excuse. Without this entry, every QA cycle burns time
rediscovering — or worse, mis-attributing — the same red test.*

**2. `backend/app/Outreach/` — chronic lint debt.** Roughly nine files carry
long-standing PSR-12 violations (operator spacing, concatenation spacing).
Running the linter repo-wide produces noise unrelated to the diff under
review; lint **scoped to changed files** and note the module's debt separately.

*Earns its place: prevents a known-dirty module from turning every unrelated
review into a false REJECTED.*

## Approved Exceptions

**1. Webhook signature verification is bypassed when the environment is
`local` or `testing`.** Reviewed and accepted by the owner — required for
local webhook simulation. Flag it ONLY if `production` ever appears in the
bypass list.

*Earns its place: an unrecorded exception is re-flagged forever. Recording the
precise boundary ("only if production enters the list") keeps the check alive
without the noise.*

## Disproven Findings

**1. "`QueueService::nextForAgent()` race allows double-assignment"** —
investigated in depth: the `SELECT ... FOR UPDATE` inside the surrounding
transaction covers the window; the 5-minute `locked_until` is a UX guard, not
the concurrency mechanism. Disproven; do not re-report.

**2. "`PaymentAllocationService` drops the rounding remainder on allocation"** —
the remainder is *intentionally* folded into the last installment per the
money rules (CLAUDE.md §5), and the helper tests assert it. Disproven; do not
re-report.

*Both earn their place for the same reason: a plausible-looking finding that
was expensively disproven once will be expensively re-found by every future
session unless the disproof is written down. Disproven findings are the
highest-value entries in the file — they convert wasted cycles into a lookup.*

---

## Governance note

The header's line cap (max 200 lines) is enforced, not aspirational: memory
that grows unboundedly stops being read. Prune superseded entries when adding
new ones. And the **no-secrets rule is absolute** — no tokens, passwords,
connection strings, or PII, ever. Memory files travel with the repo: anything
that lands in one is one `git push` away from being public, and any credential
that appears in a memory file must be treated as compromised.
