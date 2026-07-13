> **What is this?** The authentic recurring-patterns full-codebase sweep prompt,
> in its original Arabic operational voice — sanitized and transposed to the
> Acme Collect stack (Laravel 12 + React 19 monorepo). Run it monthly, before
> major releases, or right after a very large PR merges; it hunts the five bug
> patterns that repeatedly slipped between layers in production.
> Generic English adaptation: [`prompts/recurring-patterns-sweep.md`](../../../prompts/recurring-patterns-sweep.md).

# الأنماط المتكرّرة — مسح شامل لكامل الكود (Acme Collect)

> **الاستخدام:** انسخ القسم تحت `## PROMPT` بالكامل وأرسله لوكيل قراءة-فقط
> (qa-engineer أو وكيل عام) عندما تريد اكتشاف هذه الأنماط الـ5 في كامل الكود
> (وليس فقط في PR محدّد).
>
> **المدّة المتوقَّعة:** 30-45 دقيقة (نطاق كامل).
> **المخرَج:** تقرير منظَّم بكل النتائج الموسومة `[PATTERN-N]` مع خطوات إعادة الإنتاج.

---

## السياق التاريخي

وُثِّقت 5 أنماط متكرّرة من الأخطاء أثناء بناء وحدات كبيرة في النظام الأصلي (30+ خطأ
في مرحلتين متتاليتين). كانت تنزلق من الاختبارات لأنّ الاختبارات تتحقّق من كل طبقة
معزولة، ولا أحد يفحص **نقطة التماس بين الطبقات**. تعريفات الوكلاء في `.claude/agents/`
محدَّثة بفحوصات المنع للـ PRs المستقبلية — **هذا الـ sweep لاكتشاف الحالات الموجودة
بالفعل** قبل أن يكتشفها مستخدم نهائي.

---

## PROMPT

~~~text
## مهمة: مسح شامل للكود لاكتشاف 5 أنماط أخطاء متكرّرة — Acme Collect

**الجذر:** جذر المستودع (monorepo: backend/ + frontend/)
**الفرع:** develop (أو ما هو نشط حالياً)
**الإطار الزمني:** 30-45 دقيقة كحد أقصى

### ⚠️ مهم
- **لا تُصلح** — اكتشف فقط.
- **لا تشغّل اختبارات كاملة** — استخدم grep + قراءة الكود.
- **التزم بالـ5 أنماط** — لا تستكشف أبعد منها.
- **التقرير ≤3000 كلمة** — موجز ودقيق.
- **استبعِد:** vendor/, node_modules/, frontend/dist/, backend/storage/,
  backend/bootstrap/cache/. افحص الملفات المُتتبَّعة فقط (`git ls-files`).
- **اقرأ `.claude/agent-memory/qa-engineer/MEMORY.md` قبل البدء** — قد يحتوي
  استثناءات موثَّقة (إيجابيات كاذبة معروفة) لا يجوز إعادة الإبلاغ عنها.

---

### PATTERN-1: عدم تطابق العقد React ↔ API
مكوّن أو hook يقرأ مفتاحاً مختلفاً عمّا يُرجعه الـ controller فعلاً.

**كيفية الاكتشاف:**
1. لكل hook في `frontend/src/hooks/`، استخرج الـ URL من `queryFn`/`mutationFn`
   (grep عن `api.get|api.post|api.put|useQuery|useMutation`).
2. لكل URL، اعثر على الـ controller method: `grep -rn "Route::" backend/routes/ backend/app/*/routes.php`.
3. اقرأ مفاتيح الاستجابة التي يُرجعها الـ controller فعلاً.
4. اقرأ ما يستهلكه الـ hook/المكوّن (أنواع TypeScript + الوصول الفعلي `data.foo.bar`).
5. **أبلغ عن أي مفتاح يقرؤه الـ frontend ولا يُصدِره الـ controller** — نوع TypeScript
   مكتوب بالتخمين لا يحميك؛ قارن بالمخرَج الفعلي.

**حالات شائعة:**
- الـ hook يقرأ `debtor.name` ↔ الـ API يُرجع `display_name`
- الـ hook يتوقّع مصفوفة، الـ API يُرجع object بمفاتيح
- `data` ↔ `data.data` (استجابة paginated غير مفكوكة)
- wrapper إضافي: `stats` ↔ `stats.buckets`

