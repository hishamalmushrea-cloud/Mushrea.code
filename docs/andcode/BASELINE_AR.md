# تقرير خط الأساس — AndCode (المرحلة A)

**تاريخ الفحص:** 2026-10-25
**المستودع المستهدف:** `https://github.com/yuga-hashimoto/and-code` (عام، MIT، 185 نجمة)
**الفرع المفحوص:** `main`
**الـ commit:** `414c780649cebb80e3815a6f02115145a5bd3c66` — «Merge pull request #404 from yuga-hashimoto/feat/402-edit-local-workspace-files» بتاريخ 2026-10-25 22:48 +0900
**إصدار التطبيق في المصدر:** `v1.2.27` (ملف `.release-version`)

> ملاحظة عن مساحة العمل: المستودع `Mushrea.code` الذي تعمل فيه هذه الجلسة يحتوي على ملف المهمة فقط
> (`AndCode_AI_Evolution_Platform_Prompt_AR.md`) ولا يحتوي شيفرة AndCode. لذلك تم استنساخ المستودع
> المستهدف فعليًا (`git clone --depth 50`) والفحص والتعديل جرى على نسخة حقيقية منه. لا توجد صلاحية
> دفع إلى المستودع الأصلي: `permissions.push = false` بحسب GitHub API، لذلك سُلّم العمل كـ patch في
> `patches/` بدلًا من فرع مرفوع.

---

## 1. البنية الفعلية

البنية أحادية الوحدة تقريبًا: وحدتان في Gradle (`settings.gradle.kts`).

| الوحدة | المحتوى |
| --- | --- |
| `:app` | التطبيق كاملًا (Kotlin + Jetpack Compose) |
| `:benchmark` | وحدة baseline profile |

**حجم الشيفرة (مُقاس فعليًا):**

| العنصر | العدد |
| --- | --- |
| ملفات `.kt` في `app/src/main` | 495 |
| أسطر شيفرة الإنتاج | 68,096 |
| ملفات اختبارات الوحدة | 179 |
| أسطر اختبارات الوحدة | 29,325 |
| عدد `@Test` | 1,478 |
| ملفات `androidTest` | 12 |
| إجمالي ملفات المستودع (بدون `.git`) | 724 |
| حجم النسخة (بدون `.git`) | 14 MB |

**الحزم (`app/src/main/java/com/yugahashimoto/andcode`):**

- `core/`: `api`, `diagnostics`, `lifecycle`, `locale`, `notification`, `runtime`, `security`, `storage`, `util`
- `data/`: `connection`, `local`, `repository`, `schedule`, `settings`
- `runtime/`: `local` (91 ملفًا، أكبر منطقة في المشروع)، `remote`
- `feature/`: `activity`, `assistant`, `browser`, `chat`, `onboarding`, `schedule`, `settings`, `support`, `wakeword`, `widget`, `workspace`
- `ui/`: `components`, `navigation`, `theme`؛ إضافة إلى `di/`, `startup/`, `AndCodeApplication.kt`

**إعداد البناء (كما هو، لم يُغيَّر):**

- Android Gradle Plugin 8.5.2، Kotlin 2.0.21، KSP 2.0.21-1.0.28، Gradle 8.9 (wrapper).
- `compileSdk = 35`, `targetSdk = 35`, `minSdk = 26`.
- نكهتان: `github` (مع Firebase/Crashlytics) و`fdroid` (بدون Firebase).
- detekt 1.23.6 + spotless 6.25.0 (ktlint 1.2.1) + jacoco. أقصى طول سطر 140، وقاعدة `build.maxIssues: 0`.
- اختبارات الوحدة: JUnit 4.13.2 + `mockwebserver` + `kotlinx-coroutines-test`.

**CI (8 ملفات في `.github/workflows`):** `android.yml` (مهام: `static-analysis` = detekt + spotlessCheck،
`lint` = `lintGithubDebug`، `test-and-build` = `testGithubDebugUnitTest` + `assembleGithubDebug` +
`assembleGithubDebugAndroidTest` + `assembleGithubRelease`)، و`commitlint.yml`, `pr-apk-build.yml`,
`release.yml`, `fdroid-repository.yml`, `i18n-check.yml`, `native-source-libs.yml`, `ocr-review.yml`.

---

## 2. الميزات الموجودة بالفعل (متحقَّق منها في المصدر)

