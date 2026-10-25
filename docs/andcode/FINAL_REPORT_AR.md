# التقرير النهائي — تنفيذ مهمة تطوير AndCode

هذا التقرير يفصل بصراحة بين ما نُفِّذ وتحقق منه، وما لم يُنفَّذ، وما لم يمكن التحقق منه في هذه البيئة.

## 1. الملخص

نُفِّذت **الحزمة الأولى المتماسكة منخفضة المخاطر**: أساس دورة المهمة ومحرك المخاطر داخل التطبيق،
كـ Kotlin نقية بلا أي اعتماد على Android، مع 51 حالة اختبار جديدة (50 في الحزم الجديدة + 1 في
`SecretRedactionTest`) جُمِّعت ونُفِّذت فعليًا: 65 اختبارًا ناجحًا شملت 14 اختبارًا قائمًا. لم يُمسّ أي ملف Android، ولم يتغيّر سلوك
التطبيق الحالي، ولم يُبنَ أي APK.

الوظيفة الجديدة **غير موصولة بالواجهة بعد** بشكل مقصود: الأساس يُراجع أولًا، والربط هو التغيير التالي
(موصوف بدقة في القسم 10 وفي `PLAN_AR.md`).

## 2. خط الأساس

| العنصر | القيمة |
| --- | --- |
| المستودع | `https://github.com/yuga-hashimoto/and-code` (MIT) |
| الفرع | `main` |
| الـ commit | `414c780649cebb80e3815a6f02115145a5bd3c66` |
| الإصدار | `v1.2.27` |
| اختبارات المشروع قبل التعديل | **لم تُشغَّل** — لا JDK/SDK/شبكة إلى Maven في هذه البيئة |
| اختبارات المشروع بعد التعديل | **لم تُشغَّل عبر Gradle** — نفس السبب |

حجم المشروع المفحوص: 495 ملف Kotlin في الإنتاج (68,096 سطرًا)، 179 ملف اختبار (29,325 سطرًا، 1,478
`@Test`)، وحدتان Gradle، نكهتان `github`/`fdroid`.

## 3. التغييرات

الفرع المحلي المستخدم: `feat/policy-risk-engine` والـ commit: `a179b6c` (18 ملفًا، +1,688 سطرًا، صفر حذف).
التسليم في هذا المستودع كملف patch: `patches/0001-feat-policy-risk-engine-and-task-state-machine.patch`.

| ملف | النوع | السبب |
| --- | --- | --- |
| `core/policy/RiskLevel.kt` | جديد | ترتيب شدّة المخاطر |
| `core/policy/PolicyOperation.kt` | جديد | وصف مُطبَّع للطلب يفصل المحرك عن أسماء أدوات كل CLI |
| `core/policy/PolicyConfig.kt` | جديد | الإعدادات التي يملكها المستخدم (لا النموذج)، وقيم افتراضية محافظة |
| `core/policy/RiskAssessment.kt` | جديد | `ALLOW`/`REQUIRE_APPROVAL`/`DENY` + أكواد أسباب ثابتة |
| `core/policy/PolicyPaths.kt` | جديد | تقنين مسارات الضيف ومنع `..` من التسلل، ومطابقة أنماط |
| `core/policy/CommandInspector.kt` | جديد | قراءة سطر الأوامر مقطعًا مقطعًا وكشف الأوامر التدميرية/المميزة/النشر |
| `core/policy/RiskEngine.kt` | جديد | `assess()` دالة نقية: المستوى والقرار والأسباب |
| `core/policy/PolicyAuditLog.kt` | جديد | سجل قرارات محدود (200) يخزّن الأكواد والملخص المنقّى فقط |
| `core/task/TaskState.kt` | جديد | حالات المهمة + `TaskFacts` |
| `core/task/TaskStateMachine.kt` | جديد | جدول الانتقالات + حرّاس النجاح والموافقة |
| `core/security/SecretRedaction.kt` | تعديل (+5) | نمط لصيغة `--password value`، لأن المحرك أصبح يعرض سطور أوامر |
| 4 ملفات اختبار جديدة + توسيع `SecretRedactionTest` | جديد | 51 حالة تغطي الحالات الحساسة |
| `docs/POLICY_ENGINE.md` | جديد | تصميم، حدود، وخطة الربط |
| `README.md` | تعديل (سطر) | ربط التوثيق الجديد في قسم Documentation |

## 4. المعمارية بعد التعديل

