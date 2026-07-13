# Documentation Governance — as practiced

How Acme Collect structures its documentation so that the pipeline's
documentation stage (Stage 5 — Version & Document) has an unambiguous target
for every change. The `documentation-agent` audits against this map; a missing
update is a `DOCUMENTATION AUDIT: BLOCKED ⚠️` verdict, not a shrug.

## The numbered tree

```text
docs/
├── 1_System_Blueprints/          # architecture — how the system is built
│   ├── 1.1_Architecture_Overview.md
│   ├── 1.2_Domain_Modules.md
│   └── 1.3_Realtime_and_Queues.md
├── 2_User_Guides/                # Arabic-first, per module — how it is used
│   ├── 2.1_Debtors_Guide.md
│   ├── 2.2_Payments_Guide.md
│   ├── 2.3_Collections_Manual.md
│   ├── 2.4_Messaging_Guide.md
│   └── 2.5_Escalation_Guide.md
├── 3_Developer_Docs/             # reference — what the code exposes
│   ├── 3.1_API_Reference.md
│   ├── 3.2_Data_Dictionary.md
│   └── 3.3_Settings_Reference.md
└── _archive/                     # read-only history — never edited, never cited as current
    ├── reports/                  # one-off analysis reports, moved here on completion
    ├── audits/                   # past security/QA audit outputs
    └── roadmaps/                 # superseded planning documents
```

Numbering is deliberate: agents (and humans) reference "update 3.1" without
ambiguity, and the doc-update map below stays stable as files are added.

## The doc-update map

When implementing a change, the corresponding doc file **must** be updated in
the same pipeline run:

| Change area | Doc file to update |
| :--- | :--- |
| Debtors / contracts | `docs/2_User_Guides/2.1_Debtors_Guide.md` |
| Payments / installment allocation | `docs/2_User_Guides/2.2_Payments_Guide.md` |
| Collections queue / campaigns | `docs/2_User_Guides/2.3_Collections_Manual.md` |
| Messaging (threads, templates, webhooks) | `docs/2_User_Guides/2.4_Messaging_Guide.md` |
| Escalation levels / policies | `docs/2_User_Guides/2.5_Escalation_Guide.md` |
| Settings / runtime configuration keys | `docs/3_Developer_Docs/3.3_Settings_Reference.md` |
| API endpoints (new/changed/retired) | `docs/3_Developer_Docs/3.1_API_Reference.md` |
| Database schema (tables, columns, indexes) | `docs/3_Developer_Docs/3.2_Data_Dictionary.md` |
| Architecture (modules, channels, providers) | `docs/1_System_Blueprints/` (relevant file) |
| **Any change whatsoever** | `CHANGELOG.md` — **always required, no exceptions** |

## The rules

1. **No report files in the repo root.** Analysis outputs, audit results, and
   one-off writeups go to `docs/_archive/reports/` (or better: into a GitHub
   issue/PR). A root directory that accumulates `FINDINGS_FINAL_v2.md` files
   is the first symptom of documentation without governance.
2. **`_archive/` is read-only history.** Nothing in it is ever updated to
   "stay current" — if a document needs to stay current, it does not belong
   in the archive. Agents may cite archived documents as historical context
   only.
3. **CHANGELOG is always required.** Every pipeline run touches it; the
   Checkpoint D commit fails review without it. The CHANGELOG heading must
   equal the `VERSION` file content — the documentation agent verifies this
   equality explicitly.
4. **Test procedure runbooks live in `backend/tests/`,** not in `docs/` —
   written as **symptom → cause → solution** documents (e.g.,
   `TEST_flaky_queue_assertions.md`: what failure looks like, why it happens,
   how to fix or safely rerun). They sit next to the tests because that is
   where the person staring at a red suite actually looks.
