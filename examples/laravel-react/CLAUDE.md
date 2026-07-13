# CLAUDE.md — Acme Collect Developer Constitution

> This file is the **single source of truth** for any developer or AI agent
> working on this repository. Read it completely before writing any code.
>
> Acme Collect is an Arabic-first debt-collection platform: a Laravel 12 API
> (`backend/`) and a React 19 SPA (`frontend/`) in one monorepo.

---

## Table of Contents

1. [Project Philosophy](#1-project-philosophy)
2. [Tech Stack](#2-tech-stack)
3. [Development Commands](#3-development-commands)
4. [Architecture Overview](#4-architecture-overview)
5. [Core Data Models](#5-core-data-models)
6. [Git Workflow](#6-git-workflow-mandatory)
7. [The Agent Pipeline](#7-the-agent-pipeline-mandatory)
8. [Code Style](#8-code-style)
9. [Authentication & Authorization](#9-authentication--authorization)
10. [Documentation Governance](#10-documentation-governance)
11. [Versioning](#11-versioning)

---

## 1. Project Philosophy

Four non-negotiable principles. Every architectural decision traces back to
one of these:

| Principle | Meaning in practice |
| :--- | :--- |
| **Arabic-First** | The UI is RTL by default. All user-facing text, activity logs, notifications, and API error messages are in Arabic. English is for code and documentation only. |
| **Security-First** | Every new endpoint requires authentication middleware, an authorization policy, and input validation via a Form Request. No exceptions. |
| **Real-time by Default** | User-facing state changes (queue events, agent status, notifications) are broadcast over WebSocket (Laravel Reverb) immediately. Polling is a last resort. |
| **Docs-as-Law** | Documentation is a blocking pipeline stage, not a promise. The CHANGELOG is updated on every change; governed docs follow the doc-update map in §10. |

## 2. Tech Stack

| Layer | Technology |
| :--- | :--- |
| **Backend** (`backend/`) | Laravel 12, PHP 8.3, MySQL 8, Redis |
| **Frontend** (`frontend/`) | React 19, TypeScript (strict), Vite, React Router 7 |
| **Data fetching** | TanStack Query 5 (one custom hook per resource) |
| **UI** | Tailwind CSS 4 + shadcn/ui |
| **Real-time** | Laravel Reverb (server) + laravel-echo (client) |
| **Auth & permissions** | Laravel Sanctum + role-based access control (RBAC) |
| **Queue & jobs** | Laravel Horizon (Redis-backed) |
| **Testing** | PHPUnit (backend), Vitest (frontend) |

## 3. Development Commands

Backend commands run inside the dev container (`docker exec app ...`).
Frontend commands run on the host with `npm --prefix frontend`.

```bash
# ─── Backend ───────────────────────────────────────────────────────────────
docker exec app php artisan <command>              # any artisan command
docker exec app composer <command>                 # composer

docker exec app php artisan test                   # full backend suite
docker exec app php artisan test --filter=SomeTestName   # targeted tests
docker exec app composer exec pint                 # PSR-12 auto-fix

docker exec app php artisan optimize:clear         # clear all framework caches
docker exec app php artisan horizon:status         # queue worker status
docker exec app php artisan horizon:terminate      # restart workers (REQUIRED
                                                   # after changing job/listener code)

docker exec app php artisan db:seed                # environment-aware seeding

# ─── Frontend ──────────────────────────────────────────────────────────────
npm --prefix frontend run dev                      # dev server with HMR
npm --prefix frontend run build                    # production build
npm --prefix frontend run test                     # Vitest
npm --prefix frontend run lint                     # ESLint + type-check
```

**Database safety rail:** the only database anyone — human or agent — may
reset, wipe, or `migrate:fresh` is **`acme_test`**. Never any other.

## 4. Architecture Overview

### Self-contained domain modules

Large domains live as modules under `backend/app/`, each with its own routes,
controllers, jobs, events, and services, registered via a service provider —
the module is deletable as a unit:

```text
backend/app/Collections/          # collection queue, campaigns, promises to pay
├── Controllers/
├── Events/                       # broadcast events (Reverb)
├── Jobs/                         # Horizon-processed background work
├── Services/
└── routes.php                    # registered by CollectionsServiceProvider

backend/app/Outreach/            # omnichannel outbound/inbound messaging
├── Controllers/
├── Jobs/
├── Services/
└── routes.php                    # registered by OutreachServiceProvider
```

Cross-cutting code (models, policies, shared services) stays in the standard
Laravel locations (`backend/app/Models/`, `backend/app/Policies/`,
`backend/app/Services/`).

### Provider abstraction for external services

Every external vendor sits behind an interface + per-vendor drivers + a
manager/factory. Example — the messaging layer:

```text
MessagingManager (factory)
├── MessagingProviderInterface (contract)
├── TwilioDriver          # production SMS/WhatsApp
├── LocalGatewayDriver    # regional SMS gateway
└── LogDriver             # no-op for dev/testing
```

Driver selection priority: campaign override → system default
(`Setting::get('messaging.default_provider')`). New vendors are new drivers —
never `if ($vendor === ...)` branches in business logic.

### WebSocket channels (Laravel Reverb)

Defined in `backend/routes/channels.php`:

| Channel | Audience |
| :--- | :--- |
| `agent.{userId}` | Private per-agent events (assignments, call state) |
| `notifications.{userId}` | Notification center (private) |
| `supervisors` | Supervisor dashboards — live team-wide events |
| `messaging.thread.{threadId}` | Per-thread message/typing/delivery events |

### Key services

| Service | Responsibility |
| :--- | :--- |
| `QueueService` | Collection queue with priority scoring and 5-minute assignment locks |
| `PaymentAllocationService` | Payment waterfall allocation across installments (oldest due first; rounding remainder folds into the last installment) |
| `NotificationService` | Central dispatch with per-user preference filtering and batching |
| `OutreachService` | Thread resolution, outbound send, provider dispatch via `MessagingManager` |
| `ReminderService` | Level transitions for overdue debtors (`none → reminder → call → legal`) with de-escalation on payment |

### Runtime configuration

Use the `Setting` model (Redis-cached) for database-stored settings:

```php
Setting::get('collections.daily_send_limit', 150);
Setting::set('payments.rounding_step', 500);
```

Never call `env()` outside `config/` files.

## 5. Core Data Models

```text
Debtor (1) ──── (N) Contract (1) ──── (N) Installment
                     │
                     ├── (N) Payment
                     └── (N) PromiseToPay

Campaign (1) ──── (N) CampaignQueue
User (1) ──── (N) ContactAttempt
```

### Enum values reference

```php
// ContactOutcome (outreach_attempts.outcome)
'answered', 'no_answer', 'busy', 'wrong_number',
'promise_to_pay', 'refused_to_pay', 'callback_requested', 'do_not_call'

// PromiseStatus (payment_promises.status)
'open', 'kept', 'partial_kept', 'broken', 'rescheduled'

// CampaignStatus (campaigns.status)
'draft', 'active', 'paused', 'completed'
```

Enums are PHP enum classes in `backend/app/Enums/`. Code that compares
against string literals must match an enum case that actually exists — a
string with no matching case is a silent dead branch.

### Money — Iraqi Dinar

- Store all amounts as **`decimal(29,4)`** directly in IQD (never a
  minor-unit integer).
- Example: `1250000.0000` IQD.
- Backend helpers: `format_currency($amount)`, `currency_to_float($string)`
  in `backend/app/helpers.php`.
- Frontend helper: `formatIQD(amount)` in `frontend/src/lib/currency.ts`.
- Division/allocation must account for the rounding remainder explicitly
  (see `PaymentAllocationService`) — never let fractions of a dinar vanish.

### Important composite indexes

```sql
-- Debtor filtering for campaign targeting
CREATE INDEX idx_debtors_segment_priority ON debtors(segment, contact_priority, risk_score);
-- Queue locking
CREATE INDEX idx_queue_lock ON campaign_queue(priority_score, locked_until);
-- Contract bucketing
CREATE INDEX idx_contracts_debtor_status ON contracts(debtor_id, status, delinquency_bucket);
```

## 6. Git Workflow (MANDATORY)

**Step 1 — always check your current branch:** `git branch --show-current`.

**Step 2 — branch rules:**

| Branch | Rule |
| :--- | :--- |
| `main` | **NEVER commit directly.** Production only. |
| `develop` | **NEVER commit directly.** Integration branch; PRs target this. |
| `feature/*` / `hotfix/*` | Your working branch. Always branch from `develop`. |

**Step 3 — commit format (Conventional Commits):**

| Type | When |
| :--- | :--- |
| `feat:` | New feature |
| `fix:` | Bug fix |
| `docs:` | Documentation only |
| `refactor:` | Code refactoring |
| `chore:` | Maintenance, dependencies, wiring |
| `security:` | Security fix |

**Step 4 — commit & push after every unit of work; never stash-and-forget.**

- When a logical unit of work is complete, stage **all** of its changes,
  commit, and push. Work left uncommitted or parked in a stash is work lost.
- **Include `.claude/agent-memory/` updates in the commit** — agent memory is
  part of the deliverable, not noise. (History: forgotten memory files were
  once stranded in stashes for weeks because no step committed them.)
- `git stash` is for short-lived context switches only; pop and commit before
  the task ends.
- **Never force-push** unless explicitly requested by the user.
- A task is not done until its changes are committed and pushed.

## 7. The Agent Pipeline (MANDATORY)

**Analyze-first development:** before writing a single line of code, the work
goes through the agent pipeline — plan, human approval, parallel build,
independent review, integration, documentation, release. Never bypass it:
writing code directly means skipping QA, integration checks, and
documentation, all of which are blocking stages.

| Task type | Lane |
| :--- | :--- |
| New feature, module, page, refactor, non-trivial bug fix | Full pipeline (Stage 1 — Plan … Stage 6 — Release) |
| Small fix meeting **all** hotfix criteria | Hotfix lane (Plan → Build+Review → Document+Release) |
| UI/UX design or redesign | Full pipeline + Stage 1.5 — Design |

- The orchestrator (`team-lead`) runs as the **main session** — start it with
  `claude --agent team-lead` or select it as your session agent.
  ⚠️ **Never invoke `team-lead` via the Task/Agent tool**: sub-agents cannot
  spawn sub-agents, so a nested orchestrator silently loses its entire team.
- Two human gates: **Gate 1** approves the plan (end of Stage 1 — Plan);
  **Gate 2** approves the version bump (start of Stage 5 — Version &
  Document).
- Only the orchestrator runs `git`/`gh`. Builders and reviewers write or
  read files and report in the canonical verdict formats — nothing else.
- Agent definitions live in `.claude/agents/`; per-agent memory in
  `.claude/agent-memory/<agent>/MEMORY.md`.

## 8. Code Style

### PHP (PSR-12 + strict types)

```php
<?php
declare(strict_types=1);

class PaymentAllocationService
{
    public function __construct(
        private readonly PaymentRepository $payments,
    ) {}

    public function allocate(Payment $payment): AllocationResult
    {
        // Always use parameter and return type hints.
    }
}
```

- Controllers stay thin: HTTP concerns only; business logic in services.
- Validation lives in Form Requests (`backend/app/Http/Requests/`) — never
  inline `$request->validate()` in controllers.
- Eloquent with eager loading (`->with()`) to prevent N+1 queries; no raw SQL
  unless necessary and documented.
- Every catch block that reaches an API client goes through the shared error
  trait with an **Arabic** message; never echo `$e->getMessage()` to clients.

### React (function components + hooks + TypeScript)

```tsx
// frontend/src/hooks/usePayments.ts — ONE hook per resource, always.
export function usePayments(contractId: number) {
  return useQuery({
    queryKey: ['payments', contractId],
    queryFn: () => api.get<PaymentsResponse>(`/contracts/${contractId}/payments`),
  });
}
```

- **Function components only**, typed props, no `any`. Server state goes
  through TanStack Query custom hooks (`usePayments`, `useDebtors`) — one
  hook per resource, in `frontend/src/hooks/`. Components never call the
  HTTP client directly.
- Response types are written against the **actual** controller output — key
  names verified, not guessed. The single most common cross-boundary bug is
  the frontend reading a key the backend never emits.
- **Echo cleanup rule:** every `laravel-echo` subscription made in a
  `useEffect` must be released in the effect's cleanup function:

  ```tsx
  useEffect(() => {
    const channel = echo.private(`messaging.thread.${threadId}`);
    channel.listen('.message.received', onMessage);
    return () => echo.leave(`messaging.thread.${threadId}`);
  }, [threadId]);
  ```

- UI primitives come from shadcn/ui; do not hand-roll modals, dropdowns, or
  tables that the kit already provides.

### Tailwind CSS class order

Layout (`flex`, `grid`) → Positioning → Box model (`w`, `h`, `p`, `m`) →
Typography → Visual (`bg`, `border`, `shadow`) → Interactive (`hover:`,
`focus:`) → Responsive (`sm:`, `md:`, `lg:`) → Directional (`rtl:`, `ltr:`).

### RTL conventions (Arabic-First, concretely)

- `dir="rtl"` is set at the app root; all layout must be correct in RTL first.
- Use **logical** utilities — `ms-*`/`me-*`, `ps-*`/`pe-*`, `text-start`/
  `text-end` — never `ml-*`/`mr-*` or `text-left`/`text-right`.
- `rtl:`/`ltr:` modifiers only for genuinely directional cases (e.g., a
  chevron icon: `rtl:rotate-180`).
- Numbers, currency, and dates render via the shared Arabic-locale formatters.

## 9. Authentication & Authorization

### Authentication (Laravel Sanctum)

- SPA token auth; token expiration 7 days (configurable via env).
- Login rate-limited: `throttle:5,1`.
- Password policy: min 8 chars, mixed case, at least one number.
- Webhook endpoints (no Sanctum) verify an HMAC signature over the **raw**
  request body with `hash_equals`, and are rate-limited.

### Authorization (RBAC)

| Role | Access level |
| :--- | :--- |
| `super_admin` | Full system access (`Gate::before` bypass) |
| `admin` | Administrative functions except critical system operations |
| `supervisor` | Team dashboards, campaign management, reports |
| `collector` | Standard collection agent — own queue and own data only |
| `accountant` | Read-only financial data access |

- Policies live in `backend/app/Policies/`. **Every new endpoint must have a
  corresponding Policy or Gate check** — the security reviewer blocks on this.
- Privilege ceiling: an actor can never grant roles or permissions exceeding
  their own.
- PII files (debtor attachments, receipts) are stored on the private `local`
  disk and served only through permission-checked endpoints — never via a
  public `/storage/` URL.

## 10. Documentation Governance

Canonical docs live in `docs/`, numbered and feature-based (full tree and
rationale: [`docs-structure.md`](docs-structure.md)):

```text
docs/
├── 1_System_Blueprints/     # architecture and domain blueprints
├── 2_User_Guides/           # Arabic-first user guides, per module
├── 3_Developer_Docs/        # API reference, data dictionary, settings reference
└── _archive/                # historical reports & audits — read-only
```

When implementing changes, update the corresponding doc file:

| Change area | Doc file to update |
| :--- | :--- |
| Debtors / contracts | `docs/2_User_Guides/2.1_Debtors_Guide.md` |
| Payments | `docs/2_User_Guides/2.2_Payments_Guide.md` |
| Collections queue / campaigns | `docs/2_User_Guides/2.3_Collections_Manual.md` |
| Messaging | `docs/2_User_Guides/2.4_Messaging_Guide.md` |
| Escalation | `docs/2_User_Guides/2.5_Escalation_Guide.md` |
| Settings / configuration | `docs/3_Developer_Docs/3.3_Settings_Reference.md` |
| API changes | `docs/3_Developer_Docs/3.1_API_Reference.md` |
| Database schema | `docs/3_Developer_Docs/3.2_Data_Dictionary.md` |
| **Any change** | `CHANGELOG.md` (**always required**) |

Rules: **no report files in the repo root**; `_archive/` is read-only
history; test procedure runbooks live in `backend/tests/` as
symptom → cause → solution documents.

## 11. Versioning

```text
vMAJOR.MINOR.PATCH
│      │     └─ PATCH: bug fixes, small improvements
│      └────── MINOR: new features, new modules
└───────────── MAJOR: breaking changes, migrations with data impact
```

The current version lives in the `VERSION` file; the CHANGELOG follows
Keep a Changelog, Arabic entries with an English summary line.

**On release:**

1. Update `VERSION`.
2. Update `CHANGELOG.md` (heading must equal the `VERSION` value).
3. Merge the PR into `develop`.
4. Tag **after** merge: `git tag -a vX.Y.Z -m "Release vX.Y.Z"` — never tag
   feature branches.