```
أداة الوكيل (CLI/جلسة)
        │  طلب موافقة أو استدعاء أداة
        ▼
PermissionRequest / ClaudePermissionBridge   ← القائم، لم يُغيَّر
        │  (الربط: التغيير التالي — لم يُنفَّذ بعد)
        ▼
PolicyOperation  ──►  RiskEngine.assess(operation, PolicyConfig)
        │                     │
        │                     ├─ PathFinding: داخل/خارج مساحة العمل، مسار سري، مسار محمي
        │                     ├─ CommandInspector: تدميري / رفع صلاحيات / force-push / إصدار / أسرار / شبكة
        │                     └─ عتبة التغيير الجماعي
        ▼
RiskAssessment(level, decision, reasons)  ──►  PolicyAuditLog (محدود، منقّى)
        │
        ├─ ALLOW            → تنفيذ بلا سؤال
        ├─ REQUIRE_APPROVAL → ورقة الموافقة القائمة + الأسباب
        └─ DENY             → رفض ثابت لا يمكن تجاوزه

TaskStateMachine: DRAFT → INSPECTING → PLANNED → AWAITING_APPROVAL → READY → RUNNING
                  → VERIFYING → CHECKS_PASSED → READY_FOR_REVIEW → READY_TO_SHIP → SHIPPED
                  (+ FAILED / PAUSED / CANCELLED)، بحرّاس على checksGreen و approved.
```

**حدود مهمة:** `CommandInspector` مطابِق أنماط لا محلّل shell؛ الأوامر المُشوَّشة أو السكربتات على القرص
تُفلت منه. PRoot ليس عزلًا أمنيًا كاملًا. كل هذا موثّق في `docs/POLICY_ENGINE.md` لا مخفيًا.

## 5. الاختبارات — ما شُغِّل فعلًا

بلا Gradle (لا SDK ولا وصول إلى Maven)، جُمِّعت الشيفرة الجديدة مع **ملفات الاختبار الملتزمة كما هي**
بـ `kotlinc` 2.4.21 على OpenJDK 25 (هدف JVM 17)، ونُفِّذت عبر مشغّل انعكاسي مع بديل محلي لواجهة JUnit 4
(لأن `junit.jar` غير متاح). السجل الحرفي: `patches/verification-log.txt`.

```
CommandInspectorTest   12/12
PolicyAuditLogTest      4/4
RiskEngineTest         23/23
SecretRedactionTest    15/15   (14 قائمة + 1 جديد)
TaskStateMachineTest   11/11
RESULT passed=65 failed=0
```

**فشلان ظهرا في التشغيل الأول وأُصلحا** (لم يُخفَ أي منهما):
1. `./gradlew … assembleRelease` لم يُصنَّف كإصدار لأن النمط كان camelCase والسطر يُحوَّل لأحرف صغيرة.
2. صيغة `--password value` كانت تتسرّب من `SecretRedaction`؛ أُضيف نمط لها مع اختبار.

**لم يُشغَّل:** `detekt`, `spotlessCheck`, `lintGithubDebug`, `testGithubDebugUnitTest` بكاملها (1,478
اختبارًا)، `assembleGithubDebug/Release`, أي `androidTest`، وأي اختبار على جهاز.

## 6. البناء

لم يُبنَ أي APK أو AAB؛ لا نكهة `github` ولا `fdroid`، ولا artifacts. لا ادعاء بخلاف ذلك.

## 7. الأمان

**ما تم:**
- قرار `DENY` ثابت لا يمكن للنموذج أو لرسالة مستخدم أو لـ`PolicyConfig.permissive()` تحويله إلى موافقة.
- كتابة/حذف ملفات الاعتماد → `DENY`؛ قراءتها → `HIGH` تتطلب موافقة.
- رفض `rm -rf /`, `mkfs`, `dd of=/dev/…`, `pm clear`, `fastboot flash|erase|oem unlock`, `curl … | sh`,
  `--dangerously-skip-permissions`.
- تقنين المسارات قبل الحكم، فلا يمر `../../etc/hosts` ككتابة داخلية.
- تنقية كل تفصيل سبب وكل ملخص عملية عبر `SecretRedaction` قبل التخزين أو العرض؛ السجل يخزّن أكوادًا لا نصوصًا.
- سجل محدود الحجم (200) لمنع تضخّم الذاكرة في جلسة طويلة.

**ما لم يُتحقق منه:** لا شيء من هذا **مفعَّل** بعد لأن المحرك غير موصول؛ ولا يمكن اعتباره حدًا أمنيًا
بمعزل عن الـ PRoot الذي هو أيضًا ليس عزلًا كاملًا. لم يُجرَ أي اختبار اختراق أو تحليل أمني خارجي.

## 8. التوافق

- لم يُمسّ أي ملف Android أو Gradle أو توقيع أو نكهة أو `minSdk`؛ الإضافات في حزم جديدة `core/policy`
  و`core/task` لا يستوردها أي كود قائم.
