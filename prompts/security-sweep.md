# Security Sweep — Full Codebase, Four Parallel Agents

A standing routine ([playbook chapter 07](../playbook/07-standing-routines.md))
that reviews the **whole codebase** for exploitable vulnerabilities. Distinct
from the pipeline's `security-reviewer`, which is diff-scoped: this sweep
assumes nothing about what was recently touched. It fans out to **four
parallel read-only agents, one per attack surface**, then routes each verified
finding into an individual fix PR.

**How to run it:** fill in the trust-boundary primer below, then paste the
`## PROMPT` block into a session that can spawn parallel read-only sub-agents
(one per surface). Expect 20–40 minutes for the scan; fix time scales with
findings.

## Trust-boundary primer (operator fills this in)

The sweep is only as good as the map you give it. Before the first run,
complete this section and keep it current:

| Boundary | Your value |
| :--- | :--- |
| Authentication mechanism | *(e.g. session + API tokens, expiry policy, token middleware)* |
| Roles & bypass rules | *(role list; any super-role that bypasses all checks)* |
| Route entrypoints | *(main route files, incl. any self-registered module routes)* |
| Unauthenticated entrypoints | *(webhooks and their signature schemes; static-token integration routes; health endpoints)* |
| External data sources | *(third-party APIs, read-only mirrored databases, file imports — and what enforces the read-only)* |
| Crown jewels | *(the data whose leak is the worst case: PII, payment records, documents, recordings)* |

## Cadence

| Scenario | Run |
| :--- | :--- |
| Before a major release (`X.Y.0`) | Mandatory |
| After adding endpoints / webhooks / file uploads / payment logic | Immediately |
| After any auth/authorization/policy change | Immediately |
| Routine maintenance | Biweekly or monthly |

---

## PROMPT

~~~text
## Task: full-codebase security review — {{PROJECT_NAME}}

**Branch:** {{MAIN_BRANCH}}
**Goal:** verified vulnerabilities (MEDIUM/HIGH/CRITICAL) with a real
end-to-end attack path. Proof comes from reading the actual code — never from
grep alone.

### Rules
- **Exclusions:** vendor/dependency dirs, build artifacts, caches/storage,
  legacy quarantine dirs. Tracked files only (`git ls-files`).
- **Read memory first:** `.claude/agent-memory/security-reviewer/MEMORY.md` —
  do not re-report anything documented there as an approved pattern or
  accepted exception.
- **Verify before reporting:** for each candidate, trace
  route → middleware → handler → policy → sink. Drop anything you cannot
  demonstrate is exploitable.
- Read-only during the scan. Never reset any database except
  {{TEST_DB_NAME}}. No git writes during the scan phase.
- Trust-boundary primer: <paste the completed table from above>

### Fan out to 4 parallel agents — one per attack surface

#### Surface 1 — Authentication & authorization (+ IDOR)
- Routes that mutate data or expose PII without auth middleware or without a
  policy/permission check.
- **IDOR:** every endpoint binding a model from the URL — verify an ownership
  or authorization check runs against the *resolved* model. Pay special
  attention to **shared polymorphic attachment tables**: any route binding an
  attachment must verify `model_type` + `model_id` against the parent the
  caller is authorized on, or attachments become cross-model readable.
- Privilege escalation: user-management endpoints — can an actor grant roles
  or permissions exceeding their own ceiling?
- Self-scoping: reports/logs endpoints — do non-supervisors see only their
  own rows?

#### Surface 2 — Injection & file handling
- Dynamic SQL from user input: raw-query fragments, and **especially
  order-by column/direction taken from the request without an allowlist**
  (the classic hole).
- External/mirrored data sources: table names from a fixed allowlist, no
  input concatenated into queries.
- Path traversal: file download/serve paths must come from a DB column, not
  the request; `..` rejected; serve via the storage abstraction, never raw
  filesystem reads of request-supplied paths.
- Uploads: server-side MIME verification (not extension), sanitized
  filenames, private disk, size caps.
- Command execution: any exec/shell/process-spawn primitives reachable from
  user input.

#### Surface 3 — Webhooks & SSRF
- Signature verification on every webhook: HMAC over the **raw body** (not
  re-encoded JSON), constant-time comparison, atomic idempotency guard.
  If a non-production env bypass exists, report it only if the bypass list
  includes production.
- SSRF: any URL fetched server-side that originates from a webhook or
  external API — verify host pinning to the expected vendor domains, block
  private/reserved IP ranges and cloud metadata endpoints (IMDS), and defeat
  DNS-rebinding where the client supports it.
- Inbound payload handling: blind field access, type juggling, stored XSS via
  attacker-controlled profile names/filenames reaching unescaped rendering.
- Outbound HTTP timeouts (availability).

#### Surface 4 — Secrets, XSS & mass assignment
- Secret leakage through settings/config read APIs: sensitive keys masked for
  callers without the settings-management permission; hunt keys *outside* the
  mask list.
- Secrets/PII in logs: API keys, tokens, full phone numbers, government IDs
  in log or activity-log calls.
- XSS: unescaped template output or raw-HTML rendering fed by
  attacker-controllable data (names, message bodies, template content).
- Mass assignment: unguarded models, `create/update(request.all())` without
  validated input — can a request set privileged columns (role, balance,
  status, admin/exclusion flags)?
- Error disclosure: any handler returning raw exception messages to the
  client instead of the project's error-response path.

### Per-finding format
- **file:line** for the sink, the route, and the (missing) policy.
- **Attack path:** who (which role/anonymous) sends what (exact request) and
  obtains what.
- **Impact** + **Severity** (CRITICAL / HIGH / MEDIUM).
- A surface with no findings must say so explicitly and list what was checked
  and found clean.
~~~

---

## After the scan: the fix protocol

For each verified finding (MEDIUM and above), **one branch and one PR per
vulnerability** — never bundle unrelated fixes, so the critical one can merge
urgently without waiting for the rest:

1. Branch from `{{MAIN_BRANCH}}`: `fix/<short-slug>`.
2. **Minimal fix**, imitating the correct pattern that already exists in the
   codebase (e.g. the ownership-check idiom used by the already-safe
   endpoints). Do not redesign what is sound.
3. **Regression test, red/green:** demonstrate the test *fails* on the
   unfixed code and *passes* after the fix. Place it with the module's tests.
4. Run `{{LINT_COMMAND}}` on changed files and `{{TEST_COMMAND}}` scoped to
   the affected module (per `{{FULL_SUITE_POLICY}}`).
5. **Document:** CHANGELOG entry under a security heading; record the
   vulnerability class + fix in
   `.claude/agent-memory/security-reviewer/MEMORY.md`.
6. Commit, push, open the PR (base `{{MAIN_BRANCH}}`), title
   `security(<scope>): <description>`, attack path in the body. **The owner
   reviews and merges — never auto-merge.**

## Record clean sweeps too

A sweep that finds nothing still produces an artifact: a dated note ("clean
4-surface sweep, YYYY-MM-DD, scope + primer version") committed to the
security reviewer's memory. A clean sweep is **evidence, not noise** — it
dates your assurance, gives the next sweep a diff-baseline, and is exactly
what you want in hand when someone asks "when was this last reviewed?"

## Evolving the routine

When a new vulnerability *class* is found, add it in three places: the
matching attack surface in this prompt, the `security-reviewer` agent
template's checklist (so the pipeline catches it per-change), and the security
memory (class + example). Same growth loop as the
[bug-pattern library](../playbook/04-bug-patterns.md).