- **تشغيل الوكلاء على الجهاز:** OpenCode، Claude Code، Codex، Antigravity داخل PRoot/Alpine أو Debian
  rootfs (`runtime/local/`، 91 ملفًا؛ منها `LocalRuntimeInstaller`, `LocalRuntimeService`,
  `ClaudeCodeRuntime`, `CodexRuntime`, `AntigravityRuntime`, `DebianRootfsInstaller`).
- **اتصال بعيد:** OpenCode عبر HTTP/SSE (`core/api/OpenCodeV2ApiClient.kt`, `runtime/remote/`)،
  وLAN discovery (`feature/workspace/LanDiscovery.kt`)، وQR للاتصال (`core/security/ConnectionQrPayload.kt`).
- **الجلسات والمحادثة والبث الحي:** `feature/chat/ChatViewModel.kt` (134 KB)، ترجمة أحداث SSE،
  عرض الأدوات والتفكير، قائمة TODO التي يبلّغ عنها الوكيل (`OpenCodeTodo`, `TodoItem`).
- **الموافقات الحالية:** `PermissionRequest` / `PermissionResponse(ONCE, ALWAYS, REJECT)`،
  `ClaudePermissionBridge` (جسر ملفات بين hook في الضيف وواجهة الموافقة)، `ClaudePermissionMode`,
  `AntigravityPermissionMode`. أي أن الموافقة **تُمرَّر** من الـ CLI إلى المستخدم، ولا يصنّفها التطبيق.
- **مساحة العمل والمحرر:** `feature/workspace/*` (شجرة ملفات، عارض شيفرة، تعديل ملفات في مساحات
  محلية عبر `WorkspaceFileWriter`)، Git status/diff، طرفية مضمّنة.
- **المهام المجدولة:** `feature/schedule/*` (cron، تاريخ تشغيل، `ScheduleRetryPolicy`,
  `ScheduleRunWatchdog`, تنفيذ في خدمة أمامية).
- **الصوت والمساعد:** wake word، TTS، مساعد النظام، ودجت الشاشة الرئيسية، `feature/browser/`.
- **التخزين الآمن:** `data/connection/SecureSettingsRepository.kt` (Android Keystore)،
  `core/security/SecretRedaction.kt`، ومخازن بيانات اعتماد الوكلاء داخل rootfs
  (`/root/.config/opencode/auth.json`، `/root/.gemini`، `CODEX_HOME=/root/.codex`، `/root/.git-credentials`).
- **التعريب:** 8 لغات في `res/values-*` بما فيها `values-ar`، وسياسة RTL قائمة.

---

## 3. نتائج البناء والاختبارات قبل التعديل

**لم تُشغَّل.** بيئة هذه الجلسة لا تحتوي JDK ولا Android SDK ولا Gradle، والشبكة مسموحة إلى
`github.com` / `registry.npmjs.org` / `pypi.org` فقط، أي أن Maven Central و`dl.google.com` غير متاحين،
فتعذّر تنزيل Gradle وتوابع AGP. لذلك **لا يوجد خط أساس لنتائج `./gradlew test` أو `assemble`**، وأي رقم
بهذا الخصوص كان سيكون ادعاءً غير مثبت.

ما أمكن تشغيله فعلًا: تجميع وتنفيذ الشيفرة الجديدة (Kotlin نقية، بلا Android) بعد التعديل — التفاصيل في
`VERIFICATION_AR.md` وفي `patches/verification-log.txt`.

---

## 4. الفجوات مقابل الرؤية المطلوبة

