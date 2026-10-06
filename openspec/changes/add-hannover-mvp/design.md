# Design

## Context

Кода пока нет. Это первый change: он создаёт монорепозиторий и реализует этап 1 из docs/product/product-brief.md (см. proposal.md — Why). Нагрузка — сотни участников в одном городе. Разработка — один разработчик с AI-агентами по SDD, поэтому важны простота, предсказуемая структура и тестируемость. Хостинг — IONOS (Германия), оператор данных — owner как физлицо. Позже нужен переход SQLite → PostgreSQL без переписывания бизнес-логики.

## Goals / Non-Goals

**Goals:**
- Один репозиторий: specs, контракт API, server, frontend и adminpanel меняются одним PR.
- Один деплой-артефакт: Go-бинарник отдаёт API и оба собранных SPA.
- Архитектура, в которую Telegram Mini App и бот (change `add-telegram-mini-app`) добавляются без переделки.
- Слой данных, который переводится на PostgreSQL без изменения доменной логики.
- Контракт API — единый источник типов для Go и TypeScript.

**Non-Goals:**
- Горизонтальное масштабирование, очереди, микросервисы.
- Telegram Mini App и бот (следующий change).
- Чат внутри продукта, верификация личности, фото.
- SSR и SEO.

## Decisions

### D1. Монорепозиторий

```
reallink/
├── openspec/            # specs + changes
├── docs/product/        # концепция owner, product brief
├── docs/design/         # design system, макеты
├── api/openapi.yaml     # контракт HTTP API
├── server/              # Go: структура и правила — server/CONVENTIONS.md
├── frontend/            # SPA участника
├── adminpanel/          # SPA администратора
└── Makefile             # корневой: gen и build всех частей
```

Альтернатива — отдельные репозитории server, frontend и docs: почти каждый change затрагивает обе стороны и контракт, и specs пришлось бы синхронизировать вручную.

### D1a. Server: срезы по фичам (package-by-feature)

Структура взята из проекта stroylit_go и подробно описана в `server/CONVENTIONS.md` §2. Кратко:

```
server/internal/
├── app/        # композиционный корень
├── httpx/      # ошибки → HTTP, Bearer-middleware, rate limit
├── apigen/     # генерация oapi-codegen
├── platform/   # config, db + migrations, dbtest, log, clock, token, errs
├── participant/ admin/ matching/ meeting/ safety/ notify/ stats/   # срезы
└── web/        # embed SPA
```

Каждый срез — вертикальный: `http_*.go`, `service_*.go`, `ports.go`, а под вложенным `internal/` — `domain/` и `store/{sqlite,storetest}/`. Срезы не импортируют друг друга (проверяет `depguard`); зависимости — через интерфейсы в `ports.go` потребителя, сборка — в `app`. Соответствие срезов и capabilities — `server/CONVENTIONS.md` §2.1.

Альтернатива — раскладка по слоям (`internal/domain`, `internal/app`, `internal/store`): любое изменение фичи затрагивает все слои, и агент читает их целиком. Срез даёт агенту одну папку на задачу и границы, которые проверяет компилятор.

### D2. Contract-first API

`api/openapi.yaml` — источник правды. Backend: `oapi-codegen` (strict server поверх `net/http`). Frontend: `openapi-typescript` + `openapi-fetch`. В CI проверяется, что сгенерированный код совпадает с закоммиченным. Для агентов это главный «якорь»: сначала контракт, потом код.

Префиксы:
- `/api/public/*` — анкета и счётчик открытий;
- `/api/auth/*` — обмен ссылки на токен доступа участника, вход и выход администратора;
- `/api/p/*` — страница участника, `Authorization: Bearer <token>`;
- `/api/admin/*` — `Authorization: Bearer <token>` администратора.

Альтернатива — писать типы вручную с обеих сторон: дешевле на старте, но расхождения неизбежны.

### D3. HTTP-слой — chi + oapi-codegen strict server

`oapi-codegen` генерирует `chi-server` и `strict-server`. Каждый срез реализует свои операции в своём `Handler`, `app` собирает их встраиванием в одну структуру, которая удовлетворяет `StrictServerInterface` (`server/CONVENTIONS.md` §4). Middleware: `log/slog` (без персональных данных), recover, request id, аутентификация участника и администратора, rate limit.

Альтернатива — stdlib `net/http` без роутера: тоже работает, но chi уже используется в stroylit_go, и oapi-codegen генерирует под него напрямую.

