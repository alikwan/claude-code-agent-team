[**← العودة إلى الصفحة الرئيسية**](../../README.ar.md) &nbsp;·&nbsp; `docs/ar/`

[🇬🇧 Read this guide in English — getting-started.md](../getting-started.md)

<!-- Translated from docs/getting-started.md @ v0.1.0 (2026-07-13). English is canonical — لا تُحرّر هذا الملف دون مزامنة الأصل الإنجليزي -->

<div dir="rtl">

# دليل البدء — Getting Started

من الصفر إلى أول تشغيل لخط الأنابيب. المتطلبات المسبقة: [Claude Code](https://docs.anthropic.com/en/docs/claude-code) مثبّت، و[`gh` CLI](https://cli.github.com/) مُوثَّق الدخول، ومستودع git تعمل عليه فعلًا.

## البدء في 5 دقائق

**الكتلة 1 — انسخ القوالب إلى مشروعك:**

```bash
cd your-project
npx degit alikwan/claude-code-agent-team/agents .claude/agents --force
rm .claude/agents/README.md      # docs, not an agent definition
rm -r .claude/agents/optional    # keep only if you want the optional test-engineer
npx degit alikwan/claude-code-agent-team/templates .claude/pipeline-templates --force
mkdir -p .claude/agent-memory
```

**الكتلة 2 — ولّد دستورك.** افتح Claude Code في جذر المستودع والصق كامل محتوى `.claude/pipeline-templates/generate-your-claude-md.md`. سيستكشف الوكيل مستودعك بوضع القراءة فقط، ويسألك 6 أسئلة كحد أقصى (بيئة النشر، استراتيجية الفروع، اسم قاعدة بيانات الاختبار، لغة المستخدم النهائي…)، ثم يكتب `CLAUDE.md` ويتحقق ذاتيًا من أن كل أمر وثّقه يعمل فعلًا.

**الكتلة 3 — املأ الـ placeholders وانطلق:**

```bash
# fill the table in docs/placeholders.md with your values, find-replace, then verify —
# must return nothing (${{ ... }} in the CI templates is GitHub Actions syntax, not a placeholder):
grep -rnE '\{\{[A-Z][A-Z0-9_]*\}\}' .claude/ CLAUDE.md

claude --agent team-lead   # or pick team-lead from the agent selector
```

> ⚠️ **الأمر الوحيد الذي يجب ألا تفعله:** لا تستدعِ `team-lead` أبدًا عبر أداة Task/Agent من جلسة أخرى. الوكيل الفرعي (Sub-agent) لا يستطيع توليد وكلاء فرعيين — وسيتجمد خط الأنابيب عند المرحلة 2. المنسّق **هو** جلستك الرئيسية. (قصة الفشل الكاملة: [playbook/08-field-notes.md](../../playbook/08-field-notes.md) (EN)، الملاحظتان 1–2.)

## تجهيز هياكل الذاكرة (Memory Scaffolds)

مجلد واحد لكل وكيل، مُهيّأ من القالب:

```bash
for a in team-lead backend-dev frontend-dev qa-engineer integration-agent \
         documentation-agent security-reviewer migration-reviewer ui-ux-designer; do
  mkdir -p .claude/agent-memory/$a
  cp .claude/pipeline-templates/MEMORY.md.template .claude/agent-memory/$a/MEMORY.md
done
# kept the optional test-engineer? add it to the list above before running.
```

ثبّت هذه الهياكل بـ commit. الذاكرة خاضعة لإدارة الإصدارات وتسافر مع الكود — آلية عملها: [playbook/06-memory-system.md](../../playbook/06-memory-system.md) (EN).

## اختياري: الأذونات والـ CI

- ادمج `.claude/pipeline-templates/settings.json.template` في ملف `.claude/settings.json` لديك — فهو يمنع أوامر قاعدة البيانات وgit المدمّرة على مستوى طبقة الأذونات، وهي المكان الصحيح لحواجز الحماية ([الملاحظة الميدانية 3](../../playbook/08-field-notes.md) (EN) تشرح لماذا لا يكفي المنع المكتوب نثرًا).
- انسخ قوالب الـ CI: من `templates/ci/tests.yml.template` إلى `.github/workflows/tests.yml` (هذا هو **صمّام الأمان كامل الحزمة** الذي يجعل الاكتفاء بالاختبارات المستهدَفة آمنًا — انظر الثابت في [placeholders.md](../placeholders.md) (EN))، إضافةً إلى قالبَي workflow الخاصَّين بـ Claude إذا أردت إسناد المهام عبر @-mention والمراجعة التلقائية للـ PR.

## مهمتك الأولى — ما الذي ينبغي أن تراه

أعطِ المنسّق شيئًا حقيقيًا لكن متواضعًا («أضف زر تصدير إلى جدول الفواتير»). التشغيل السليم يبدو هكذا:

1. **خطة، ثم توقّف.** الملفات التي ستُلمَس، والنهج المتّبع، وكتلة عقد API إذا كان التغيير يعبر بين الـ backend والـ frontend، وطلب صريح لموافقتك. *إذا بدأ بتحرير الملفات فورًا، فملف CLAUDE.md لديك يفتقد قسم خط الأنابيب — أعِد توليده.*

2. **بناء متوازٍ.** استدعاءان لوكيلين فرعيين في رسالة واحدة (backend + frontend)، ثم commit: `feat(invoices): add export button`.

3. **حكم QA بالصيغة القياسية الدقيقة:**

   ```
   QA APPROVED ✅
   - Tests: 4 passed, 0 failed (targeted: InvoiceExportTest)
   - Lint: clean
   - Empirical check: GET /api/invoices/export → 200, CSV headers verified
   - Conditional reviews: none required
   - Requirements: met
   ```

   ظهور `QA REJECTED ❌` بنتائج مرقّمة سليمٌ أيضًا — سترى جولة إصلاح وcommit بعنوان `fix(invoices): address QA feedback`، ثم يعيد QA التشغيل.

4. **التكامل والتوثيق وبوابة الإصدار.** جولة توصيل (`INTEGRATION COMPLETE ✅`)، ثم يقترح المنسّق رفع رقم الإصدار وينتظرك — هذه هي البوابة 2.

5. **PR.** الفرع مدفوع، والـ PR مفتوح على فرع التكامل لديك مع جدول أحكام المراحل واحدةً واحدة في جسمه. يبلّغك المنسّق بالرابط.

إذا لم يطابق تقريرُ أي مرحلة الصيغَ الموثّقة في [playbook/03](../../playbook/03-communication-protocol.md) (EN)، فالأرجح أن ملف الوكيل المقابل قد عُدّل — سلاسل الأحكام النصية عنصر حامل (load-bearing) في المنظومة.

## استكشاف الأخطاء وإصلاحها

| العَرَض | السبب ← الإصلاح |
| :--- | :--- |
| `Agent type 'team-lead' not found` | القوالب ليست في `.claude/agents/`، أو أنك لست في جذر المستودع. |
| المنسّق «يُكمل» دون commit أو push | شُغِّل في الخلفية أو كوكيل فرعي. افحص `git status` بنفسك وأكمل خطوات الإصدار في الجلسة الرئيسية — [الملاحظة الميدانية 2](../../playbook/08-field-notes.md) (EN). |
| خط الأنابيب يتجمد بعد الخطة | توليد الوكلاء الفرعيين غير متاح (جلسة متداخلة، أو أداة Task معطَّلة عبر الأذونات). شغّل `team-lead` كجلسة رئيسية. |
| المطوّرون يحرّرون الملف نفسه ويكتب أحدهم فوق الآخر | prompts الاستدعاء تفتقد قوائم الملفات الحصرية — انظر ملكية الموارد في [playbook/01](../../playbook/01-pipeline.md) (EN)، المرحلة 2. |
| QA «يمرّر» حزمة الاختبارات الكاملة لمدة 40 دقيقة | اضبط `{{FULL_SUITE_POLICY}}: targeted-only` — لكن فقط إذا كان الـ CI يشغّل الحزمة الكاملة ([الثابت](../placeholders.md) (EN)). |
| الوكلاء يعيدون مناقشة اختبارات معروفة الكسر في كل تشغيل | هيّئ [KNOWN_FAILURES.md](../../templates/KNOWN_FAILURES.md.template) (EN) وأشِر إليه من prompt استدعاء QA. |
| ظهور `{{TOKEN}}` في مخرجات وكيل | فاتك placeholder — أعِد تشغيل فحص `grep -rn '{{'`. |

## إلى أين تذهب بعد ذلك

- افهم ما شغّلته للتو: [playbook/01 — The Pipeline](../../playbook/01-pipeline.md) (EN) — وتتوفر [نظرة عامة على فصول الـ playbook بالعربية](playbook-overview.ar.md)
- قلّص الفريق لمشروع صغير (3 وكلاء تكفي): [playbook/02 — Roles](../../playbook/02-roles.md) (EN)
- جدوِل الروتينات الدورية: [playbook/07](../../playbook/07-standing-routines.md) (EN) + [prompts/](../../prompts/) (EN)
- شاهد كل ملف عام وقد جُسِّد واقعيًا: [examples/laravel-react/](../../examples/laravel-react/) (EN)

</div>

---

<div align="center">

[الرئيسية بالعربية](../../README.ar.md) · [دليل البدء](getting-started.ar.md) · [نظرة عامة على المنهجية](playbook-overview.ar.md) · [دراسة الحالة](case-study.ar.md)

</div>
