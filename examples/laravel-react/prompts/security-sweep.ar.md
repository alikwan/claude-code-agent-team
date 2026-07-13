> **What is this?** The authentic periodic security-sweep prompt, in its
> original Arabic operational voice — sanitized and transposed to the Acme
> Collect stack (Laravel 12 + React 19 monorepo). It fans out over four attack
> surfaces in parallel, verifies end-to-end attack paths before reporting, and
> ships one PR per confirmed vulnerability with a regression test.
> Generic English adaptation: [`prompts/security-sweep.md`](../../../prompts/security-sweep.md).

# المراجعة الأمنية الدورية — كامل الكود (Acme Collect)

> **الاستخدام:** انسخ القسم تحت `## PROMPT` وأرسله للجلسة المنسِّقة (المستوى الأعلى)،
> لتوزّعه على **4 وكلاء متوازين** — وكيل قراءة-فقط لكل سطح هجوم. **لا** تُشغّله كوكيل
> `security-reviewer` واحد: ذاك مُوجَّه نحو `git diff` لـ PR محدّد، وهذا الروتين يفحص
> **كامل** الكود.
>
> **المخرَج:** قائمة بالثغرات المُتحقَّق منها فقط (medium/high/critical) مع مسار هجوم
> end-to-end حقيقي، ثمّ **إصلاح + PR منفصل لكل ثغرة** (الدمج والمراجعة على المالك).
>
> **المدّة المتوقَّعة:** 20-40 دقيقة للمسح + وقت الإصلاح حسب عدد النتائج.

---

## السياق

Acme Collect نظام تحصيل ديون يتعامل مع بيانات حسّاسة (PII للمدينين، سجلّات دفع،
مرفقات مستندات). خرق أمني = تسريب بيانات شخصية أو تلاعب بالأموال. حدود الثقة:

- **المصادقة:** Laravel Sanctum (توكنات 7 أيام) + rate-limiting على الدخول.
- **التفويض:** RBAC (5 أدوار)، سياسات في `backend/app/Policies/`، **`super_admin`
  يتجاوز كل شيء عبر `Gate::before`** — لذا أي فحص يعتمد على الدور وحده بدل السياسة
  يفتح ثقباً لبقية الأدوار.
- **المداخل:** `backend/routes/api.php` + ملفات `routes.php` في الوحدات المستقلّة
  (`backend/app/{Collections,Messaging}/`).
- **مداخل بلا Sanctum:** webhooks مزوّدي الرسائل — موقّعة HMAC + rate-limited.
- **التكامل الخارجي:** مزوّدو SMS/WhatsApp خلف طبقة drivers (`MessagingManager`).

> **اقرأ أولاً:** `.claude/agent-memory/security-reviewer/MEMORY.md` — يحتوي الأنماط
> المعتمَدة (approved patterns). **لا تُعِد الإبلاغ** عن أيّ شيء موثَّق هناك كمقبول.

---

## PROMPT

~~~text
## مهمة: مراجعة أمنية شاملة لكامل كود Acme Collect

**الجذر:** جذر المستودع (monorepo: backend/ + frontend/)
**الفرع:** develop
**الهدف:** اكتشاف ثغرات مُتحقَّق منها (medium/high/critical) لها مسار هجوم
end-to-end حقيقي. الإثبات بقراءة الكود الفعلي — لا استنتاج من grep وحده.

### ⚠️ قواعد
- **استبعِد:** vendor/, node_modules/, frontend/dist/, backend/storage/,
  backend/bootstrap/cache/. افحص الملفات المُتتبَّعة فقط (`git ls-files`).
- **اقرأ الذاكرة أولاً:** `.claude/agent-memory/security-reviewer/MEMORY.md`.
  لا تُبلِغ عن أيّ نمط معتمَد هناك.
- **تحقّق قبل الإبلاغ:** لكل مرشَّح، تتبّع route → controller → policy → sink.
  أسقِط أيّ شيء لا تستطيع إثبات استغلاله.
- **قراءة فقط أثناء المسح** — الإصلاح مرحلة لاحقة منفصلة.

### وزّع على 4 أسطح هجوم (وكيل متوازٍ لكل سطح)