### D4. SQLite → PostgreSQL

- Драйвер `modernc.org/sqlite` (pure Go, без cgo). WAL, `foreign_keys=ON`, `busy_timeout`, пишущие транзакции — `BEGIN IMMEDIATE`.
- SQL пишется руками и живёт только в `<срез>/internal/store/sqlite/`. Миграции — `goose`.
- Каждое хранилище проверяется набором `<срез>/internal/store/storetest/`, который не зависит от реализации.
- Правила переносимого SQL — `server/CONVENTIONS.md` §5: UUIDv7 в `TEXT`, время формирует Go (RFC 3339 UTC, миллисекунды), `CHECK` для перечислений, `ON CONFLICT ... DO UPDATE` вместо `INSERT OR REPLACE`, конкурентные переходы — условием в `UPDATE ... WHERE`.

При переходе на PostgreSQL в каждом срезе рядом появится `store/postgres/`, и тот же `storetest` подтвердит, что реализация ведёт себя так же. Миграции получат отдельный каталог для PostgreSQL. `domain`, сервисы и API не изменятся.

Альтернативы — sqlc (генерация кода из SQL; для двух СУБД всё равно нужны два набора запросов, а контрактные тесты дают ту же гарантию) и ORM (скрывает SQL, хуже с переносимостью). Ручной SQL со `storetest` уже проверен в stroylit_go.

### D5. Модель данных

- `participants` — анкета, `status` (active, paused, blocked, deleted), `locale` (ru, uk, ro, de), `source`, `age_confirmed_at`, `consent_at`, `consent_version`, `telegram_username`.
- `participant_languages` — язык и роль (`native`, `practice`, `home`); `participant_scenarios`; `participant_slots`; `participant_interests`.
- `participant_tokens` — токен персональной ссылки: `token_hash` (SHA-256), `revoked_at`. Сам токен не хранится.
- `participant_sessions` — токены доступа участника: `token_hash`, `participant_id`, `expires_at`, `revoked_at`, `last_used_at`.
- `places` — публичные места: название, адрес, ссылка на карту, тип, район, `active`.
- `matches` — участники, `scenario` (language, home), `meeting_language`, `status`, ответы обоих, снимок причин и оценки на момент создания.
- `meetings` — `starts_at`, `place_id`, `status`, `outcome`.
- `feedback` — уникальная пара (`meeting_id`, `participant_id`).
- `reports` — автор, на кого, причина, комментарий, `urgent`, статус, решение.
- `notifications` — получатель, тип события, ключ идемпотентности (уникальный), текст, `status` (pending, sent, obsolete), `channel` (пока только `manual`; поле нужно для бота).
- `admins`, `admin_sessions` (`token_hash`, `expires_at`, `revoked_at`), `admin_audit_log`.

Удаление участника: персональные поля обнуляются, `status = deleted`, токен ссылки и все сессии отзываются; строки остаются для статистики.

### D6. Матчинг и объяснения

`server/internal/matching/internal/score` — чистые функции, полностью покрыты unit-тестами.
- Фильтры — из spec `matching` и `safety`.
- Оценка 0–100: интересы до 40 (по 15 за общий, максимум 40); район 25 (один район), 12 (соседний, таблица в коде), 0 иначе; возраст 20 (тот же или соседний диапазон), 10 (через один), 0 иначе; время 15 (два и больше общих слота), 8 (один слот).
- Уровень для участника: ≥ 70 — «сильное совпадение», 45–69 — «хорошее», < 45 — «базовое».
- Причины: шаблоны с параметрами (общие интересы, взаимные языки, общий родной язык, район), переведённые на 4 языка; в ответ попадают 2–3 самые весомые.
- Снимок причин сохраняется в `matches`, чтобы предложение не менялось, если участник позже отредактирует анкету.

Кандидаты считаются на лету: при сотнях участников O(n²) не проблема. Предрасчёт — преждевременная оптимизация. На этапе 2 сюда же встраивается AI-объяснение, а интерфейс «кандидат + причины» не меняется.

### D7. Аутентификация — Bearer-токены

Cookie не используем. И участник, и администратор получают непрозрачный токен доступа и передают его в `Authorization: Bearer <token>`. Токены — 32 байта из `crypto/rand`, base64url; в БД хранится только SHA-256.

