> Примечание: этот репозиторий (omni-top-schedule) — web/Capacitor-исходник (MVP).
> Ниже — ревью нативного APK 2.0-native (ru.topacademy.omni), собранного на основе
> этого проекта. Общие находки по безопасности см. в SECURITY.md.

---

# Ревью APK: omni-top-release (2).apk

**Дата:** 2026-11-23
**Файл:** `C:\Users\Sharman\Downloads\Telegram Desktop\omni-top-release (2).apk` (3 072 329 байт)

## 1. Паспорт сборки

| Параметр | Значение |
|---|---|
| Package | `ru.topacademy.omni` |
| Название | Omni TOP |
| versionCode / versionName | 2 / `2.0-native` |
| minSdk / targetSdk / compileSdk | 26 / 36 / 36 |
| AGP / Gradle / Kotlin | 8.13.0 / 8.14.3 / 2.0.21 |
| UI | Jetpack Compose (UI 1.7.5, Material3 1.3.1) — **нативный, без WebView** |
| Сеть | OkHttp 4.12.0 + Jsoup + Gson |
| БД | Room 2.6.1 (`omni_native.db`) |
| Фон | WorkManager 2.9.1, Glance 1.1.1 (виджет) |
| Подпись | **v2 only, сертификат `CN=Android Debug, O=Android, C=US`** (debug-ключ, создан 2026-10-29) |
| Обфускация | R8 (агрессивная), baseline-profiles присутствуют |
| ABI | arm64-v8a, armeabi-v7a, x86, x86_64 (только `libandroidx.graphics.path.so`) |

Приложение — нативный клиент преподавательского портала `https://omni.top-academy.ru`:
логин по корпоративным учётным данным, расписание, задачи, «презенты», виджет расписания,
фоновая синхронизация каждые 2 часа.

## 2. Критические замечания

### 🔴 C1. «Release» подписан debug-ключом
`ApkVerifier`: verified=true, v1=false, **v2=true**, v3=false.
Сертификат: `C=US, O=Android, CN=Android Debug`, serial=1, SHA256withRSA,
notBefore=2026-10-29, notAfter=2056-10-21.

Последствия:
- Google Play **не примет** такую сборку; для商店/распространения нужен release-keystore.
- Debug-ключ считается небезопасным: если keystore утёк/общий (CI, репозиторий), любой может
  подписать обновление с тем же package и установить поверх приложения → перехват учётных данных.
- Нет v3 → невозможна ротация ключа.

**Fix:** настроить `signingConfigs.release` (отдельный keystore, пароль не в коде),
включить v1+v2+v3 (v4 по желанию), пересобрать. Debug-ключ удалить из release-пайплайна.

### 🟠 C2. Пароль и сессия хранятся в открытом виде + включён backup
`OmniSessionManager` → `SharedPreferences("omni_native_session", MODE_PRIVATE)`:
- `key_saved_username`
- `key_saved_password` — **пароль в plaintext**
- `key_user_session_json` — JSON с `cookieStr`, `idLocalHash`, `teacherName`, `username`, `branches`.

В манифесте: `android:allowBackup="true"`, **нет** `dataExtractionRules` / `fullBackupContent` /
`networkSecurityConfig`. На targetSdk 36 автобэкап включит эти prefs → учётные данные
преподавателя можно извлечь через backup / root / adb.

**Fix:**
- `EncryptedSharedPreferences` (Jetpack Security) или Android Keystore для пароля;
  лучше вообще не хранить пароль, а держать только сессию + refresh-механизм.
- `android:allowBackup="false"` либо `dataExtractionRules`/`fullBackupContent` с исключением
  `omni_native_session.xml` и `omni_native.db`.
- Рассмотреть `FLAG_SECURE` на экранах с чувствительными данными (опционально).

## 3. Средние замечания

### 🟡 M1. Логирование PII в release
В репозитории (`c7.u`, тег `OmniRepo`) ~20 вызовов `android.util.Log.i/d/w/e` без guards:
- `"Starting implicit relogin for " + username`
- `"Implicit relogin succeeded for " + username`
- `"Session expired during fetchTasks/fetchSchedule..."`, ошибки сети.

Логи не вырезаны R8 → unconditional в release. Username/ФИО и внутреннее состояние утекают в logcat.

**Fix:** убрать логи из release (R8 `-assumenosideeffects class android.util.Log { ... }`
или обёртка с `BuildConfig.DEBUG`), не логировать username/ФИО.

### 🟡 M2. Мёртвые Capacitor-артефакты + небезопасный leftover-конфиг
В `assets/`:
- `capacitor.config.json`: `webDir: "public"`, `server.androidScheme: "http"`, **`cleartext: true`**,
  включены `CapacitorHttp`/`CapacitorCookies`, SplashScreen.
- `capacitor.plugins.json`: `@capacitor/browser`.
- **`assets/public/` отсутствует**, кода Capacitor (`com.getcapacitor.*`) в dex нет,
  `setJavaScriptEnabled`/`WebView.loadUrl` — 0 вхождений.

То есть конфиг не используется, но декларирует cleartext/http. Если когда-нибудь вернут WebView —
получат небезопасную конфигурацию. Плюс это мусор от миграции web→native.

**Fix:** удалить `capacitor.config.json`, `capacitor.plugins.json` и зависимость Capacitor из сборки.