**ركّز على:** لوحات الإشراف، المكوّنات البيانية (charts)، النماذج التي تحفظ عبر mutation.

---

### PATTERN-2: عدم تطابق Enum/نص شبحي
الكود يشير إلى قيمة enum أو مفتاح setting أو عمود **غير موجود**.

```bash
# A. قيم enum داخل in_array — تحقّق أن كل نص موجود في صنف الـ enum
grep -rn "in_array(" backend/app/ | grep -E "'[a-z_]+'.*'[a-z_]+'" | head -30

# B. مفاتيح الإعدادات — كل Setting::get يجب أن يقابله بذر في seeder
grep -rohE "Setting::get\('[a-z_.]+'" backend/app/ | sort -u > /tmp/gets.txt
grep -rohE "'[a-z_.]+'" backend/database/seeders/SettingsSeeder.php | sort -u > /tmp/seeded.txt
# أي مفتاح في gets وليس في seeded = مفتاح شبحي (يعمل بالقيمة الافتراضية صامتاً)

# C. أعمدة eager-load — تحقّق أن rel:id,col يطابق مخطط الجدول فعلاً
grep -rn "with(\['" backend/app/ | grep ":id," | head -30
# لكل واحد: تحقّق من وجود الأعمدة في migration الجدول
```

**تذكير من التاريخ:** الأخطاء هنا نمطية — جدول يملك `display_name` والكود يطلب
`name`؛ enum يملك `promise_made` والكود يفحص `promise_to_pay`. **اقرأ المخطط
الفعلي وصنف الـ enum الفعلي؛ لا تفترض.**

---

### PATTERN-3: ميزات شبحية (مُعرَّفة لكن غير موصولة)
الميزة موجودة كـ class/عمود/setting لكن **لا أحد يُطلقها/يقرؤها/يفرضها**.

```bash
# A. إشعارات بلا مواقع dispatch
for cls in $(find backend/app/Notifications -name "*.php" -exec basename {} .php \;); do
    count=$(grep -rn "new $cls\|->notify.*$cls\|$cls::dispatch" backend/app/ | wc -l)
    [ "$count" = "0" ] && echo "[PATTERN-3-A] $cls — ZERO dispatch sites"
done

# B. أحداث بلا مواقع broadcast
for cls in $(find backend/app -path "*Events*" -name "*.php" -exec basename {} .php \;); do
    count=$(grep -rn "new $cls\|broadcast(new $cls\|$cls::dispatch" backend/app/ | wc -l)
    [ "$count" = "0" ] && echo "[PATTERN-3-B] $cls — ZERO broadcast sites"
done

# C. مفاتيح تبديل (toggles) مبذورة ولا تُقرأ أبداً
grep -rohE "'[a-z_.]+\.enabled'" backend/database/seeders/ | tr -d "'" | sort -u | \
    while read key; do
        count=$(grep -rn "Setting::get('$key'" backend/app/ | wc -l)
        [ "$count" = "0" ] && echo "[PATTERN-3-C] $key — toggle never checked"
    done

# D. صلاحيات مبذورة ولا تُفرَض أبداً
grep -rohE "'[a-z_]+\.[a-z_]+'" backend/database/seeders/RolesAndPermissionsSeeder.php | \
    tr -d "'" | sort -u | \
    while read perm; do
        count=$(grep -rn "permission:$perm\|->can('$perm'\|authorize.*'$perm'" backend/app/ backend/routes/ | wc -l)
        [ "$count" = "0" ] && echo "[PATTERN-3-D] $perm — never enforced"
    done

# E. مستمعو Echo في الـ frontend بلا broadcaster خلفي
grep -rn "\.listen('" frontend/src/ | sed -E "s/.*listen\('\.?([a-z._-]+)'.*/\1/" | sort -u | \
    while read evt; do
        count=$(grep -rn "broadcastAs.*'$evt'" backend/app/ | wc -l)
        [ "$count" = "0" ] && echo "[PATTERN-3-E] frontend listens for '$evt' — no backend broadcaster"
    done

# F. أعمدة جديدة على جداول قائمة — تحقّق من $fillable في الـ Model
# اقرأ كل migration من نوع add_columns_to_X_table ثم افحص Model X
```

---