#### سطح 1 — المصادقة والتفويض
- مسارات بلا `auth:sanctum` أو بلا فحص policy تُعدِّل بيانات أو تكشف PII.
- **IDOR:** أيّ endpoint يربط نموذجاً بـ route-model-binding ({debtor}, {contract},
  {payment}, {thread}, **{attachment}**) دون `$this->authorize()` أو فحص ملكيّة.
  **انتبه بشدّة لجدول `attachments` المشترك متعدّد الأشكال (polymorphic)** — أيّ
  route يربط `{attachment}` يجب أن يتحقّق
  `model_type === ParentModel::class && model_id === $parent->id`،
  وإلا صار كل مرفق في النظام قابلاً للقراءة/الحذف عبر أي نموذج أب.
- تصعيد صلاحيات: مسارات إنشاء/تعديل المستخدمين — تحقّق من **سقف الصلاحيات**
  (لا يمنح الفاعل ما يتجاوز صلاحياته هو).
- self-scoping: التقارير وسجلّات النشاط — غير المشرفين يرون بياناتهم فقط.

#### سطح 2 — الحقن والملفات
- SQL ديناميكي من إدخال المستخدم: `whereRaw/orderByRaw/DB::raw/selectRaw` —
  **خاصّةً `orderBy` بعمود/اتجاه من الطلب دون allowlist** (الثقب الكلاسيكي).
- path traversal: مسارات تنزيل/خدمة الملفات — المسار من عمود DB لا من الطلب،
  `..` محظور، يُخدَم عبر `Storage::disk()` لا بمسار من الطلب مباشرة.
- رفع الملفات: MIME مُتحقَّق **خادمياً** (لا الامتداد فقط)، اسم مُعقَّم
  (`Str::slug`)، قرص `local` **خاص** (لا `/storage/` عام)، حجم مُقيَّد.
- تنفيذ أوامر: `exec/shell_exec/proc_open/system/Process::` بإدخال مستخدم؟

#### سطح 3 — Webhooks و SSRF
- تحقّق توقيع كل webhook: HMAC على **الجسم الخام** (لا JSON مُعاد ترميزه)،
  `hash_equals` (لا `==`)، حماية idempotency ذرّية ضدّ إعادة التشغيل.
  **تذكير:** تجاوز التحقق في بيئات local/testing نمط معتمَد — أبلِغ فقط إن دخلت
  `production` للقائمة.
- SSRF: أيّ URL يصل من webhook أو API خارجي ويُجلَب خادمياً (وسائط واردة مثلاً) —
  تحقّق **تثبيت المضيف (host pinning)** على allowlist نطاقات المزوّد قبل الجلب،
  ورفض عناوين IP الخاصة/المحجوزة (بما فيها metadata السحابة).
- معالجة الواردات: وصول أعمى لـ `$payload['field']`، type juggling، XSS مخزَّن
  عبر اسم ملف/اسم مرسِل يصل للواجهة.
- مهلات (timeouts) على كل HTTP صادر (توافر).

#### سطح 4 — الأسرار و XSS و mass-assignment
- تسريب أسرار عبر مسارات الإعدادات (`GET /api/settings...`) — تحقّق إخفاء المفاتيح
  الحسّاسة (`***`) لغير مالكي صلاحية التعديل. اعثر على أيّ مفتاح حسّاس جديد
  **خارج** قائمة الإخفاء.
- أسرار/PII في السجلّات: `Log::/logger(/activity()` — مفاتيح API، توكنات، أرقام
  هواتف كاملة، أرقام هوية.
- **XSS في React:** الحماية الافتراضية تسقط عند حقن HTML خام —
  دقّق كل استخدام لخاصية حقن الـ HTML الخطرة:
  ```bash
  grep -rn "dangerouslySetInnerHTML" frontend/src/
  ```
  أيّ HTML مبنيّ من بيانات يتحكّم بها مهاجم (أسماء مدينين، نصوص رسائل واردة،
  أسماء ملفات تعريف من webhooks) = ثغرة مخزَّنة ما لم يُعقَّم عبر مكتبة تعقيم
  (DOMPurify أو ما يوازيها) — والأفضل استبداله بعرض نصّي آمن. تحقّق أيضاً من أي
  `<a href=...>` بقيمة من المستخدم (`javascript:`).