**Участник**
1. Открывает `/p/:linkToken`. Frontend сразу отправляет `POST /api/auth/participant/link { linkToken }` и убирает токен ссылки из адресной строки (`history.replaceState` на `/me`).
2. Сервер проверяет токен ссылки и создаёт запись в `participant_sessions` (срок 90 дней), отвечает `{ accessToken, expiresAt }`.
3. Frontend хранит `accessToken` в **localStorage** — так участник возвращается без ссылки (spec participant-registration, «Вход на устройстве по ссылке»).
4. «Выйти на этом устройстве» → `POST /api/auth/logout` отзывает сессию, frontend очищает localStorage. Перевыпуск ссылки, удаление и блокировка отзывают все сессии участника.

**Администратор**
1. `POST /api/auth/admin/login { email, password }` → `{ accessToken, expiresAt }` (срок 7 дней, spec admin-access).
2. Токен хранится в **sessionStorage** (закрыл вкладку — вышел). С отметкой «Запомнить меня» — в localStorage.
3. Выход отзывает сессию в `admin_sessions`.

**Общее**
- Middleware возвращает `participantID` или `adminID`; use cases не знают способа входа. В change `add-telegram-mini-app` появится ещё один обмен — `POST /api/auth/participant/telegram { initData }` — с тем же ответом и тем же токеном доступа.
- Ответ `401` → frontend удаляет токен и показывает вход (админ) или экран «откройте свою ссылку» (участник).
- Без cookie нет CSRF, заголовок `X-Requested-With` не нужен.
- Пароли администраторов — argon2id.

Альтернативы:
- HttpOnly-cookie: токен недоступен JavaScript (лучше против XSS), но нужны CSRF-защита и одинаковый origin, а с Mini App (iframe в Telegram) cookie работают ненадёжно. Выбран Bearer по решению команды.
- JWT: отзыв сложнее (нужен denylist), а сервер один; непрозрачный токен с проверкой в БД проще и отзывается сразу.
- Использовать токен ссылки как Bearer напрямую: проще, но тогда «выйти на устройстве» невозможно без перевыпуска ссылки, и Mini App не вписывается в ту же схему.

### D8. Уведомления — очередь текстов

Use case в той же транзакции, что и изменение пары или встречи, пишет строку в `notifications` с ключом идемпотентности `<event>:<match_id>:<participant_id>` и готовым текстом на языке получателя (шаблоны на 4 языках в срезе `notify`). Админка показывает очередь `pending`; «Отправлено» переводит в `sent`. При отмене пары неотправленные уведомления этой пары становятся `obsolete`.

В change `add-telegram-mini-app` к той же таблице добавится воркер доставки через бота (`channel = bot`), а ручная очередь останется запасным путём. Альтернатива — генерировать тексты на лету в админке: проще, но тогда нет истории «что и когда отправлено» и придётся переделывать под бота.

### D9. Почему без Telegram в первой версии

Решение owner — запуститься быстрее на вебе. Telegram при этом остаётся каналом связи: контакт партнёра — Telegram username, уведомления администратор отправляет в Telegram вручную. Следующий change добавит Mini App как второй вход в тот же профиль и автоматическую доставку.

### D10. Фоновые задачи

Одна горутина-планировщик раз в минуту: истечение предложений (72 часа), уведомления-напоминания (за 24 часа), запросы фидбека (+2 часа), итоги `unknown` (+7 дней). Все переходы идемпотентны, время инжектируется (`Clock`) для тестов. Альтернатива — внешний cron: лишняя движущаяся часть.

### D11. Frontend: состояние и UI

Подробные правила — `frontend/CONVENTIONS.md`. Кратко:
- **Mobile first**: базовая вёрстка под 360 px, брейкпоинты только расширяют.
- **React Query** — всё серверное: анкета, страница участника, очереди, пары, места, жалобы, статистика.
- Клиентское состояние — `useState`, react-hook-form, параметры URL (фильтры админки), i18next (язык), небольшой контекст (токен участника, сессия админа).
- **Redux Toolkit** — только для сложного клиентского состояния по критериям CONVENTIONS §2.3; добавляется в PR, где впервые нужен. RTK Query не используется.
- Bootstrap: сетка и утилиты — классами (`container`, `row`, `col-*`), без компонентов `Container`, `Row`, `Col`; react-bootstrap — для компонентов с поведением и форм.
- Формы — `react-hook-form` + `zod`, i18n — `react-i18next` (ru, uk, ro, de), `react-router`.

- Маршруты `frontend`: `/` (анкета), `/p/:linkToken` (обмен ссылки на токен), `/me` (страница участника), `/legal/impressum`, `/legal/privacy`.