- الملف القائم الوحيد المعدَّل: `core/security/SecretRedaction.kt` (+5 أسطر نمط واحد). الاختبارات الـ14
  القائمة له تمر كما هي بعد التعديل — تم التحقق من ذلك بالتشغيل الفعلي.
- `README.md`: سطر واحد في قائمة التوثيق.
- لم تُضف أي اعتمادية، ولا Flutter/NDK/CMake، ولا خدمة مدفوعة، ولا صلاحية Android جديدة.

## 9. التكاليف والاعتماديات

- لا اعتماديات جديدة في المشروع.
- تكلفة الجلسة فقط: `kotlin-compiler` من npm و`jdk4py` من PyPI استُخدما **في بيئة الاختبار فقط**،
  وليسا جزءًا من المستودع ولا من التطبيق.

## 10. المتبقي (بحسب الأولوية)

1. **P2.1 — ربط المحرك بمسار الموافقة** (APK جديد): محوّل `PermissionRequest`/طلبات
   `ClaudePermissionBridge` → `PolicyOperation`، ثم `ALLOW`/`REQUIRE_APPROVAL`/`DENY` على المسار القائم،
   مع مفاتيح `PolicyConfig` في الإعدادات. الملف الحرج: `feature/chat/ChatViewModel.kt` (134 KB) — تدخّل
   جراحي ومراجعة diff كاملة.
2. **سجل تدقيق مثبَّت** (append-only) إن لزم البقاء بعد إغلاق التطبيق؛ الحالي في الذاكرة فقط.
3. **P3 — نظام القدرات والذاكرة الهندسية**: عقد قدرة بمخطط وإصدارات ودورة حياة، وذاكرة مرتبطة بـ commit.
4. **P1.b — استئناف/إلغاء للمهمة**: تخزين `TaskState` وحفظ الخطوات المكتملة ومنع تكرار الكتابة عند الاستئناف،
   وإلغاء فعلي للعمليات القابلة للإيقاف.
5. **P4 — اكتشاف المشروع وتشغيل الفحوص المحدودة** وتصنيف الفشل (بيئة/شيفرة/اختبار).
6. **إكمال العرض في الواجهة**: مركز المهام، خطة التغيير، القدرات، نتائج المهمة، الأمان والموافقات.
7. **P5**: وكيل متصفح، أدوات USB/ADB للقراءة فقط (المحرك يرفض عمليات الجهاز التدميرية أصلًا)،
   تلخيص صوتي/إشعارات.

## 11. طريقة التجربة (لغير المتخصص)

لا يوجد حتى الآن شيء **مرئي** داخل التطبيق — وهذا مقصود: أُنجز الأساس المنطقي الموسَّع باختبارات، ولم تُضف
أزرار أو شاشات تدّعي وظائف غير موصولة. للتجربة اليوم:

```bash
git clone https://github.com/yuga-hashimoto/and-code.git
cd and-code
git checkout -b feat/policy-risk-engine
git am /path/to/patches/0001-feat-policy-risk-engine-and-task-state-machine.patch

# الاختبارات (تحتاج JDK 17 + Android SDK كأي بناء آخر للمشروع)
./gradlew :app:testGithubDebugUnitTest --tests 'com.yugahashimoto.andcode.core.policy.*'
./gradlew :app:testGithubDebugUnitTest --tests 'com.yugahashimoto.andcode.core.task.*'
./gradlew detekt spotlessCheck
```

ثم لقراءة التصميم: `docs/POLICY_ENGINE.md` (بالإنجليزية، بأسلوب توثيق المستودع).
ولقراءة التحليل العربي: `docs/andcode/BASELINE_AR.md` ثم `PLAN_AR.md` ثم `VERIFICATION_AR.md`.

## 12. حالة Git

- العمل جرى على نسخة استنساخ حقيقية للمستودع المستهدف في مسار مؤقت خارج هذا المستودع:
  الفرع `feat/policy-risk-engine`، commit واحد `a179b6c` فوق `414c780`، شجرة عمل نظيفة بعده.
- **لم يُدفع** أي شيء إلى `yuga-hashimoto/and-code`: حساب GitHub المتصل لا يملك صلاحية `push`
  (`permissions.push = false` بحسب API). لذلك سُلِّم العمل كـ patch.
- في هذا المستودع (`Mushrea.code`، الفرع `arena/ea667b72-mushrea-code`) أُضيفت:
  `patches/0001-…patch`، `patches/verification-log.txt`، `docs/andcode/BASELINE_AR.md`،
  `PLAN_AR.md`، `VERIFICATION_AR.md`، `FINAL_REPORT_AR.md` (هذا الملف)، و`README_AR.md`.
- لم يُنشأ إصدار، ولم يُنشر أي artifact، ولم يُحذف أو يُعاد تسمية أي شيء في المستودع الأصلي.
