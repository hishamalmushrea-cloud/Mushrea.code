# التقرير النهائي — تنفيذ مهمة تطوير AndCode

هذا التقرير يفصل بصراحة بين ما نُفِّذ وتحقق منه، وما لم يُنفَّذ، وما لم يمكن التحقق منه في هذه البيئة.

## 1. الملخص

نُفِّذت **خمس حزم متماسكة منخفضة المخاطر**: أساس دورة المهمة ومحرك المخاطر، ثم الاستئناف/التأكيد/الإلغاء
ومحوّل استدعاء الأداة، ثم نظام القدرات، ثم ذاكرة المشروع الهندسية، ثم اكتشاف المشروع وتخطيط الفحوص
وتصنيف الفشل. كل ذلك Kotlin نقية بلا أي اعتماد على Android، مع **143 حالة اختبار جديدة** جُمِّعت
ونُفِّذت فعليًا: **156 اختبارًا ناجحًا** شملت 14 اختبارًا قائمًا. لم يُمسّ أي ملف Android، ولم يتغيّر سلوك
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

الفرع المحلي المستخدم: `feat/policy-risk-engine`، commit‑ان فوق `414c780`:

| # | commit | المحتوى | الحجم |
| --- | --- | --- | --- |
| 1 | `a179b6c` | محرك المخاطر + آلة حالات المهمة | 18 ملفًا، +1,688 |
| 2 | `1a8a67a` | الاستئناف/التأكيد/الإلغاء + محوّل استدعاء الأداة | 7 ملفات، +643 / −10 |
| 3 | `6c8a905` | نظام القدرات: العقد + التحقق + دورة الحياة | 7 ملفات، +1,090 |
| 4 | `53a0c20` | ذاكرة المشروع: لا أسرار، ولا ثقة بملخص النموذج | 5 ملفات، +611 |
| 5 | `b9c4419` | اكتشاف المشروع + تخطيط فحوص محدودة + تصنيف الفشل | 6 ملفات، +728 |

إجمالي الخمسة: 40 ملفًا، +4,750 سطرًا، صفر حذف. التسليم في هذا المستودع كخمسة ملفات patch تُطبَّق بالترتيب:
`patches/0001-feat-policy-risk-engine-and-task-state-machine.patch`،
`patches/0002-feat-task-resume-and-tool-call-mapper.patch`،
`patches/0003-feat-capability-registry.patch`، `patches/0004-feat-project-memory.patch`،
`patches/0005-feat-project-detection-and-checks.patch`.

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
| `core/task/TaskLimits.kt` | جديد | حدود المحاولات والوقت وحجم السجل لكل مهمة |
| `core/task/TaskProgress.kt` | جديد | سجل خطوات المهمة + ما يلزم للاستئناف (خطوات، محاولات، بداية) |
| `core/task/TaskResume.kt` | جديد | قواعد الاستئناف والتأكيد والإلغاء الفعلي |
| `core/policy/ToolCallMapper.kt` | جديد | `ToolCall(name, args)` → `PolicyOperation` بلا معرفة بأي CLI |
| `core/capability/CapabilityManifest.kt` | جديد | عقد القدرة: نوع، مصدر، صلاحيات، مخطط مدخلات/مخرجات، حدود، `SemVer` |
| `core/capability/CapabilityValidator.kt` | جديد | تحقق المخطط والتوافق والصلاحيات، بعيوب حاجبة وأخرى للتنبيه |
| `core/capability/CapabilityRegistry.kt` | جديد | دورة حياة القدرة مع بوابة تفعيل وسجل أحداث |
| `core/memory/MemoryFact.kt` | جديد | نوع الحقيقة ومصدرها وحدودها، وحالة «تحتاج تحققًا» |
| `core/memory/ProjectMemory.kt` | جديد | تسجيل/تحقق/استرجاع مرتَّب بميزانية أحرف، ورفض الأسرار |
| `core/buildcheck/ProjectProfile.kt` | جديد | نوع المشروع، أمر فحص آمن، نتيجة فحص، تصنيف الفشل |
| `core/buildcheck/ProjectDetector.kt` | جديد | كشف نوع المشروع من قائمة الملفات وقائمة فحوص لكل نوع |
| `core/buildcheck/FailureClassifier.kt` | جديد | بيئة/شيفرة/اختبار/مهلة/مجهول من مخرجات الفحص |
| `core/security/SecretRedaction.kt` | تعديل (+5) | نمط لصيغة `--password value`، لأن المحرك أصبح يعرض سطور أوامر |
| 12 ملف اختبار جديد + توسيع `SecretRedactionTest` | جديد | 143 حالة تغطي الحالات الحساسة |
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

