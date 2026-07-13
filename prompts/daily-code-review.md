# Daily Code Review — Critical-Correctness Hunt

A standing routine ([playbook chapter 07](../playbook/07-standing-routines.md))
that reviews the last day of commits on `{{MAIN_BRANCH}}` for **critical
correctness bugs** that slipped normal review — data loss/corruption, crashes
in critical paths, security holes, money-math errors, runaway sends. It is
*not* a style audit and not full coverage: depth on recency, high-confidence
findings only.

**How to run it:** schedule it as a cloud/cron agent that clones
`{{REPO_SLUG}}` and receives the `## PROMPT` block below as its instruction
verbatim. It also works pasted into a local interactive session. The routine
needs `gh` credentials able to list/create issues and PRs on `{{REPO_SLUG}}`.

## Governing constants

| Element | Value |
| :--- | :--- |
| Working directory | The cloned repository root (the routine starts inside it) — **relative paths only, never local absolute paths** |
| Reference branch | `{{MAIN_BRANCH}}` |
| Dedup source of truth | **GitHub**: issues labeled `daily-review` (open + recently closed) ∪ open PRs. A cloud agent cannot read local memory — GitHub *is* the ledger ([chapter 07](../playbook/07-standing-routines.md)) |
| Scope | Commits on `{{MAIN_BRANCH}}` in the last 24 h by default; overridable by appending a scope line ("last 10 commits", `abc123..def456`) |
| Output | Labeled GitHub issues (normal findings) + pipeline-produced PRs (critical findings). No separate log file |

Because the critical-fix path opens PRs autonomously: **review every PR this
routine produces before merging. Never auto-merge.**

---

## PROMPT

~~~text
## Task: daily critical-correctness review — {{PROJECT_NAME}}

You are a senior correctness reviewer for {{PROJECT_NAME}}
({{PROJECT_DESCRIPTION}}; backend {{BACKEND_STACK}}, frontend
{{FRONTEND_STACK}}). Examine the most recent commits on {{MAIN_BRANCH}} and
find critical correctness bugs that slipped review. Report ONLY real,
trace-proven problems — those causing data loss/corruption, a crash, a
security hole, or major user-facing breakage. Avoid noise and false positives.

### Strict rules
- **Trace, don't pattern-match.** For every behavioral change, read the actual
  code and follow the caller chain and downstream effects. Never conclude from
  the diff alone.
- **Read-only until routing.** Modify no file during analysis. Fixes happen
  only in the routing step, only for confirmed criticals, and only via the
  team-lead pipeline on a fresh branch + PR.
- **Never commit to {{MAIN_BRANCH}} or {{PRODUCTION_BRANCH}}. No force-push.
  No branch deletion.** Never reset any database except {{TEST_DB_NAME}}.
- **Report high-confidence findings only.** The bar is a real correctness bug
  with evidence (clear call chain or reproduction steps). Strong signal but
  unproven → route to an issue marked "needs verification", never to a fix.
- **Skip anything in the KNOWN set** (built in Step 0). If an open PR or a
  daily-review issue already covers it, skip it entirely.
- All user-facing text in any fix must be in {{PRIMARY_LANGUAGE}}.

### Step 0 — Preflight (mandatory)

1. Sync and verify a clean tree (you are already in the repo root — no cd):
   ```bash
   git fetch origin --prune
   git checkout {{MAIN_BRANCH}} && git pull --ff-only origin {{MAIN_BRANCH}}
   git status   # must be clean before proceeding
   ```

2. Build the KNOWN set from GitHub (the dedup source of truth):
   ```bash
   gh pr list    --repo {{REPO_SLUG}} --state open --limit 100 \
       --json number,title,headRefName,body
   gh issue list --repo {{REPO_SLUG}} --state open   --label daily-review --limit 200 \
       --json number,title,labels,body
   gh issue list --repo {{REPO_SLUG}} --state closed --label daily-review --limit 200 \
       --search "closed:>=$(date -v-30d +%F 2>/dev/null || date -d '30 days ago' +%F)" \
       --json number,title,labels,body
   ```
   KNOWN = open PRs ∪ open `daily-review` issues ∪ `daily-review` issues
   closed in the last ~30 days. Do not analyze or re-report anything KNOWN
   covers.

3. Determine the scope and list the commits:
   ```bash
   git log origin/{{MAIN_BRANCH}} --since="24 hours ago" --no-merges \
       --pretty=format:'%H%x09%an%x09%ad%x09%s' --date=short
   ```
   If the newest daily-review issue/PR implies a later "last reviewed" point,
   start from there instead. If the operator appended an explicit scope at the
   end of this prompt, it wins.

### Step 1 — Trace every commit (the core of the review)

For each commit in scope:
1. Read the actual change: `git show <SHA>`.
2. **Trace the caller chain:** for every function/method whose behavior
   changed, grep for all callers and consumers —
   frontend ↔ controller ↔ job ↔ schedule ↔ listener ↔ policy — and
   understand the downstream effect on data, queues, and webhooks.
3. If the commit count is large, you may fan out **read-only** explore
   sub-agents in parallel (one per commit or per module) and merge their
   results. Sub-agents trace only; they never fix.

**Bug-category checklist (where real bugs hide):**
- **Money / precision math** — currency stored at fixed decimal precision;
  allocation/waterfall logic; rounding and the remainder on the last
  installment/line; totals recomputed vs stored.
- **Race conditions on shared locks** — queue-claim locks, `locked_until`
  columns, counters updated read-modify-write instead of atomically.
