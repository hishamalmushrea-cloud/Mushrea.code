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
2. شُغِّلت الاختبارات عبر مشغّل انعكاسي صغير في `/tmp` (خارج المستودع، غير ملتزم) مع بديل محلي لواجهة
   JUnit 4 (`org.junit.Test`, `org.junit.Before`, `org.junit.Assert`) لأن ملف `junit.jar` غير متاح.
   **أجساد الاختبارات هي الملتزمة نفسها** — لم تُعدَّل لتلائم البديل.

الأمر الكامل والمخرجات محفوظة حرفيًا في `patches/verification-log.txt`.

### النتيجة

```
== com.yugahashimoto.andcode.core.policy.CommandInspectorTest (12 tests)   -> 12 PASS
== com.yugahashimoto.andcode.core.policy.PolicyAuditLogTest (4 tests)      -> 4 PASS
== com.yugahashimoto.andcode.core.policy.RiskEngineTest (23 tests)         -> 23 PASS
== com.yugahashimoto.andcode.core.security.SecretRedactionTest (15 tests)  -> 15 PASS   (14 قائمة + 1 جديد)
== com.yugahashimoto.andcode.core.task.TaskStateMachineTest (11 tests)     -> 11 PASS

RESULT passed=65 failed=0
```

الشيفرة تُترجم بلا أخطاء ولا تحذيرات، والاختبارات الـ 14 القائمة لـ`SecretRedaction` تمر بعد إضافة النمط
الجديد — أي لا انحدار في الملف المعدَّل الوحيد من الشيفرة القائمة.

### أخطاء اكتُشفت بالتشغيل وأُصلحت (بدل الادعاء بأنها كانت صحيحة)

| الخطأ | السبب | الإصلاح |
| --- | --- | --- |
| `./gradlew :app:assembleRelease` لم يُصنَّف كإصدار | كان سطر الأمر يُحوَّل إلى أحرف صغيرة قبل المطابقة، والنمط مكتوب `assembleRelease` | النمط صار بأحرف صغيرة مع تعليق يوضّح العقد |
| تفاصيل سبب قد تحمل سرًا بصيغة `--password value` | `SecretRedaction` يغطي `key=value` و`Bearer` فقط | أُضيف نمط لصيغة أعلام سطر الأوامر + اختبار له |

### فحوص تنسيق أُجريت يدويًا (لأن spotless غير متاح)

سكربت فحص على الملفات المتأثرة: طول السطر ≤ 140، لا مسافات زائدة في نهاية السطر، لا تبويبات،
سطر جديد في نهاية الملف → `files checked=15 issues=0`. الملفات تتبع أسلوب المستودع (تعليقات KDoc
تشرح السبب، أسماء اختبارات بين backticks، فواصل لاحقة trailing commas كما في بقية المشروع).

## فحص تطبيق الـ patch على نسخة نظيفة

```
$ git clone --depth 1 https://github.com/yuga-hashimoto/and-code.git /tmp/applycheck
$ git am patches/0001-feat-policy-risk-engine-and-task-state-machine.patch
Applying: feat(policy): add an in-app risk engine and task state machine
$ git log --oneline -2
d6ffb50 feat(policy): add an in-app risk engine and task state machine
414c780 Merge pull request #404 from yuga-hashimoto/feat/402-edit-local-workspace-files
```

أي أن الـ patch يُطبَّق بلا تعارض على `main` الحالي (`414c780`)، والملفات الثمانية الجديدة موجودة بعد
التطبيق.

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