بعد إعادة تشغيل التطبيق:
TaskProgress المحفوظ ──► TaskResume.plan(now, limits)
        ├─ خطوة ذات أثر جانبي كانت RUNNING → تنتظر تأكيدًا صريحًا (لا تُعاد تلقائيًا)
        ├─ خطوة بلا أثر جانبي كانت RUNNING → تُعاد بأمان
        ├─ attempts / maxWallClockMs مستنفدة → Blocked (والساعة لا تُصفَّر بالاستئناف)
        └─ TaskResume.cancel() → يعيد معرّفات الخطوات التي يجب إيقافها فعلًا

القدرات:
CapabilityManifest (غير موثوق) ──► CapabilityValidator (مخطط/توافق/صلاحيات/حدود)
        │                                   │
        │                                   └─ 3 صلاحيات ممنوعة تمامًا:
        │                                      قراءة الأسرار، تفليش جهاز، تعطيل السياسة
        ▼
CapabilityRegistry: DRAFT → VALIDATED → TESTED → ENABLED → DISABLED، وREVOKED نهائي
        └─ التفعيل يشترط: تحققًا نظيفًا + نجاح الاختبارات المعلنة + منح المستخدم + مراجعة المصدر
           (سير العمل لا يملك مسارًا مختصرًا: كل أداة يستدعيها تمرّ على RiskEngine كالاستدعاء المباشر)

الذاكرة والبناء:
حقائق المشروع ──► ProjectMemory (رفض الأسرار + ثقة مشروطة بالـ commit والعمر)
                   └─ recall(query) مرتَّب بميزانية أحرف، ويُبلّغ عمّا حذفه، وCONSTRAINT تُضمَّن دائمًا
ملفات المشروع ──► ProjectDetector (بلا تنفيذ) ──► فحوص محدودة معلومة الهوية والحدود
مخرجات الفحص ──► FailureClassifier (مهلة ← بيئة ← اختبار ← شيفرة ← مجهول) وليس تخمينًا
```

**حدود مهمة:** `CommandInspector` مطابِق أنماط لا محلّل shell؛ الأوامر المُشوَّشة أو السكربتات على القرص
تُفلت منه. PRoot ليس عزلًا أمنيًا كاملًا. كل هذا موثّق في `docs/POLICY_ENGINE.md` لا مخفيًا.

## 5. الاختبارات — ما شُغِّل فعلًا

بلا Gradle (لا SDK ولا وصول إلى Maven)، جُمِّعت الشيفرة الجديدة مع **ملفات الاختبار الملتزمة كما هي**
بـ `kotlinc` 2.4.21 على OpenJDK 25 (هدف JVM 17)، ونُفِّذت عبر مشغّل انعكاسي مع بديل محلي لواجهة JUnit 4
(لأن `junit.jar` غير متاح). السجل الحرفي: `patches/verification-log.txt`.

```
CapabilityRegistryTest    15/15
CapabilityValidatorTest   16/16
CommandInspectorTest      12/12
FailureClassifierTest      9/9
PolicyAuditLogTest         4/4
ProjectDetectorTest       12/12
ProjectMemoryTest         15/15
RiskEngineTest            23/23
ToolCallMapperTest        11/11
SecretRedactionTest       15/15   (14 قائمة + 1 جديد)
TaskResumeTest            13/13
TaskStateMachineTest      11/11
RESULT passed=156 failed=0
```

والمصادر في التشغيل الأخير مأخوذة من نسخة استنساخ نظيفة طُبِّق عليها الـ patch‑ان بـ `git am`، أي أن
المُختبَر هو نفس ما سيصل إلى المستودع.

**خمسة إخفاقات ظهرت في التشغيل وأُصلحت** (لم يُخفَ أي منها):
1. `./gradlew … assembleRelease` لم يُصنَّف كإصدار لأن النمط كان camelCase والسطر يُحوَّل لأحرف صغيرة.
2. صيغة `--password value` كانت تتسرّب من `SecretRedaction`؛ أُضيف نمط لها مع اختبار.
3. `writeFile` صُنِّف `UNKNOWN` لأن فصل camelCase كان يستبدل المطابقة كلها بمسافة (`write ile`).
4. قدرة من مصدر غير موثوق كانت **مستحيلة التفعيل** حتى بعد المراجعة، لأن عيب `UNTRUSTED_SOURCE` كان
   يُحتسب حاجبًا ويتعارض مع بوابة المراجعة في `enable()`؛ صار العيب حاجبًا/غير حاجب صراحةً.
5. معرّفات قدرات عميقة (`a.b.c.d.e.x`) كانت تُقبل بينما المفروض رفضها؛ حُدّ النمط إلى أربعة مقاطع.

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
- عند الاستئناف، لا تُعاد خطوة ذات أثر جانبي (كتابة/دفع/نشر) تلقائيًا لأنها قد تكون وقعت قبل الإغلاق؛
  تنتظر تأكيدًا. وميزانية الوقت لا تُصفَّر بالاستئناف، فلا مأزق يمتد بلا حد.
- القدرات: ثلاث صلاحيات ممنوعة تمامًا (قراءة الأسرار، تفليش جهاز، تعطيل السياسة) تُرفض في التحقق، ولا
  يمكن تفعيل قدرة بغير تحقق نظيف + اختبارات ناجحة + منح صريح من المستخدم + مراجعة المصادر غير المضمّنة،
  و`REVOKED` نهائي. وتغييرات النواة (`CORE_CHANGE`) لا تُقبل كبيانات من مصدر خارجي.
- الذاكرة: كل حقيقة تُفحص بنمط كشف الأسرار نفسه قبل الحفظ، وإذا احتوت اعتمادًا تُرفض ولا تُخزَّن منقّصة؛
  وملخص الوكيل لا يُعدّ موثّقًا أبدًا قبل تحققه من الكود، والحقيقة المرتبطة بـ commit آخر أو الأقدم من 14
  يومًا تُعاد موسومة بأنها تحتاج تحققًا.
- الفحوص: لا تُنفَّذ أوامر من المشروع؛ الأوامر المولَّدة محدودة بقائمة تنفيذيات مسموحة، وتُمرَّر كـ argv بلا
  shell (لا `; | & $ \` > <`)، وبحدود زمن وحجم مخرجات. وفشل الإعداد (SDK/شبكة/قرص) يُصنَّف بيئيًا لا
  «شيفرة معطوبة».

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

