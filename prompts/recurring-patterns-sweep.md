# Recurring-Patterns Sweep — Full Codebase

A standing routine ([playbook chapter 07](../playbook/07-standing-routines.md))
that hunts the [5 recurring bug patterns](../playbook/04-bug-patterns.md) and
re-runs the [phantom checks A–H](../playbook/05-phantom-checks.md) across the
**entire codebase** — not one diff. The pipeline checks these per-change;
this sweep catches every instance that merged *before* a pattern was known.

**How to run it:** paste the `## PROMPT` block into a fresh session (or spawn
it to a read-only QA-type agent). Detection only — the sweep never fixes.
Expect 30–45 minutes for a full pass.

## Cadence

| Scenario | Run |
| :--- | :--- |
| After each large merge (50+ files) | Immediately after merging |
| Before a major release (`X.Y.0`) | Mandatory |
| Routine maintenance | Monthly |
| A user-facing bug matching one of the patterns appears | Immediately |

---

## PROMPT

~~~text
## Task: full-codebase sweep for the 5 recurring bug patterns — {{PROJECT_NAME}}

**Branch:** {{MAIN_BRANCH}} (or the currently active branch)
**Time box:** 30–45 minutes

### Rules
- **Do NOT fix anything** — detect and report only. Never run git write
  operations. Never reset any database except {{TEST_DB_NAME}}.
- **Do not run the full test suite** — this sweep is grep + code reading.
- **Stay inside the 5 patterns + checks A–H** — no open-ended exploration.
- **Exclusions:** vendor/dependency directories (`vendor/`, `node_modules/`),
  build artifacts (`public/build/`, `dist/`), caches/storage, and any legacy
  directories your project quarantines. Sweep tracked files only
  (`git ls-files`).
- **Read the relevant agents' memory first**
  (`.claude/agent-memory/qa-engineer/MEMORY.md`,
  `.claude/agent-memory/integration-agent/MEMORY.md`) — Approved Exceptions
  and Disproven Findings there are known false positives. Do not re-report
  them.
- **When in doubt, read the file** — a grep hit is a candidate, not a finding.
- Report ≤ 3000 words, in the required format below.

### PATTERN-1 — Frontend ↔ API contract mismatch
For each frontend component/hook that calls the API:
1. Grep {{FRONTEND_DIR}}/src/ for HTTP calls (your client or data-access
   modules).
2. For each URL, find the route registration, then the controller/handler.
3. Read the literal response keys the controller emits.
4. Read the keys the frontend consumes (property access, destructuring,
   typed interfaces, mapped arrays).
5. Report every key read but never emitted, and every shape mismatch
   (wrapper-vs-bare, object-vs-array, snake_case-vs-nested).
Prioritize: dashboards, chart-feeding components, forms that POST.
This is the slowest pattern — run it last.

### PATTERN-2 — Enum / config-key string mismatch
```bash
# A. Membership checks against string lists — verify each string is a real enum case
grep -rn "in_array(" {{BACKEND_DIR}}/ | grep -E "'[a-z_]+'.*'[a-z_]+'" | head -30

# B. Settings reads without a matching seeded key
grep -rohE "Setting::get\('[a-z_.]+'" {{BACKEND_DIR}}/ | sort -u > /tmp/reads.txt
grep -rohE "Setting::set\('[a-z_.]+'" database/seeders/ {{BACKEND_DIR}}/ | sort -u > /tmp/writes.txt
diff /tmp/reads.txt /tmp/writes.txt   # read-but-never-seeded = phantom key

# C. Relation column lists vs actual schema
grep -rn "with(\['" {{BACKEND_DIR}}/ | grep ":id," | head -30
# For each: verify every column against the table's migration/schema.
```
Adjust the idioms to your stack; the invariant is: every string literal that
references a source of truth gets checked against that source of truth.

### PATTERN-3 — Phantom features (checks A–D, run codebase-wide)
```bash
# A. Notifications/events/jobs with zero dispatch sites
for cls in $(find {{BACKEND_DIR}}/app/Notifications {{BACKEND_DIR}}/app/Events -name "*.php" -exec basename {} .php \; 2>/dev/null); do
    n=$(grep -rn "new $cls\|$cls::dispatch\|notify(.*$cls\|broadcast(new $cls" {{BACKEND_DIR}}/ | wc -l)
    [ "$n" = "0" ] && echo "[PATTERN-3][check A] $cls — ZERO dispatch sites"
done

# B. Broadcast events with no frontend listener (and the inverse)
#    For each broadcast event name, grep {{FRONTEND_DIR}}/src/ for a subscriber;
#    for each frontend subscription, grep the backend for the broadcaster.

# C. Settings toggles never read
grep -rohE "Setting::set\('[a-z_.]+\.enabled'" database/seeders/ | sed -E "s/.*'([a-z_.]+)'.*/\1/" | sort -u | \
    while read key; do
        n=$(grep -rn "Setting::get('$key'" {{BACKEND_DIR}}/ | wc -l)
        [ "$n" = "0" ] && echo "[PATTERN-3][check C] $key — toggle never checked"
    done

# D. Permissions seeded but never enforced
#    Extract permission names from the roles seeder, then:
#    grep -rn "permission:<perm>\|can('<perm>'" {{BACKEND_DIR}}/ routes/ {{FRONTEND_DIR}}/src/
#    Zero hits = decorative permission.
```

