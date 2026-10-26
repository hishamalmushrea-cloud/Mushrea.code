# التحقق الفعلي — ما شُغِّل وما لم يُشغَّل

## القيد البيئي

هذه الجلسة تعمل داخل حاوية بلا:

- JDK (تم توفيره لاحقًا محليًا من حزمة `jdk4py` على PyPI، OpenJDK 25)،
- Android SDK،
- وصول إلى Maven Central أو `dl.google.com` (المتاح: `github.com`, `registry.npmjs.org`, `pypi.org`).

لذلك **`./gradlew` لا يمكن تشغيله هنا**، وأي ادعاء بنتائج `detekt`/`spotless`/`lint`/`assemble` أو
بتشغيل 1,478 اختبار المشروع سيكون غير صحيح. تعذّر أيضًا تشغيل `spotlessCheck` لأن ktlint يُحمَّل من
Maven.

## ما شُغِّل فعلًا

بدل الاكتفاء بالقراءة، جُمِّعت الشيفرة الجديدة (Kotlin نقية) مع **ملفات الاختبار الملتزمة نفسها** بلا أي
تعديل، ونُفِّذت فعليًا:

1. `kotlinc` 2.4.21 (من حزمة `kotlin-compiler` على npm) مقابل OpenJDK 25، هدف JVM 17.
2. شُغِّلت الاختبارات عبر مشغّل انعكاسي صغير خارج المستودع (غير ملتزم) مع بديل محلي لواجهة
   JUnit 4 (`org.junit.Test`, `org.junit.Before`, `org.junit.Assert`) لأن ملف `junit.jar` غير متاح.
   **أجساد الاختبارات هي الملتزمة نفسها** — لم تُعدَّل لتلائم البديل.

الأمر الكامل والمخرجات محفوظة حرفيًا في `patches/verification-log.txt`.

### النتيجة

```
== core.buildcheck.FailureClassifierTest     (9 tests) ->  9 PASS
== core.buildcheck.ProjectDetectorTest      (12 tests) -> 12 PASS
== core.buildcheck.TestReportParserTest     (18 tests) -> 18 PASS
== core.capability.CapabilityRegistryTest   (15 tests) -> 15 PASS
== core.capability.CapabilityValidatorTest  (16 tests) -> 16 PASS
== core.memory.ProjectMemoryTest            (15 tests) -> 15 PASS
== core.policy.CommandInspectorTest         (12 tests) -> 12 PASS
== core.policy.PolicyAuditFileTest          (22 tests) -> 22 PASS
== core.policy.PolicyAuditLogTest            (4 tests) ->  4 PASS
== core.policy.RiskEngineTest               (23 tests) -> 23 PASS
== core.policy.ToolCallMapperTest           (11 tests) -> 11 PASS
== core.security.SecretRedactionTest        (15 tests) -> 15 PASS   (14 قائمة + 1 جديد)
== core.task.TaskProgressCodecTest          (17 tests) -> 17 PASS
== core.task.TaskReportBuilderTest          (15 tests) -> 15 PASS
== core.task.TaskResumeTest                 (13 tests) -> 13 PASS
== core.task.TaskRunnerTest                 (18 tests) -> 18 PASS
== core.task.TaskStateMachineTest           (11 tests) -> 11 PASS

RESULT passed=246 failed=0
```

المصادر في التشغيل الأخير مأخوذة من **نسخة استنساخ نظيفة طُبِّقت عليها الـ patches العشرة بـ `git am`** (لا
من شجرة عمل معدّلة يدويًا)، أي أن ما تم تشغيله هو بالضبط ما سيصل إلى المستودع.

الشيفرة تُترجم بلا أخطاء ولا تحذيرات، والاختبارات الـ 14 القائمة لـ`SecretRedaction` تمر بعد إضافة النمط
الجديد — أي لا انحدار في الملف المعدَّل الوحيد من الشيفرة القائمة.

### أخطاء اكتُشفت بالتشغيل وأُصلحت (بدل الادعاء بأنها كانت صحيحة)

| الخطأ | السبب | الإصلاح |
| --- | --- | --- |
| `./gradlew :app:assembleRelease` لم يُصنَّف كإصدار | كان سطر الأمر يُحوَّل إلى أحرف صغيرة قبل المطابقة، والنمط مكتوب `assembleRelease` | النمط صار بأحرف صغيرة مع تعليق يوضّح العقد |
| تفاصيل سبب قد تحمل سرًا بصيغة `--password value` | `SecretRedaction` يغطي `key=value` و`Bearer` فقط | أُضيف نمط لصيغة أعلام سطر الأوامر + اختبار له |
| `writeFile` صُنِّف `UNKNOWN` بدل `WRITE_FILE` | فصل camelCase كان يستبدل المطابقة كلها بمسافة فتصبح `write ile` | الاستبدال صار `"$1 $2"` مع تعليق يشرح الخطأ |
| قدرة من مصدر غير موثوق **لا يمكن تفعيلها أبدًا** حتى بعد المراجعة | كان `UNTRUSTED_SOURCE` يُسجَّل كعيب حاجب، فتعارض مع بوابة المراجعة في `enable()` | أُضيف `blocking` للعيب: المصدر غير الموثوق يُبلَّغ عنه ولا يمنع الانتقال؛ المنع في `enable()` |
| معرّفات القدرات العميقة (`a.b.c.d.e.x`) كانت مقبولة | النمط سمح بخمسة مستويات | حُدّ النمط إلى أربعة مقاطع كحد أقصى |
| حزمة `core/build` **لم تُسجَّل في Git إطلاقًا** | `.gitignore` في المستودع يتجاهل `build/` فأسقط مجلد الحزمة من `git add` | أُعيدت التسمية إلى `core/buildcheck`، وأُضيف سبب التسمية إلى `docs/BUILD_CHECKS.md` حتى لا يتكرر الفخ |
| كل خطوة لم تبدأ أو لم تنته بعد تُرفض عند قراءة المهمة المحفوظة | `optionalLong` كانت تُعيد `null` للعمود الفارغ، و`?: return Rejected(...)` عند موقع الاستدعاء تلتبس بين «غائب» و«غير صالح» | فُصلت الحالة صراحة: عمود فارغ = `null`، وغير الفارغ غير الرقمي = رفض مسمّى |
| رسالة فشل تحوي `|` أو سطرًا جديدًا **تُفسد الملف المحفوظ** | عمود الفشل كان يُمرَّر على `SecretRedaction` فقط دون `escape` | أُضيف `escape` بعد التنقيح، واختبار برسالة تحوي الأنواع الثلاثة |
| اختبار تقرير الاختبارات توقّع اختبارين من سطر واحد | بيانات الاختبار نفسها كانت ناقصة (سطر واحد لاختبار ناجح لم يُكتب) | أُضيف السطر الناقص؛ وكشف التشغيل أن الفارق بين المعلن والمقروء يُعلَن بـ`countsDisagree` لا يُخفى |

