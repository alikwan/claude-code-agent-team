[🇬🇧 Read this guide in English — README.md](README.md)

<!-- Translated from README.md @ v0.1.0 (2026-07-13). English is canonical — لا تُحرّر هذا الملف دون مزامنة الأصل الإنجليزي -->

<div dir="rtl">

# فريق وكلاء الذكاء الاصطناعي في Claude Code — منهجية تطوير متعددة الوكلاء مجرّبة في الإنتاج

![License: MIT](https://img.shields.io/badge/license-MIT-blue)
![Made for Claude Code](https://img.shields.io/badge/made%20for-Claude%20Code-d97757)
![Docs](https://img.shields.io/badge/docs-EN%20%7C%20AR-green)
![Production proven](https://img.shields.io/badge/production-200%2B%20releases-brightgreen)

> حوِّل Claude Code إلى فريق تطوير برمجيات متكامل: 9 وكلاء متخصصين (Sub-agents)، خط تسليم من 6 مراحل، ذاكرة للوكلاء، وبروتوكولات مراجعة — مستخلصة من نظام إنتاجي حقيقي شحن أكثر من 200 إصدار بهذه الطريقة.

![فريق وكلاء Claude Code — تسعة روبوتات تحمل خط تسليم برمجي بنقاط تثبيت: قائد برتقالي يوجّه، بنّاؤون بالأزرق يعملون بالتوازي، فاحص جودة بالأزرق المخضرّ يدقق بعدسة، ومراجعون بالبنفسجي يحملون قوائم فحص](assets/hero-banner.svg)

دليل منهجي (playbook) مجرّب في الإنتاج لتطوير البرمجيات متعدد الوكلاء داخل Claude Code: تسعة وكلاء متخصصين (Sub-agents) يعملون معًا كفريق تطوير واحد منسّق. منسّقٌ من نوع team-lead يقود وكلاء الـ backend والـ frontend والـ QA والتكامل والمراجعة الأمنية ومراجعة الـ migrations وتصميم واجهات الاستخدام والتوثيق عبر خط أنابيب من 6 مراحل، بنقاط commit تفتيشية وبروتوكولات أحكام (verdicts) صارمة. يراكم الوكلاء معرفتهم في نظام ذاكرة مستقل لكل وكيل، وتغذّي الأخطاء المتكررة مكتبةَ أنماط، وتحافظ prompts الروتينات الدورية (مراجعة الكود اليومية، مسح الأنماط المتكررة، المسح الأمني) على الجودة من الانحراف مع الزمن. يتضمن المستودع قالب «دستور» CLAUDE.md، وmeta-prompt يولّد دستورًا خاصًا بمشروعك، ومثالًا تطبيقيًا كاملًا بـ Laravel + React. وكل شيء موثّق بالإنجليزية والعربية معًا.

**أكثر من 200 إصدار إنتاجي · مطوّر واحد · 9 وكلاء · منصة حيّة «عربية أولًا» (Arabic-first)**

## لمن هذا المستودع (ولمن ليس له)

**هذا لك إذا كنت:** تشغّل Claude Code على قاعدة كود حقيقية طويلة العمر — وحدك أو ضمن فريق صغير — وتريد أن تصلك التغييرات التي يبنيها الذكاء الاصطناعي مخطَّطةً ومُراجَعةً ومتكاملةً وموثّقةً ومثبَّتةً بـ commits منضبطة، بدل أن تُرَشّ عشوائيًا في شجرة العمل لديك.

**ليس لك إذا كنت:** تريد سربَ وكلاء مستقلًا بالكامل (انظر CrewAI / AutoGen — رفٌّ مختلف)، أو تستخدم سلسلة أدوات غير Claude Code (*المنهجية* تنتقل، لكن صيغة ملفات الوكلاء لن تنتقل)، أو كان مشروعك تجربةَ عطلة نهاية أسبوع تُرمى لاحقًا، حيث يكلّفك خط أنابيب من 6 مراحل أكثر مما يحميك. راجع [متى *لا* تستخدم خط الأنابيب الكامل](docs/faq.md#when-should-i-not-use-the-full-pipeline) (EN).

## كيف تُجهّز فريق وكلاء متعدد في Claude Code

ثلاث خطوات انسخ-والصق (الشرح الكامل خطوة بخطوة: [دليل البدء](docs/ar/getting-started.ar.md)):

**1 — انسخ قوالب الوكلاء إلى مشروعك:**

```bash
npx degit alikwan/claude-code-agent-team/agents .claude/agents --force
rm .claude/agents/README.md      # docs, not an agent definition
rm -r .claude/agents/optional    # keep only if you want the optional test-engineer
npx degit alikwan/claude-code-agent-team/templates .claude/pipeline-templates --force
```

**2 — ولّد دستور مشروعك.** افتح Claude Code في جذر مستودعك والصق الـ meta-prompt من [`templates/generate-your-claude-md.md`](templates/generate-your-claude-md.md) (EN) — سيستكشف مستودعك، ويطرح عليك 6 أسئلة كحد أقصى، ثم يكتب ملف `CLAUDE.md` كاملًا.

**3 — املأ الـ placeholders وشغّل مهمتك الأولى:**

```bash
# replace the {{PLACEHOLDER}} tokens per docs/placeholders.md, then verify —
# must return nothing (${{ ... }} in CI templates is GitHub Actions syntax, not a placeholder):
grep -rnE '\{\{[A-Z][A-Z0-9_]*\}\}' .claude/ CLAUDE.md

claude --agent team-lead   # the orchestrator IS your main session
                           # (or pick team-lead from the agent selector)
```

أعطه مهمة حقيقية. سيقرأ دستورك، ويعرض عليك خطة، وينتظر موافقتك، ثم يقود الفريق.

## كيف يعمل خط الأنابيب ذو المراحل الست

![مخطط خط أنابيب فريق وكلاء الذكاء الاصطناعي في Claude Code المكوّن من 6 مراحل: منسّق team-lead يقود مراحل التخطيط والبناء المتوازي والمراجعة والتكامل والإصدار والتوثيق ثم التسليم، مع بوابتي موافقة بشرية وأربع نقاط commit تفتيشية](assets/pipeline-diagram.svg)

بوابتان بشريتان (توافق أنت على **الخطة**، ثم على **الإصدار**)، وأربع نقاط commit تفتيشية (كل مرحلة تترك خلفها تاريخًا قابلًا للفحص)، وأحكامٌ على هيئة سلاسل نصية دقيقة تُفسَّر آليًا (`QA APPROVED ✅` — وليس أبدًا «يبدو جيدًا في مجمله»)، وقاعدة حديدية واحدة: **لا يلمس git أحدٌ سوى المنسّق.** الآليات الكاملة: [playbook/01-pipeline.md](playbook/01-pipeline.md) (EN) — وتتوفر [نظرة عامة على فصول الـ playbook بالعربية](docs/ar/playbook-overview.ar.md).

## ماذا ستحصل عليه

| | الأصل | المكان | ما هو |
| :---: | :--- | :--- | :--- |
| <img src="assets/icons/playbook.svg" width="44" alt="أيقونة الدليل المنهجي"> | **الدليل المنهجي (playbook)** | [`playbook/`](playbook/) (EN) — [نظرة عامة بالعربية](docs/ar/playbook-overview.ar.md) | 9 فصول: خط الأنابيب، الأدوار، البروتوكولات، أنماط الأخطاء، فحوص «الميزات الشبح»، الذاكرة، الروتينات الدورية، الملاحظات الميدانية |
| <img src="assets/icons/agents.svg" width="44" alt="أيقونة الوكلاء"> | **قوالب 9 وكلاء (+1 اختياري)** | [`agents/`](agents/) (EN) | ملفات Sub-agents جاهزة لـ Claude Code، مُعامَلة بالكامل (parameterized) — انسخ، استبدل القيم، شغّل |
| <img src="assets/icons/prompts.svg" width="44" alt="أيقونة البرومبتات"> | **روتينات مراجعة دورية** | [`prompts/`](prompts/) (EN) | مراجعة كود يومية، مسح للأنماط المتكررة، مسح أمني رباعي الأسطح — جدوِلها وانسَها |
| <img src="assets/icons/templates.svg" width="44" alt="أيقونة القوالب"> | **قوالب** | [`templates/`](templates/) (EN) | دستور CLAUDE.md + الـ meta-prompt المولِّد له، ذاكرة الوكلاء، عقد الـ API، أجسام الـ PR، الأذونات، CI |
| <img src="assets/icons/examples.svg" width="44" alt="أيقونة المثال التطبيقي"> | **مثال تطبيقي كامل** | [`examples/laravel-react/`](examples/laravel-react/) (EN) | كل ملف عام مُجسَّدًا لمستودع monorepo يجمع Laravel 12 API + React SPA — بما في ذلك prompts المراجعة الإنتاجية العربية الأصلية |

## بمَ يختلف هذا عن مستودعات الوكلاء الأخرى؟

- **ليس مجموعة prompts.** كل ملف متشابك مع غيره: الوكلاء يستشهدون بخط الأنابيب، وخط الأنابيب يستشهد ببروتوكول الأحكام، والـ CI يفرض المفردات نفسها. إنها منهجية متكاملة، لا مقتطفات متناثرة.
- **ليس إطار عمل مستقلًا ذاتيًا.** هذا نظام يُبقي الإنسان في الحلقة عن عمد — بوابتا موافقة، وانضباط git، وإيصالات في كل مرحلة. الهدف تغييرات جديرة بالثقة على قاعدة كود تهمّك، لا أقصى قدر من الاستقلالية.
- **مُكتسَب بالتجربة، لا مُخترَع.** كل قاعدة هنا تعود إلى شيء انكسر فعلًا في الإنتاج. جاءت [مكتبة أنماط الأخطاء](playbook/04-bug-patterns.md) (EN) من نحو 30 عائق مراجعة (blockers) حقيقيًا؛ وتوثّق [الملاحظات الميدانية](playbook/08-field-notes.md) (EN) إخفاقاتنا، بما فيها تلك التي يصلحها الإصدار v1.1 من هذا الخط.

## أدوار الوكلاء التسعة (Sub-agents)

| الوكيل | مهمته | القيد الأساسي |
| :--- | :--- | :--- |
| `team-lead` | يخطّط، ينسّق، يملك **كل** عمليات git، ويفتح الـ PR | لا يستطيع كتابة الملفات — عليه التفويض |
| `backend-dev` / `frontend-dev` | يبنيان بالتوازي وفق عقد API مشترك | لا يشغّلان git أبدًا |
| `qa-engineer` | يراجع الـ diff، يشغّل الاختبارات والـ lint، ويجرّب المسار فعليًا | قراءة فقط — القاضي لا يُمسك قلم التصحيح |
| `security-reviewer` | قائمة فحص أمنية من 10 فئات (شرطي) | قراءة فقط، نموذج اقتصادي |
| `migration-reviewer` | سلامة migrations قاعدة البيانات (شرطي) | قراءة فقط، نموذج اقتصادي |
| `integration-agent` | يكتشف «الميزات الشبح» — المبنية لكن غير الموصولة — ويصلحها | يوصّل ولا يُصدر أحكامًا أبدًا |
| `documentation-agent` | تدقيق توثيق مُعطِّل (blocking) — والـ CHANGELOG دائمًا | لا PR قبل COMPLETE |
| `ui-ux-designer` | مواصفات تصميم قبل الكود (اختياري) | واعٍ بواجهات RTL والعربية أولًا |

لماذا يوجد كل وكيل — وبأي ثلاثة منهم يمكنك البدء: [playbook/02-roles.md](playbook/02-roles.md) (EN).

## القصة

مطوّر واحد، ومنصة تحصيل كبيرة «عربية أولًا»، واكتشافٌ مفاده أن جلسة ذكاء اصطناعي واحدة تكتب الكود دون إشراف تُنتج بالضبط الأخطاء التي تتوقعها من مطوّر بلا ذاكرة، وبلا مُراجع، وبلا خوف. تكثّف خط الأنابيب هذا من إصلاح ذلك — إصدارًا بعد إصدار، على مدى أكثر من 200 إصدار. اقرأ الرواية الصادقة، بما فيها ما لا يزال لا يعمل حتى الآن: [دراسة الحالة](docs/ar/case-study.ar.md).

This methodology is fully documented in English too — [start here](README.md).

## خريطة المستودع

```
playbook/     the methodology, 9 chapters — the "why and how"
agents/       10 sub-agent templates ({{parameterized}})
prompts/      standing routines — pasted into sessions on a schedule
templates/    files that live in YOUR repo after adaptation
docs/         using this repo: getting started, placeholders, case study, FAQ (+ Arabic twins in docs/ar/)
examples/     laravel-react/ — every generic file made concrete
```

## الأسئلة الشائعة والمساهمة والترخيص

- الأسئلة الشائعة — التكلفة، الفِرَق المصغّرة، الحزم التقنية غير Laravel: [docs/faq.md](docs/faq.md) (EN)
- المساهمات مرحّب بها — خصوصًا مواءمات `examples/<your-stack>/` وأنماط أخطاء مدعومة بالأدلة: [CONTRIBUTING.md](CONTRIBUTING.md) (EN)
- الترخيص: [MIT](LICENSE) © 2026 Ali Kwan

</div>