### PATTERN-4: فجوات توصيل الطوابير (Jobs/Queues)
Job يستخدم queue غير مُعرَّف في Horizon ← الـ jobs تُدفَع لكنّها لا تُنفَّذ أبداً.

```bash
# A. كل الطوابير المخصّصة المستخدمة في jobs مقابل config/horizon.php
grep -rohE "onQueue\('[a-z_-]+'" backend/app/ | sed -E "s/.*'([a-z_-]+)'/\1/" | sort -u | \
    while read q; do
        in_horizon=$(grep -c "'$q'" backend/config/horizon.php)
        [ "$in_horizon" = "0" ] && echo "[PATTERN-4-A] Queue '$q' not in horizon.php"
    done

# B. مهامّ مجدوَلة مكرّرة في نفس التوقيت
grep -B1 -E "dailyAt\('[0-9:]+'\)" backend/routes/console.php | head -50
# تحقّق يدوياً أنّ لا مهمّتين في نفس الوقت تُطلقان نفس الإشعار

# C. jobs تتشارك نفس علم الـ idempotency
grep -rn "reminder.*sent\|processed_at\|dispatched_at" backend/app/*/Jobs/ | head -20
# لو job وcommand يستخدمان نفس العلم = إرسال مكرّر محتمل
```

---

### PATTERN-5: انجراف التوثيق عن الكود
CLAUDE.md / CHANGELOG.md / docs/ تذكر classes أو settings أو قنوات **لا تطابق**
الكود الفعلي.

```bash
# A. أسماء classes في التوثيق مقابل الملفات الفعلية
grep -rohE "[A-Z][a-zA-Z]+(Job|Notification|Event|Service|Controller|Listener)" \
    CLAUDE.md docs/ | sort -u | \
    while read cls; do
        found=$(find backend/app/ -name "$cls.php" 2>/dev/null | head -1)
        [ -z "$found" ] && echo "[PATTERN-5-A] $cls mentioned in docs but NOT in backend/app/"
    done

# B. مفاتيح settings في التوثيق مقابل الـ seeders
# قارن docs/3_Developer_Docs/3.3_Settings_Reference.md بمخرَج البذر الفعلي

# C. قنوات WebSocket في التوثيق مقابل backend/routes/channels.php
grep -rohE "'[a-z._{}-]+'" CLAUDE.md | sort -u | head -20   # قارن يدوياً

# D. تاريخ "Last updated" في README مقابل آخر commit
```

---

## تنسيق التقرير المطلوب

- **Summary:** عدد النتائج لكل نمط `[PATTERN-1..5]` + المجموع، مع الفرع والتاريخ.
- **لكل نتيجة:** الوسم `[PATTERN-N]` + الملف:السطر في الطرفين (frontend + backend
  عند الاقتضاء) + الأثر + الخطورة (HIGH|MEDIUM|LOW) + إصلاح مقترح من سطر واحد.
- **Confidence Notes:** ما فُحص بشمولية 100% وما قد يكون ناقصاً (مثلاً: مستمعون
  inline خارج hooks/).
- **Suggested Priority:** 🔴 فوراً [HIGH] | 🟠 هذا السبرنت [MEDIUM] | 🟡 لاحقاً [LOW].

نصائح: ابدأ بـ PATTERN-3 (الأسرع — grep بسيط)، واترك PATTERN-1 آخراً (الأبطأ —
قراءة كل زوج hook/controller). عند الشك، اقرأ الملف — لا تستنتج من grep وحده.

ابدأ الآن.
~~~

---

## ملاحظات للمستخدم (خارج الـ prompt)

| السيناريو | التكرار |
|---|---|
| بعد PR كبير (50+ ملف) | فوراً بعد الدمج |
| قبل release رئيسي (X.Y.0) | إلزامي |
| صيانة دورية | شهرياً |
| عند رصد خطأ user-facing مماثل | فوراً |

**استهلاك التقرير:** HIGH ← PR إصلاح فوري لكل واحدة؛ MEDIUM ← PR صيانة مجمَّع كل
أسبوعين؛ LOW ← سجّلها وعالجها في الجولة التالية.

**تطوير الـ sweep:** عند اكتشاف نمط جديد، أضِفه إلى (1) هذا الملف (PATTERN-6 مع
grep recipe)، (2) تعريفات الوكلاء الأربعة (qa/backend/frontend/integration)،
(3) `.claude/agent-memory/qa-engineer/MEMORY.md`.
