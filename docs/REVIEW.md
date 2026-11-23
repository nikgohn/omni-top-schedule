# Code Review — Omni TOP (Android)

Ревью нативного Android-приложения из этого репозитория (`android/`, package
`ru.topacademy.omni`, версия `2.0-native`) и связанного web-MVP (корень репо).
Ниже — находки по убыванию критичности, с привязкой к файлам. Сводка по
безопасности — в [SECURITY.md](SECURITY.md).

---

## Резюме

Кодовая база UI/доменного слоя написана аккуратно (MVVM на StateFlow, разделение
на компоненты, осмысленный WorkManager). Однако в текущем виде репозиторий
**не собирается**: часть слоя данных не закоммичена. Отдельно — release собирается
debug-ключом и тащит в прод OkHttp logging-interceptor. Это нужно закрыть до любого
распространения APK.

---

## 🔴 Блокирующие

### 1. Репозиторий не компилируется — не закоммичен слой данных

В `main` отсутствуют классы, на которые ссылается код:

| Класс (ожидаемый) | Где используется |
|---|---|
| `ru.topacademy.omni.data.repository.OmniRepository` | `ui/MainViewModel.kt:15`, `worker/ScheduleSyncWorker.kt:12` |
| `ru.topacademy.omni.data.parser.OmniScheduleParser` | `ui/MainViewModel.kt:14` |
| `ru.topacademy.omni.data.db.OmniDatabase` | `widget/ScheduleGlanceWidget.kt:37` |

README (раздел «Архитектура») и `app/proguard-rules.pro` дополнительно упоминают
`data/local/dao`, `data/local/entity`, `data/network`, `data/session` — их в дереве
тоже нет. Без них `:app:assembleRelease` падает, а значит CI
`.github/workflows/build.yml` всегда красный.

Также несогласованность пакетов: код импортирует `data.db.*`, а proguard-правила
написаны под `data.local.dao.*` / `data.local.entity.*`.

**Что сделать:** закоммитить недостающие пакеты `data/` (network/parser/repository/
session/db), привести имена пакетов к одному виду (код vs proguard), прогнать
`./gradlew :app:assembleRelease` локально и в CI.

### 2. Release собирается debug-ключом

`android/app/build.gradle`:
```gradle
buildTypes {
    release {
        signingConfig signingConfigs.debug   // ← debug-ключ в релизе
        minifyEnabled true
        shrinkResources true
    }
}
```
Итоговый APK подписан `CN=Android Debug` (только v2). Google Play такую сборку не
примет; при утечке debug-keystore возможна подмена обновлений.

**Что сделать:** завести release-keystore, читать его из `keystore.properties`
(не коммитить), включить v2+v3:
```gradle
// app/build.gradle
def ks = rootProject.file("keystore.properties")
def ksProps = new Properties()
if (ks.exists()) { ksProps.load(new FileInputStream(ks)) }

signingConfigs {
    release {
        storeFile ks.exists() ? file(ksProps['storeFile']) : null
        storePassword ksProps['storePassword']
        keyAlias ksProps['keyAlias']
        keyPassword ksProps['keyPassword']
    }
}
buildTypes { release { signingConfig signingConfigs.release } }
```
`keystore.properties` и `*.jks` — в `.gitignore`.

---

## 🟠 Безопасность

### 3. OkHttp logging-interceptor попадает в release

`app/build.gradle`: `implementation 'com.squareup.okhttp3:logging-interceptor:4.12.0'`
(не `debugImplementation`). Если интерцептор подключён к клиенту, в release в logcat
утекают заголовки/тела запросов — включая `Cookie`, `X-CSRF-Token` и логин/пароль.

**Что сделать:** заменить на `debugImplementation`, а в коде подключать только при
`BuildConfig.DEBUG`.

### 4. `allowBackup="true"` без правил

`AndroidManifest.xml:5` — `android:allowBackup="true"`, нет `dataExtractionRules` /
`fullBackupContent`. Локальная БД и настройки (в т.ч. сессия) попадают в бэкап.
→ `allowBackup="false"` либо правила с исключением БД и SharedPreferences сессии.

### 5. `LessonDialogActivity` экспортирована без необходимости

`AndroidManifest.xml:33` — `android:exported="true"`. Запускается из виджета через
`PendingIntent` — экспорт не требуется; любое приложение может дёрнуть её с
`EXTRA_LESSON_ID`. → `android:exported="false"`.

### 6. FileProvider со слишком широкими путями

`res/xml/file_paths.xml`: `<external-path path="."/>` и `<cache-path path="."/>`.
При выдаче URI это экспонирует всю внешнюю/кэш-папку. → сузить до конкретных
подкаталогов.

### 7. Web-MVP (корень репо): http + сессия в localStorage + открытый сервер

- `capacitor.config.json`: `server.androidScheme:"http"`, `server.cleartext:true`.
- `public/omniClient.js`: сессия (`cookieStr`, `IdLocalHash`) в `localStorage`
  (`omni_top_session`) — доступна любому JS поверх http-оригина.
- `public/app.js`: рендер данных портала через `innerHTML` → риск XSS и угона сессии.
- `server.js`: `app.use(cors())` (origin `*`) и `app.listen(PORT)` без хоста
  (`0.0.0.0`); сессия в памяти; пароль телом по http.

Детали и фиксы — в [SECURITY.md](SECURITY.md).

---

## 🟢 Качество / гигиена

- **Нет тестов**, хотя в зависимостях junit/espresso. Хотя бы unit на
  `OmniScheduleParser` и `LessonStateCalculator`.
- **UI-строки захардкожены** в Compose (рус.), нет локализации; вынести в
  `strings.xml` + `localeConfig`.
- `android/variables.gradle`: `minSdkVersion 24` против `minSdk 26` в
  `app/build.gradle` — рассинхрон (Capacitor-остаток).
- `android/app/capacitor.build.gradle` требует Java 21, тогда как app — Java 17;
  файл, судя по всему, не применяется (в `app/build.gradle` нет `apply from`) —
  мёртвый leftover вместе с `capacitor.config.json`/`server.js`/`-public/` на корню.
  Стоит решить: либо web-MVP в отдельную папку/репо, либо удалить из Android-ветки.
- Нет crash-reporting (Crashlytics/Sentry); сохранять mapping.txt от R8.
- Зависимости местами устарели (Compose BOM 2024.10.01, WorkManager 2.9.1 и т.д.) —
  планово обновить.

---

## ✅ Что сделано хорошо

- Чистый MVVM: `MainViewModel` + `MainUiState` на `StateFlow`, отмена `Job` при
  пересоздании подписок, корректный `logout()` со сбросом состояния.
- Вдумчивый `ScheduleSyncWorker`: `NetworkType.CONNECTED` + `setRequiresBatteryNotLow`,
  тихие ночные часы (23:00–07:00), 2-часовой интервал, `ExistingPeriodicWorkPolicy.UPDATE`,
  без агрессивных ретраев.
- Доменные модели — аккуратные data-классы с комментариями (`Models.kt`).
- R8: `minifyEnabled` + `shrinkResources` + проработанный `proguard-rules.pro`.
- Минимальные разрешения (только `INTERNET`), нативный Compose без WebView.
- Целевой размер APK ~3 МБ.

---

## Методология

Статический анализ исходников `main` (android/), `app/build.gradle`, манифеста,
proguard, CI-workflow, web-MVP (server.js, public/, capacitor.config.json); сверка с
собранным APK `2.0-native` (apktool/jadx/apksig) для подтверждения подписи и состава.