### فحوص تنسيق أُجريت يدويًا (لأن spotless غير متاح)

سكربت فحص على الملفات المتأثرة: طول السطر ≤ 140، لا مسافات زائدة في نهاية السطر، لا تبويبات،
سطر جديد في نهاية الملف → `files checked=45 issues=0` على كل ملفات الحزم الجديدة. الملفات تتبع أسلوب المستودع (تعليقات KDoc
تشرح السبب، أسماء اختبارات بين backticks، فواصل لاحقة trailing commas كما في بقية المشروع).

## فحص تطبيق الـ patches على نسخة نظيفة

```
$ git clone --depth 1 https://github.com/yuga-hashimoto/and-code.git /tmp/applycheck2
$ git am patches/0001-*.patch ... patches/0010-*.patch
Applying: feat(policy): add an in-app risk engine and task state machine
Applying: feat(task): add resume, confirmation and cancellation rules, plus the tool-call mapper
Applying: feat(capability): add a validated capability registry with a lifecycle
Applying: feat(memory): add a project memory that refuses secrets and mistrusts summaries
Applying: feat(buildcheck): detect the project type and plan bounded, classified checks
Applying: feat(task): build the end-of-task report with a verdict that cannot be talked up
Applying: feat(task): add a runner that drives plan, policy, checks and report
Applying: feat(task): add a strict, dependency-free format for a saved task
Applying: feat(buildcheck): read per-test results out of a check's output
Applying: feat(policy): save the audit trail in a versioned, append-friendly file format
$ git log --oneline -11
d1b7394 feat(policy): save the audit trail in a versioned, append-friendly file format
ae1719c feat(buildcheck): read per-test results out of a check's output
8b2664e feat(task): add a strict, dependency-free format for a saved task
cbacc56 feat(task): add a runner that drives plan, policy, checks and report
48d408a feat(task): build the end-of-task report with a verdict that cannot be talked up
56030ec feat(buildcheck): detect the project type and plan bounded, classified checks
899401b feat(memory): add a project memory that refuses secrets and mistrusts summaries
e227757 feat(capability): add a validated capability registry with a lifecycle
4382803 feat(task): add resume, confirmation and cancellation rules, plus the tool-call mapper
46a357f feat(policy): add an in-app risk engine and task state machine
414c780 Merge pull request #404 from yuga-hashimoto/feat/402-edit-local-workspace-files
```

أي أن الـ patches العشرة تُطبَّق بالترتيب وبلا تعارض على `main` الحالي (`414c780`)، وتشغيل الاختبارات
أعلاه جرى على هذه النسخة بالضبط (45 ملفًا فُحص تنسيقيًا: لا سطر > 140، لا مسافات زائدة).

## ما لم يُشغَّل (وهذا مذكور كقيد، لا كنجاح)

- `./gradlew detekt spotlessCheck` — لم يُشغَّل.
- `./gradlew :app:testGithubDebugUnitTest` بالكامل — لم يُشغَّل (1,478 اختبارًا قائمًا لم تُعَد).
- `./gradlew :app:lintGithubDebug`, `assembleGithubDebug`, `assembleGithubRelease` — لم تُشغَّل.
- أي اختبار على محاكٍ أو جهاز (`androidTest`) — لم يُشغَّل.
- لم يُبنَ أي APK، ولم يُنشر أي إصدار، ولم يُدفع أي شيء إلى مستودع بعيد.

## كيف يعيد المستخدم/المراجع التحقق

```bash
git clone https://github.com/yuga-hashimoto/and-code.git
cd and-code
git checkout -b feat/policy-risk-engine
git am /path/to/patches/0001-feat-policy-risk-engine-and-task-state-machine.patch

./gradlew :app:testGithubDebugUnitTest --tests 'com.yugahashimoto.andcode.core.policy.*'
./gradlew :app:testGithubDebugUnitTest --tests 'com.yugahashimoto.andcode.core.task.*'
./gradlew detekt spotlessCheck
```

هذه الأوامر لم تُشغَّل في هذه الجلسة، لذلك نتيجتها عندك هي المصدر الوحيد للحكم على التوافق مع
detekt/spotless وخط CI الفعلي.