1. **P2.1 — ربط المحرك بمسار الموافقة** (APK جديد): النصف المستقل عن الـ CLI جاهز
   (`ToolCallMapper`)؛ الباقي محوّل واحد لكل backend ينتج `ToolCall` من `PermissionRequest` ومن طلبات
   `ClaudePermissionBridge`، ثم `ALLOW`/`REQUIRE_APPROVAL`/`DENY` على المسار القائم، مع مفاتيح
   `PolicyConfig` في الإعدادات. الملف الحرج: `feature/chat/ChatViewModel.kt` (134 KB) — تدخّل جراحي
   ومراجعة diff كاملة.
2. **تثبيت حالة المهمة**: حفظ `TaskProgress` في تخزين التطبيق وربط زر الإلغاء بـ`TaskResume.cancel()`
   (القواعد جاهزة ومختبرة، والنقل إلى التخزين هو ما تبقّى).
3. **سجل تدقيق مثبَّت** (append-only) إن لزم البقاء بعد إغلاق التطبيق؛ الحالي في الذاكرة فقط.
4. **تنفيذ الفحوص فعليًا** عبر مسار الطرفية/الـ runtime القائم مع `core/policy` أمامه، وتحليل تقارير
   الاختبارات لكل اختبار (حاليًا يُصنَّف نوع الفشل فقط)، وحلقة إعادة محاولة محدودة.
5. **تثبيت الذاكرة وسجل القدرات** في تخزين التطبيق + شاشة القدرات وشاشة الذاكرة (APK جديد).
6. **P5 — متصفح لاختبار الويب، أدوات USB/ADB للقراءة فقط، إشعارات/تلخيص صوتي** بعد استقرار ما سبق.
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
  الفرع `feat/policy-risk-engine`، خمسة commits (`a179b6c`, `1a8a67a`, `6c8a905`, `53a0c20`, `b9c4419`)
  فوق `414c780`، شجرة عمل نظيفة بعدها.
- **لم يُدفع** أي شيء إلى `yuga-hashimoto/and-code`: حساب GitHub المتصل لا يملك صلاحية `push`
  (`permissions.push = false` بحسب API). لذلك سُلِّم العمل كـ patch.
- في هذا المستودع (`Mushrea.code`، الفرع `arena/ea667b72-mushrea-code`) أُضيفت:
  `patches/0001-…patch` … `patches/0005-…patch`، `patches/verification-log.txt`،
  `docs/andcode/BASELINE_AR.md`، `PLAN_AR.md`، `VERIFICATION_AR.md`، `FINAL_REPORT_AR.md` (هذا الملف)،
  و`README_AR.md`. وقد تم **دفع الفرع إلى `origin`** (`hishamalmushrea-cloud/Mushrea.code`).
- لم يُنشأ إصدار، ولم يُنشر أي artifact، ولم يُحذف أو يُعاد تسمية أي شيء في المستودع الأصلي.
