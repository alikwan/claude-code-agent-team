# PR Body Templates

Used by the orchestrator at **Stage 6 — Release** (`gh pr create --base
{{MAIN_BRANCH}}`). The verdicts table is the PR's audit trail: a reviewer can
see at a glance which stages ran, which were N/A, and how many cycles QA took.

> The source project wrote its PR bodies in its primary user-facing language
> (Arabic); a generic Arabic variant is collapsed at the bottom. Whatever the
> language, the canonical verdict strings stay in English — they are machine
> contracts, not prose.

## Feature PR

```markdown
## Summary
<!-- What changed and why, 2–5 lines. Link the issue if one exists. -->

## Pipeline verdicts
| Stage | Agent | Verdict |
| :--- | :--- | :--- |
| Stage 3 — Review | qa-engineer | QA APPROVED ✅ (cycles: N) |
| Stage 3 — Review | security-reviewer | SECURITY REVIEW: APPROVED ✅ / N/A |
| Stage 3 — Review | migration-reviewer | MIGRATION REVIEW: APPROVED ✅ / N/A |
| Stage 4 — Integrate | integration-agent | INTEGRATION COMPLETE ✅ (+ delta re-check if files modified) |
| Stage 5 — Version & Document | documentation-agent | DOCUMENTATION AUDIT: COMPLETE ✅ |

## Files changed
<!-- Grouped list: created / modified / deleted. -->

## Test results
<!-- N passed, 0 failed (targeted: <scope>) — plus QA's empirical check. -->

## Breaking changes / migration notes
<!-- None, or: schema changes, deploy steps (cache clear, worker restart),
     post-deploy commands. Be explicit — this section is read at deploy time. -->

Version: vX.Y.Z (approved at Gate 2)
```

## Hotfix PR

```markdown
## Summary
<!-- The bug, the root cause, the fix. 2–4 lines. -->

## Hotfix eligibility
<!-- Confirm: ≤2 files, no new functions/classes, no migrations, no new
     permissions/routes/middleware, no UI layout changes, no new deps. -->

## Verdicts
| Stage | Agent | Verdict |
| :--- | :--- | :--- |
| Build + Review | qa-engineer | QA APPROVED ✅ |
| Document + Release | documentation-agent | DOCUMENTATION AUDIT: COMPLETE ✅ |

## Test results
<!-- Targeted tests around the fix. -->
```

<details>
<summary>النسخة العربية — قالب ميزة (generic Arabic feature variant)</summary>

```markdown
## الملخص
<!-- ما الذي تغيّر ولماذا، في سطرين إلى خمسة أسطر. -->

## نتائج مراحل خط الأنابيب
| المرحلة | الوكيل | النتيجة |
| :--- | :--- | :--- |
| المرحلة 3 — المراجعة | qa-engineer | QA APPROVED ✅ (عدد الدورات: N) |
| المرحلة 3 — المراجعة | security-reviewer | SECURITY REVIEW: APPROVED ✅ / لا ينطبق |
| المرحلة 3 — المراجعة | migration-reviewer | MIGRATION REVIEW: APPROVED ✅ / لا ينطبق |
| المرحلة 4 — الدمج | integration-agent | INTEGRATION COMPLETE ✅ |
| المرحلة 5 — الإصدار والتوثيق | documentation-agent | DOCUMENTATION AUDIT: COMPLETE ✅ |

## الملفات المتغيّرة
<!-- قائمة مجمّعة: مُنشأة / معدّلة / محذوفة. -->

## نتائج الاختبارات
<!-- عدد الاختبارات الناجحة والفاشلة، ونطاق الاختبارات المستهدفة. -->

## تغييرات جذرية / ملاحظات النشر
<!-- لا يوجد، أو: تغييرات المخطط، خطوات النشر، أوامر ما بعد النشر. -->

الإصدار: vX.Y.Z
```

</details>
