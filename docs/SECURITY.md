# Security — Omni TOP

Фокусный разбор безопасности. Привязка к файлам `main`. Общий code review — в
[REVIEW.md](REVIEW.md). OWASP MASVS в скобках.

## 🔴 Критично

| # | Файл | Проблема | Фикс |
|---|---|---|---|
| S1 | `android/app/build.gradle` | release подписан `signingConfigs.debug` → APK с `CN=Android Debug` (только v2) | release-keystore из `keystore.properties`, v2+v3 (MASVS-RESILIENCE) |
| S2 | `android/app/build.gradle` | `logging-interceptor` в `implementation` (release) → утечка `Cookie`/CSRF/логина в logcat | `debugImplementation`, подключать только при `BuildConfig.DEBUG` (MASVS-STORAGE) |

## 🟠 Средне

| # | Файл | Проблема | Фикс |
|---|---|---|---|
| S3 | `AndroidManifest.xml:5` | `allowBackup="true"` без `dataExtractionRules` → БД/сессия в бэкапе | `allowBackup="false"` или правила-исключения (MASVS-STORAGE) |
| S4 | `AndroidManifest.xml:33` | `LessonDialogActivity` `exported="true"` без нужды | `exported="false"` (MASVS-PLATFORM) |
| S5 | `res/xml/file_paths.xml` | `<external-path path="."/>` — вся внешняя папка под FileProvider | сузить `path` (MASVS-PLATFORM) |
| S6 | `capacitor.config.json` | `androidScheme:"http"`, `cleartext:true` | `https`-scheme, `cleartext:false`, `networkSecurityConfig` (MASVS-NETWORK) |
| S7 | `public/omniClient.js` | сессия в `localStorage` (`omni_top_session`) plaintext | не хранить в localStorage; httpOnly-cookie/нативное хранилище (MASVS-STORAGE) |
| S8 | `public/app.js` | `innerHTML`-рендер данных портала → XSS → угон сессии (S7) | санитизация/`textContent`, CSP (MASVS-PLATFORM) |
| S9 | `server.js` | `cors()` origin `*` + `app.listen(PORT)` на `0.0.0.0`; сессия в памяти; пароль телом по http | `cors({origin})`, listen `127.0.0.1`, TLS, TTL сессии, rate-limit (MASVS-NETWORK/AUTH) |

## 🟢 Низко

- Подробные `console.log`/`Log` с username и состоянием сессии — вырезать в проде.
- Нет CSP / security-заголовков (helmet) на Express.
- Хранение темы/настроек в plaintext SharedPreferences — ок, но не смешивать с сессией.
- Нет certificate pinning (приемлемо для этого кейса, но отметить).

## Приоритет исправлений

1. S1, S2 — до любой раздачи APK.
2. S3–S5 — конфиг нативного приложения.
3. S6–S9 — web-MVP (или вывод web-части из прод-сборки).

## Что уже хорошо

- Портал дергается только по HTTPS; корректный CSRF-flow логина.
- Поле пароля — `type="password"` (web) / маскирование + password-keyboard (native).
- Минимальные разрешения; нативный UI без WebView (в `2.0-native`).