| # | المطلوب في المهمة | الحالة الفعلية | الفجوة |
| --- | --- | --- | --- |
| 1 | دورة مهمة بحالات صريحة (تخطيط → تنفيذ → تحقق → مراجعة) | لا توجد حالات مهمة في التطبيق؛ الموجود قائمة TODO يبلّغ عنها الوكيل، وجلسة محادثة | **فجوة كاملة** — أُنجز أساسها في هذه الجلسة (`core/task`) |
| 2 | محرك مخاطر مستقل عن النموذج | لا يوجد. الموافقة تُمرَّر من الـ CLI، و`PermissionMode` يقرره المستخدم لكن لا تصنيف للتشغيل | **فجوة كاملة** — أُنجز أساسها في هذه الجلسة (`core/policy`) |
| 3 | سجل تدقيق للقرارات | لا يوجد أي `audit` في المصدر قبل هذا التغيير | أُنجز سجل محدود في الذاكرة، غير مثبَّت (non-persistent) |
| 4 | نظام قدرات بعقد ومخطط وإصدارات | `runtime/RuntimeCapabilities.kt` مجرد أعلام قدرات لكل backend (بيانات منطقية)؛ لا سجل قدرات، ولا مخطط، ولا دورة حياة | فجوة قائمة |
| 5 | ذاكرة هندسية للمشروع | `AndCodeAgentContext` يكتب ملفات سياق للوكيل، وليس ذاكرة هندسية مرتبطة بـ commit | فجوة قائمة |
| 6 | اكتشاف نوع المشروع وتشغيل البناء/الاختبارات | طرفية مضمّنة + جدولة أوامر؛ لا كشف لمشروع ولا فحص محدود الموارد ومصنَّف | فجوة قائمة |
| 7 | استئناف/إلغاء فعلي للمهام | الطرفية والجداول لها إلغاء/watchdog؛ لا شيء على مستوى «مهمة تطوير» | فجوة قائمة (الأساس ممكّن الآن) |
| 8 | حدود APK مقابل قدرات وصفية | غير موثق في التطبيق | فجوة قائمة (موثّقة في `docs/POLICY_ENGINE.md` للتغيير الحالي) |
| 9 | وكيل متصفح لاختبار الويب | `feature/browser/BrowserCommandWatcher` موجود لفتح روابط؛ لا اختبار واجهات | فجوة قائمة |
| 10 | أدوات USB/ADB | `runtime/local/AdbConnectionManager.kt` موجود للاتصال؛ لا عمليات جهاز | فجوة قائمة |

---

## 5. المخاطر والتبعيات والقيود

1. **PRoot ليس عزلًا أمنيًا كاملًا.** أوامر الضيف تعمل بصلاحيات التطبيق نفسها على الجهاز؛ لا يمكن
   الاعتماد عليه كحد أمني وحده.
2. **بيانات اعتماد الوكلاء داخل rootfs** (`auth.json`, `.gemini`, `.codex`, `.git-credentials`).
   أي قدرة تقرأ ملفات بحرية يمكنها الوصول إليها؛ لذلك صُنِّفت هذه المسارات في المحرك الجديد كأسرار.
3. **الموافقة اليوم «تمريرية»**: التطبيق يعرض ما يطلبه الـ CLI؛ لا يوجد تصنيف مستقل. هذا هو السبب
   الجذري لأولوية P2 في المهمة.
4. **حجم الواجهة:** `ChatViewModel.kt` 134 KB و`ChatHomeScreen.kt` 112 KB و`AndCodeApp.kt` 93 KB —
   أي ربط جديد بالواجهة يجب أن يكون تدخّلًا جراحيًا ومحدودًا.
5. **لا بناء في هذه البيئة** لغياب JDK/SDK/الشبكة إلى مستودعات Maven؛ لذلك لم يُمسّ أي ملف Android
   في هذا التغيير (كل الشيفرة المضافة Kotlin نقية بلا `android.*`).
6. **نكهتان (`github`/`fdroid`)**: أي إضافة يجب ألا تُدخل Firebase أو خدمات مدفوعة في `fdroid`.
7. **قاعدة `AGENTS.md` في المستودع** تطلب إنشاء worktree عبر `scripts/new-worktree.sh` قبل أي تعديل.
   لم أستطع اتباعها حرفيًا لأن العمل جرى على نسخة استنساخ منفصلة أصلًا (لا worktree ثانٍ ولا صلاحية دفع)،
   وهذا مذكور بصراحة هنا بدل الادعاء بالالتزام.

---

## 6. قائمة ما يجب الحفاظ عليه (لا يُحذف ولا يُعاد بناؤه)

- المسارات الأربعة للوكلاء (OpenCode / Claude Code / Codex / Antigravity) وسلوكيات `RuntimeCapabilities`.
- الموافقات الحالية: `PermissionRequest`، `ClaudePermissionBridge`، `PermissionResponse`، وأوضاع الصلاحيات.
- `SecureSettingsRepository` وتخزين الاعتماد داخل rootfs — لا نقل أسرار إلى تفضيلات Android.
- الجلسات، الحفظ، الاستئناف، المهام المجدولة و`ScheduleRetryPolicy`/`ScheduleRunWatchdog`.
- التعريب الثمانية واتجاه RTL، وتسجيل الدخول/الاستئناف (`BEFORE`-style lifecycle) للـ runtime.
- نكهة `fdroid` خالية من Firebase، وعدم كسر `assembleGithubRelease`.
- `SecretRedaction` وكيل تدقيق الأسرار القائم (`CrashReportSanitizer`).