### PATTERN-4 — Job/queue wiring gaps (checks E–F)
```bash
# E. Named queues not in the worker configuration
grep -rohE "onQueue\('[a-z_]+'\)" {{BACKEND_DIR}}/ | sort -u | sed -E "s/.*'([a-z_]+)'.*/\1/" | \
    while read q; do
        n=$(grep -rc "'$q'" config/ | awk -F: '{s+=$2} END {print s+0}')
        [ "$n" = "0" ] && echo "[PATTERN-4][check E] queue '$q' not in worker config"
    done

# F. Duplicate scheduled tasks at the same time
grep -B1 -E "dailyAt\('[0-9:]+'\)" routes/console.php | head -50
# Manually verify no two tasks at the same time produce the same effect.
# Also flag schedule entries mutating shared state without overlap guards.
```

### PATTERN-5 — Documentation/code drift
```bash
# A. Class names mentioned in governed docs vs actual files
grep -rohE "[A-Z][a-zA-Z]+(Job|Notification|Event|Service|Controller|Listener)" \
    CLAUDE.md CHANGELOG.md README.md docs/ 2>/dev/null | sort -u | \
    while read cls; do
        found=$(find {{BACKEND_DIR}}/ -name "$cls.*" 2>/dev/null | head -1)
        [ -z "$found" ] && echo "[PATTERN-5] $cls documented but not found in code"
    done

# B. Setting keys in docs vs seeders — grep each documented key in database/seeders/
# C. Channel/route/command names in docs vs their registration files
# D. Any "last updated" stamps vs recent commit dates
```
Check ALL governed docs in the same pass — drift hides in the file you skip.

### Required report format

```markdown
# Recurring Pattern Sweep — {{PROJECT_NAME}}

**Branch:** <branch>   **Date:** YYYY-MM-DD
**Scope:** full repository (exclusions listed above)

## Summary
| Pattern | Findings |
| PATTERN-1 (frontend↔API) | N |
| PATTERN-2 (enum/config strings) | N |
| PATTERN-3 (phantom features, A–D) | N |
| PATTERN-4 (queue/schedule wiring, E–F) | N |
| PATTERN-5 (doc drift) | N |
| **Total** | **N** |

## Findings
### [PATTERN-1] ...
#### 1.1 <Component>:<line> — reads `x.y`, API emits `x_y`
- **Frontend:** path:line — what it consumes
- **API:** controller::method line — what it emits
- **Impact:** <user-visible effect>
- **Severity:** HIGH | MEDIUM | LOW
- **Suggested fix:** <alias key / correct literal / wire or remove>
(repeat per finding; every finding carries its [PATTERN-N] label)

## Confidence notes
- <which checks were exhaustive, which were sampled, known blind spots>

## Suggested priority
- Fix immediately: <HIGH findings>
- Fix this sprint: <MEDIUM>
- Fix when convenient: <LOW>
```

Begin now. Run PATTERN-3 first (fastest — pure grep), PATTERN-1 last.
~~~

---

## Operator notes (outside the prompt)

**Consuming the report:** HIGH findings → individual fix PRs through the
pipeline now; MEDIUM → batch into a periodic maintenance PR; LOW → track and
fold into the next round.

**False positives:** every candidate the sweep raises and you (or a deeper
read) disprove must be recorded in the relevant agent's memory under
**Disproven Findings** ([chapter 06](../playbook/06-memory-system.md)) — the
sweep reads memory at start, so a recorded disproof permanently silences that
false positive.

**New patterns:** when the sweep (or any review) surfaces a *new* recurring
pattern, it gets added in four places — [chapter 04](../playbook/04-bug-patterns.md)
as `[PATTERN-6]` with a detection recipe, the QA and integration agent
templates, the relevant agent memory, and this prompt — so both the pipeline
and future sweeps hunt it. That growth loop is the whole point:
patterns are earned from real blockers, then institutionalized.