- mass-assignment: `$guarded = []` أو `create/fill/update($request->all())` بلا
  `validated()` — هل يمكن تعيين `is_vip/is_excluded/role/balance/status`؟
- كشف الأخطاء: `catch` يُعيد `$e->getMessage()` للعميل (يجب المرور عبر معالج
  الأخطاء الموحَّد برسالة عربية — التفاصيل التقنية للسجلّ فقط).

### تنسيق كل نتيجة
- **الملف:السطر** للـ sink والـ route والـ policy.
- **مسار الهجوم:** مَن (أيّ دور) يرسل ماذا (أيّ طلب) ليحصل على ماذا.
- **الأثر** + **الخطورة** (CRITICAL/HIGH/MEDIUM).
- إن لم تجد شيئاً في سطحك: قُل ذلك صراحةً واذكر ما تحقّقت منه ووجدته نظيفاً.
~~~

---

## بعد المسح: الإصلاح وفتح PR

لكل ثغرة مُتحقَّق منها (medium فأعلى):

1. **افرَّع من develop:** `git checkout develop && git pull && git checkout -b fix/<slug>`.
2. **أصلِح بأقلّ تغيير ممكن** — حاكِ النمط الصحيح الموجود في الكود (مثل حارس
   `model_type/model_id` للمرفقات). لا تُعِد تصميم ما هو سليم.
3. **اكتب اختبار regression (أحمر/أخضر):** أثبِت أنّ الاختبار **يفشل** على الكود غير
   المُصلَح ثمّ **يمرّ** بعد الإصلاح. ضعه تحت `backend/tests/Feature/<Module>/`.
4. **شغّل:** `docker exec app composer exec pint` على الملفات المُغيَّرة + الاختبارات
   المستهدَفة (`docker exec app php artisan test --filter=<Module>`).
5. **وثِّق:** أدخِل `CHANGELOG.md` (قسم أمان، عربي + ملخّص إنجليزي) وارفع `VERSION`
   (PATCH). سجّل الثغرة في `.claude/agent-memory/security-reviewer/MEMORY.md`
   (فئة النمط + المرجع) — **دون أي أسرار أو بيانات حسّاسة في الذاكرة أبداً**.
6. **commit + push + PR منفصل** لكل ثغرة (base: develop). العنوان:
   `security(<scope>): <وصف>`. اربط مسار الهجوم في الوصف. المالك يتولّى الدمج.

> **لا تُجمِّع ثغرات غير مترابطة في PR واحد** — كل ثغرة PR مستقلّ ليتمكّن المالك من
> الدمج العاجل للحرِج دون انتظار الباقي.

**المسح النظيف يُسجَّل أيضاً:** حتى عند **صفر نتائج**، سجّل نتيجة المسح (التاريخ،
النطاق، الأسطح الأربعة، «نظيف») في ذاكرة security-reviewer وادفعها في commit من نوع
`chore(security):` — السجلّ النظيف الموثَّق هو ما يثبت انتظام العناية الدورية ويمنع
إعادة فحص عشوائية لنفس الأسطح.

---

## متى تُشغّل هذا الروتين؟

| السيناريو | التكرار |
|---|---|
| قبل release رئيسي (X.Y.0) | إلزامي |
| بعد إضافة endpoints/webhooks/رفع ملفات/منطق دفع جديد | فوراً |
| بعد تغيير في المصادقة/التفويض/السياسات | فوراً |
| صيانة دورية | شهرياً |

**استهلاك النتائج:** CRITICAL/HIGH ← PR إصلاح فوري منفصل لكل واحدة؛ MEDIUM ← PR لكل
واحدة أو تجميع المتقارب موضوعياً؛ LOW/hardening ← سجّلها في الذاكرة وعالجها في جولة
الصيانة التالية.

**تطوير الروتين:** عند اكتشاف فئة ثغرة جديدة، أضِفها إلى (1) هذا الملف (السطح
المناسب)، (2) `.claude/agents/security-reviewer.md` (القائمة المرجعية)،
(3) `.claude/agent-memory/security-reviewer/MEMORY.md` (الفئة + المثال).