### D11a. Админ-панель — отдельное приложение

`adminpanel/` — отдельное SPA с тем же стеком и теми же правилами (`adminpanel/CONVENTIONS.md` дополняет `frontend/CONVENTIONS.md`).
- Отдаётся тем же Go-сервером по пути `/admin/` (Vite `base: '/admin/'`, SPA-fallback на `/admin/index.html`). Один origin с API — CORS не нужен; аутентификация Bearer (D7).
- Свой API-клиент генерируется из того же `api/openapi.yaml`; использует только `/api/admin/*` и `/api/public/*` для справочников.
- Интерфейс на русском (ключи через i18next, чтобы позже добавить языки). Тексты уведомлений для участников приходят с сервера на языке участника.
- Mobile first сохраняется: администратор отправляет уведомления в Telegram с телефона, поэтому очередь уведомлений и подбор пар должны быть удобны на 360 px.
- Маршруты: `/admin/login`, `/admin/queue`, `/admin/matches`, `/admin/participants`, `/admin/places`, `/admin/reports`, `/admin/notifications`, `/admin/stats`.

Почему отдельно, а не раздел внутри `frontend`: публичный бандл не тянет код и зависимости админки, у админки свой релизный цикл и своя политика безопасности (например, `noindex`, строгий CSP), а агенты работают в чётко разделённых границах. Альтернатива — `/admin/*` внутри одного SPA с lazy-loading: меньше конфигурации, но общие зависимости и риск случайно открыть админский код публично. Общий код (тема, утилиты) на этапе MVP не выносим в пакет: тему копируем, API-клиенты генерируются отдельно; npm workspaces — когда общего кода станет заметно больше.

### D12. Деплой

`make build`: `frontend` собирается в `server/internal/web/dist/app`, `adminpanel` — в `server/internal/web/dist/admin`, оба встраиваются в бинарник через `embed`. На IONOS VPS (Германия): systemd-сервис за Caddy (автоматический HTTPS), файл SQLite на диске, ежедневный бэкап через `sqlite3 .backup` в IONOS Object Storage (регион DE). Секреты — переменные окружения.

## Risks / Trade-offs

- [Утечка персональной ссылки] → ссылка сразу обменивается на токен и убирается из адресной строки, `Referrer-Policy: no-referrer`, перевыпуск ссылки, Telegram партнёра скрыт до `accepted`.
- [Токен в localStorage/sessionStorage доступен при XSS] → строгий CSP (`script-src 'self'`, без сторонних скриптов и inline), запрет `dangerouslySetInnerHTML`, токены отзываемые и с ограниченным сроком, у администратора по умолчанию sessionStorage.
- [Ручная отправка уведомлений отнимает время администратора] → готовые тексты и очередь; бот — следующий change.
- [SQLite и параллельные записи (планировщик + запросы)] → WAL, одно соединение на запись, короткие транзакции.
- [Дрейф SQL-диалектов] → правила D4 и `storetest` для каждого среза; реализация `postgres` проверяется тем же набором.
- [Пересечение RTK и React Query] → RTK только по исключению, правила в `frontend/CONVENTIONS.md`.
- [Агенты генерируют код мимо контракта или в обход границ срезов] → CI-проверка `make gen` без diff, `depguard`; задачи начинаются с изменения OpenAPI.
- [Расход токенов агентами] → работа в одном срезе, вложенные `CLAUDE.md`, запрет чтения генерированного кода (`.claude/settings.json`), комментарии только «почему», без отдельных документов на фичу (`server/CONVENTIONS.md` §11).
- [GDPR, оператор — физлицо] → минимум данных, без страны и национальности, удаление, хостинг в DE, Impressum и Datenschutzerklärung до запуска (тексты пишет owner, это не код).
- [Чувствительность смешанных пар ru/uk] → Home Connection только по общему языку, который выбрал сам участник; пару подтверждает человек.

## Migration Plan

Первый запуск, переносить нечего. Если этап 0 уже прошёл, участники из Google Form импортируются CLI из CSV — только с подтверждёнными согласием и 18+. Откат — остановить сервис и вернуться к ручному процессу; данные остаются в файле SQLite.

## Open Questions

- Go module path: пока `reallink/server`; меняется, если репозиторий переедет на GitHub или GitLab.
- Таблица соседства районов Ганновера и стартовый список публичных мест — данные, которые заполняются через админку или CLI.
- Тексты Impressum и Datenschutzerklärung — готовит owner.