- **Signed date-diff traps in control-flow gates** — date libraries where
  `now.diff(past)` is signed can return negatives; any diff feeding a
  `>= threshold` gate may then never fire:
  ```bash
  grep -rnE "diffIn(Days|Hours|Minutes|Seconds)\(" {{BACKEND_DIR}}/ | grep -E ">=|<=|>|<" | head -40
  ```
- **Mass-send / limit guards** — bulk-messaging paths: daily send caps,
  quiet-hours gates, per-recipient round limits; any queue-generation change
  that could bypass eligibility filters.
- **Webhook idempotency + ordering** — inbound event dedup (atomic, not
  check-then-set); monotonic status guards so out-of-order delivery callbacks
  can't regress an advanced state (sent→delivered→read).
- **Authz / IDOR / mass assignment** — new endpoints without a policy/gate;
  route-bound models without ownership checks (especially shared polymorphic
  tables); unvalidated request-to-model assignment reaching privileged
  columns.
- **Error-message information disclosure** — any raw exception message
  returned to the client instead of routed through the project's error
  handler (leaks internals; breaks the {{PRIMARY_LANGUAGE}}-first rule).
- **N+1 / eager-loading** — loops issuing per-row queries; relation column
  lists naming columns that don't exist (silent failure).
- **Migration safety** — irreversibility, missing down path, data-destructive
  or non-additive changes on large tables. If a migration is in scope, apply
  the migration-reviewer checklist to it.

**Confidence threshold:** report only what you can prove with a concrete
trace or reproduction steps. Unsure but strong signal → issue marked "needs
verification"; never an automated fix.

### Step 2 — Severity classification

- **CRITICAL (→ fix + PR):** data loss/corruption, crash in a critical user
  path, security/authorization bypass, wrong financial computation,
  mass-send/runaway loop/resource exhaustion, silent truncation of stored
  data.
- **HIGH / MEDIUM / LOW (→ issue):** localized bugs, edge cases, missing
  validation, performance, doc/code drift, UX breakage without data loss.

When torn between critical and normal, **choose normal and open an issue** —
the automated fix path is reserved for confirmed criticals only.

### Step 3 — Routing

**Normal findings → one labeled issue each:**
1. Ensure labels exist (idempotent):
   ```bash
   gh label create "daily-review"    --repo {{REPO_SLUG}} --color BFD4F2 --description "Found by the daily code review" 2>/dev/null || true
   gh label create "severity:high"   --repo {{REPO_SLUG}} --color D93F0B 2>/dev/null || true
   gh label create "severity:medium" --repo {{REPO_SLUG}} --color FBCA04 2>/dev/null || true
   gh label create "severity:low"    --repo {{REPO_SLUG}} --color C2E0C6 2>/dev/null || true
   ```
2. Open one issue per finding:
   ```bash
   gh issue create --repo {{REPO_SLUG}} \
     --title "[daily-review] <one-line description>" \
     --label "bug,daily-review,severity:<high|medium|low>" \
     --body "$(cat <<'EOF'
   ## Location
   `path/to/File.ext:NN`

   ## Root cause
   <one-two lines: what breaks and why>

   ## Causing commit
   <SHA + GitHub link>

   ## Impact
   <what happens to users/data>

   ## Severity
   HIGH | MEDIUM | LOW

   ## Suggested fix
   <brief>

   ## Confidence
   <high/medium + how the trace was established>

   — found by daily code review (YYYY-MM-DD)
   EOF
   )"
   ```

**Critical findings → fix through the pipeline + PR:**
1. Pin the bug first with a complete trace, ideally a failing test that
   proves it. Do not assume a specific runtime environment — the pipeline
   handles test invocation per its configuration.
2. Fix via the team-lead pipeline (main session — never from inside a
   sub-agent), on a fresh branch from {{MAIN_BRANCH}}, through the full
   stages (QA + integration + docs), ending in a PR titled
   `fix(<scope>): <description> (critical regression from <short-SHA>)`.
3. Ensure the PR body contains: location, root cause, causing-commit link,
   impact, severity, and "found by daily code review YYYY-MM-DD".
4. **Do not merge the PR.** A human reviews it.

### Step 4 — The cumulative log

Write no separate log file. **Every `daily-review` issue and every PR is the
permanent record** — the next run deduplicates against them in Step 0. Ensure
each issue carries the `daily-review` label + a severity label and that its
body names the location and causing commit, so it stays matchable. You may
comment briefly on an old issue that appears implicitly fixed, but never
reopen anything closed within the 30-day window.

### Step 5 — Session report (final output)

Present a concise summary:
- Scope reviewed (period/range) and commit count.
- Findings table: location | severity | action (issue #N / PR #N /
  skipped-known).
- Counters: criticals (PRs) · normals (issues) · skipped (already known).
- Links to every issue/PR opened this run.

If there are no criticals, say so in one line ("no critical bugs in scope;
opened N normal issues") — do not pad.

Begin with Step 0 now.
~~~

---

## Operator notes (outside the prompt)

- Set the routine's schedule (daily works well); the default 24 h scope
  matches it. Run manually only for special ranges (a big merge, since the
  last deploy).
- **Consuming the output:** review and merge critical PRs (they already passed
  QA + integration + docs in the pipeline); triage `severity:high` issues into
  the pipeline soon; batch `medium/low` into a periodic maintenance PR.
- Dedup depends on the labels: don't strip `daily-review` from an open issue,
  or the finding will be re-reported. Normal close/merge is the only
  resolution signal needed.
- **Evolving the prompt:** when a new bug *category* recurs, add it to the
  Step 1 checklist (with a grep recipe if possible) and to the agent templates
  so the pipeline prevents it going forward — see
  [chapter 04](../playbook/04-bug-patterns.md), "growing your own pattern
  library".
