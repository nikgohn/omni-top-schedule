# Security Review — Omni TOP (web/Capacitor MVP)

Статический разбор этого репозитория (`omni-top-schedule`) и связанного нативного APK
`2.0-native`. Находки привязаны к конкретным файлам. Детальный разбор APK — см.
[REVIEW.md](REVIEW.md).

## 🔴 Критично (нативная сборка APK)

1. **Release-APK подписан debug-ключом** (`CN=Android Debug, O=Android, C=US`, только v2).
   Google Play не примет; при утечке debug-keystore возможна подмена обновлений.
   → release-keystore, подписи v2+v3.
2. **Учётные данные в открытом виде + `allowBackup=true`** (нативный код):
   `SharedPreferences("omni_native_session")` с plaintext-паролем и сессией, без
   `dataExtractionRules`. → EncryptedSharedPreferences / Keystore; `allowBackup=false`.

## 🟠 Средне (этот репозиторий)

3. **Capacitor: http + cleartext** — `capacitor.config.json`:
   `server.androidScheme: "http"`, `server.cleartext: true`. Контент WebView грузится по
   http, разрешён cleartext-трафик. → `https`-scheme, `cleartext:false`, при необходимости
   networkSecurityConfig.

4. **Сессия в `localStorage` в открытом виде** — `public/omniClient.js`:
   ключ `omni_top_session` (plaintext `cookieStr` + `IdLocalHash`), плюс `omni_auth`,
   `omni_user`. Доступна любому выполняющемуся JS, поверх http-оригина.
   → не хранить сессию в localStorage; httpOnly-cookie / нативное хранилище; как минимум
   не логировать и не дублировать чувствительные поля.

5. **Рендер данных портала через `innerHTML`** — `public/app.js`
   (ветки, студенты, карточки, `container.innerHTML = ...`). Если данные портала содержат
   HTML/JS → XSS, что в связке с п.4 ведёт к угону сессии. → экранировать/санитизировать,
   использовать `textContent`/шаблоны без innerHTML, добавить CSP.

6. **Express: открытый CORS + listen на всех интерфейсах** — `server.js`:
   `app.use(cors())` (Access-Control-Allow-Origin: `*`) и `app.listen(PORT)` без хоста
   (по умолчанию `0.0.0.0`). Сервер проксирует авторизованные запросы к порталу и хранит
   сессию в памяти процесса → любой сайт/хост в LAN может вызывать `/api/*` и пользоваться
   чужой сессией; пароль передаётся в `req.body` по http (нет TLS).
   → ограничить `cors({ origin: ... })`, слушать `127.0.0.1`, добавить TLS или отказаться
   от хранения сессии на сервере, rate-limit на `/api/auth/login`.

7. **Пароль через локальный сервер по http** — `server.js` `/api/auth/login` принимает
   username/password телом по http. → TLS или логин только из клиента напрямую к порталу.

## 🟢 Низко / гигиена

- `server.js`/`omniParser.js`: подробные `console.log`/`console.warn` с username и
  состоянием сессии — убрать/ограничить в проде.
- Нет CSP, нет security-заголовков на Express (helmet).
- `package.json`: нет тестов; `license: ISC` (в репо нет LICENSE-файла). Зависимости:
  express ^5.3.0, capacitor ^8.6.0 — планово обновлять и прогонять `npm audit`.
- `server.js` держит `activeOmniSession` в памяти процесса без таймаута/инвалидации.
- Нативный APK: verbose-логирование (тег `OmniRepo`, username/ФИО), мёртвые
  Capacitor-артефакты в assets, внутренний JSON расписания в assets.

## Что уже хорошо

- Запросы к порталу — по **HTTPS** (`https://omni.top-academy.ru`).
- Корректный CSRF-flow логина (`/login/index` → token → `/auth/login` с `X-CSRF-Token`).
- Поле пароля в UI — `type="password"` (index.html); в нативном APK — маскирование +
  password-keyboard.
- Детект протухшей сессии → повторный логин.

## Быстрый план исправлений

1. Capacitor: `androidScheme:"https"`, `cleartext:false`.
2. Убрать сессию из localStorage; санитизировать рендер (убрать innerHTML), добавить CSP.
3. Express: `cors({ origin: <whitelist> })`, `app.listen(PORT, '127.0.0.1')`, helmet,
   rate-limit, не держать сессию в памяти (или с TTL).
4. Нативная сборка: release-keystore (v2+v3), шифрование учётных данных, `allowBackup=false`,
   вырезать release-логи.