### 🟡 M3. В assets зашит реальный внутренний JSON расписания (64 КБ, не используется)
`assets/teacher_schedule_all_branches.json`: `originalCityId: 95`, `totalBranches: 3`,
реальные группы (`Б2411`, `Б4411`, `Б2312`…), аудитории (`2 (ссылка YouTube)`),
названия курсов (`Программирование микроконтроллеров BBC Micro:bit` и т.д.).
В коде **0 ссылок** на этот файл → мёртвый груз + распространение внутренних операционных данных.

**Fix:** удалить из assets; если нужен demo/fallback — генерировать синтетические данные.

## 4. Низкие замечания

- 🟢 **L1.** `LessonDialogActivity` `exported="true"` без permission. Запускается из виджета через
  PendingIntent — exported не требуется. Любой app может дёрнуть её с `EXTRA_LESSON_ID`.
  → `exported="false"`.
- 🟢 **L2.** FileProvider paths слишком широкие: `<external-path path="."/>`, `<cache-path path="."/>`.
  Сузить до конкретных подпапок.
- 🟢 **L3.** `MainActivity` `screenOrientation="portrait"` — на Android 16 / планшетах / foldables
  ориентация игнорируется или мешает UX. Рассмотреть adaptive layout.
- 🟢 **L4.** Все UI-строки зашиты в Compose-код (хардкод кириллицы), в `strings.xml` только
  библиотечные строки. Нет локализации, нет `localeConfig`. → вынести в ресурсы.
- 🟢 **L5.** `FOREGROUND_SERVICE` объявлен без типа (`dataSync` и т.п.). Для periodic work не
  критично, но при expedited work на Android 14+ будет crash. Уточнить/добавить тип или убрать.
- 🟢 **L6.** В APK shipped метаданные: `DebugProbesKt.bin`, `kotlin-tooling-metadata.json`,
  `META-INF/*.version` (версии всех androidx-библиотек), `version-control-info.textproto`
  (`NO_SUPPORTED_VCS_FOUND`). Раскрывают стек сборки. → `packagingOptions.resources.excludes`.
- 🟢 **L7.** x86/x86_64 `.so` нужны только для эмулятора. → ABI splits / App Bundle для уменьшения
  размера загрузки.
- 🟢 **L8.** Версии библиотек слегка устарели: Compose 1.7.5 / Material3 1.3.1, Kotlin 2.0.21,
  WorkManager 2.9.1, Room 2.6.1, Glance 1.1.1, DataStore 1.0.0. OkHttp 4.12.0 — актуален.
  → плановое обновление.
- 🟢 **L9.** Нет crash-reporting (Firebase Crashlytics / Sentry не обнаружены). Для release
  рекомендуется; R8-маппинг обязательно сохранять и загружать в репортинг.
- 🟢 **L10.** Интеграция построена на скрейпинге HTML (Jsoup + regex по `user_name`,
  `changeCity(\d+)`, PHP-сериализованным cookie `city_id`), spoofed desktop Chrome UA,
  `X-Requested-With: XMLHttpRequest`. Хрупко: любое изменение вёрстки портала сломает парсинг.
  `discoverBranchesAndProfile` глотает все exceptions и возвращает пустой профиль молча.
  → по возможности официальный API; иначе — устойчивые селекторы + явная обработка ошибок.

## 5. Что сделано хорошо

- ✅ Вся сеть — **HTTPS** (`https://omni.top-academy.ru`), cleartext-URL в коде нет.
- ✅ Логин корректно повторяет web-flow: GET `/login/index` → scraping CSRF-token →
  POST `/auth/login` с `X-CSRF-Token` и cookies.
- ✅ Поле пароля замаскировано (`PasswordVisualTransformation`) + `KeyboardType.Password`.
- ✅ Детект протухшей сессии (302/401/403 или HTML login-страницы) → implicit relogin.
- ✅ Минимальный набор permissions (INTERNET, ACCESS_NETWORK_STATE, WAKE_LOCK,
  RECEIVE_BOOT_COMPLETED, FOREGROUND_SERVICE) — без лишних опасных.
- ✅ `debuggable`/`testOnly` отсутствуют; нативный Compose вместо WebView → нет JS-bridge рисков.
- ✅ R8-обфускация + baseline profiles; компактный APK (~3 МБ).
- ✅ targetSdk 36 (актуальный), minSdk 26 разумный.
- ✅ Виджет на Glance, фоновая синхронизация через WorkManager с constraint CONNECTED.
- ✅ В коде не найдено захардкоженных API-ключей/токенов/секретов.

## 6. Приоритетный план действий

1. **Немедленно:** пересобрать с release-keystore (v2+v3), убрать debug-подпись. (C1)
2. **До распространения:** шифровать учётные данные (EncryptedSharedPreferences/Keystore),
   отключить backup чувствительных файлов или `allowBackup=false`. (C2)
3. **Перед релизом:** вырезать release-логи, удалить Capacitor-артефакты и
   `teacher_schedule_all_branches.json`. (M1–M3)
4. **Планово:** L1–L10 (exported, FileProvider, локализация, обновление библиотек,
   crash-reporting, устойчивость парсинга).

## 7. Методология

- Распаковка APK, `apktool d` (manifest + ресурсы + smali), `jadx` (декомпиляция Java).
- Проверка подписи: `com.android.apksig:apksig:8.13.0` (`ApkVerifier`).
- Статический поиск по smali/dex: URL, секреты, WebView, Log, SharedPreferences,
  PasswordVisualTransformation, exported-компоненты, assets-ссылки.
- Версии библиотек: `META-INF/*.version`, `kotlin-tooling-metadata.json`, строки в dex.
