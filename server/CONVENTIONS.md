# Server Conventions

Правила Go-кода в `server/`. Обязательны для людей и AI-агентов. Чтобы изменить правило, сначала меняем этот файл (PR), потом код.

Поведение системы описано в `openspec/` (specs и changes), контракт API — в `api/openapi.yaml`. Здесь только **как** писать код.

## 0. Порядок работы агента

1. Прочитать задачу в `openspec/changes/<change>/tasks.md` и spec своей capability (`specs/<capability>/spec.md`).
2. Прочитать этот файл.
3. Работать **в одном срезе** (`internal/<slice>/`). В соседние срезы и `platform/` заглядывать только за сигнатурами.
4. Перед завершением задачи пройти чек-лист §12.

Не читать: `internal/apigen/*.gen.go` (генерируется), `data/`, `bin/`.

## 1. Стек

| Что | Чем |
|-----|-----|
| Go | последняя стабильная версия, указана в `go.mod` |
| HTTP | `net/http` + `github.com/go-chi/chi/v5` |
| Контракт | `api/openapi.yaml` → `oapi-codegen` (chi-server, strict-server, models) → `internal/apigen` |
| БД | SQLite через `modernc.org/sqlite` (без cgo); позже PostgreSQL через `pgx` |
| SQL | пишется руками в `store/sqlite`; ORM и sqlc не используем |
| Миграции | `goose`, SQL-файлы в `internal/platform/db/migrations` |
| Логи | `log/slog` (JSON в production) |
| Пароли | argon2id (`golang.org/x/crypto/argon2`) |
| ID | UUIDv7 (`github.com/google/uuid`), генерирует сервер |
| Линтер | `golangci-lint` v2 (`.golangci.yml`) |

Новая зависимость — только с обоснованием в описании PR.

## 2. Структура

```
server/
├── cmd/server/          # main: конфиг → БД → app.New → HTTP + фоновые задачи
├── cmd/admin/           # CLI: создать администратора, импорт мест
└── internal/
    ├── app/             # композиционный корень: единственное место, знающее все срезы
    ├── httpx/           # ошибки → HTTP, Bearer-middleware, rate limit, request id
    ├── apigen/          # сгенерировано из OpenAPI — руками не править
    ├── platform/        # config, db (+migrations), dbtest, log, clock, token, errs
    ├── participant/     # срез
    ├── admin/           # срез
    ├── matching/        # срез
    ├── meeting/         # срез
    ├── safety/          # срез
    ├── notify/          # срез
    ├── stats/           # срез (только чтение)
    └── web/             # embed собранных frontend и adminpanel
```

### 2.1 Срезы и capabilities

| Срез | Capabilities (openspec) | Таблицы |
|------|------------------------|---------|
| `participant` | participant-registration | participants, participant_languages, _scenarios, _slots, _interests, participant_tokens, participant_sessions |
| `admin` | admin-access | admins, admin_sessions, admin_audit_log |
| `matching` | matching | matches |
| `meeting` | meeting-coordination, meeting-feedback | places, meetings, feedback |
| `safety` | safety | reports (блокировку выполняет `participant` через порт) |
| `notify` | notifications | notifications |
| `stats` | admin-stats | своих нет — читает чужие (единственное исключение, §3) |

Таблицей владеет ровно один срез. Писать в неё может только он.

### 2.2 Анатомия среза

```
internal/matching/
├── http.go, http_<topic>.go        # хендлеры: apigen-запрос → service → apigen-ответ
├── service.go, service_<topic>.go  # use cases, транзакции, проверка прав
├── ports.go                        # интерфейсы, которые срез ПОТРЕБЛЯЕТ (репозиторий, соседи, Clock)
├── matchingapi/                    # публичные типы для соседей (только stdlib), если нужны
├── internal/
│   ├── domain/                     # сущности, статусы, правила, доменные ошибки — без I/O
│   ├── score/                      # (пример) чистые функции оценки
│   └── store/
│       ├── sqlite/                 # весь SQL среза
│       └── storetest/              # контрактные тесты репозитория для любой реализации
└── *_test.go
```

- Вложенный `internal/` — граница, которую проверяет компилятор: домен и SQL среза недоступны снаружи.
- Файл — до ~400 строк. Растёт — делим по теме: `service_offer.go`, `service_cancel.go`.
- Имена файлов предсказуемые (`http_*`, `service_*`, `store/sqlite/*`), чтобы нужное находилось поиском по имени, а не чтением.

## 3. Зависимости

- **Срез не импортирует срез.** Нужен сосед — объявляем узкий интерфейс в своём `ports.go`, реализацию передаёт `app`. Типы для обмена — в `<сосед>api/` (только stdlib).
- `platform`, `httpx`, `apigen` не импортируют срезы.
- `internal/domain` не импортирует `database/sql`, `net/http`, `apigen`, `platform/db`.
- **Исключение:** `stats` читает таблицы других срезов напрямую своими SELECT, ничего не пишет. При изменении схемы другого среза проверяем `stats`.
- Правила проверяет `depguard` в `.golangci.yml`.

## 4. HTTP и контракт

- Contract-first: сначала `api/openapi.yaml`, затем `make gen`, затем код. Сгенерированное не правим.
- `apigen` даёт один `StrictServerInterface`. Каждый срез реализует свои операции в своём `Handler`; `app` собирает их встраиванием:
  ```go
  type server struct {
      *participant.Handler
      *matching.Handler
      // ...
  }
  var _ apigen.StrictServerInterface = server{}
  ```
- Хендлер тонкий: распаковать запрос → вызвать сервис → упаковать ответ. Бизнес-правил в хендлере нет.
- Кто вызывает (участник или админ), берётся из контекста (`httpx.ParticipantID(ctx)`, `httpx.AdminID(ctx)`), никогда из тела запроса.

**Ошибки → коды** (маппинг один, в `httpx`):

| Доменная ошибка | Код | Когда |
|-----------------|-----|-------|
| `errs.Validation` (с полями) | 400 | неверные данные анкеты |
| `errs.Unauthorized` | 401 | нет или отозван токен |
| `errs.Forbidden` | 403 | заблокирован; админ без прав |
| `errs.NotFound` | 404 | нет объекта **или он чужой** — не раскрываем существование |
| `errs.Conflict` | 409 | недопустимый переход статуса, дубль username |
| `errs.RateLimited` | 429 | лимит запросов |

Тело ошибки: `{"code": "...", "message": "...", "fields": {"district": "required"}}`. `code` — стабильная строка для фронтенда; `message` — для логов и разработчика, не для показа пользователю.

## 5. База данных

- SQL — **только** в `<slice>/internal/store/sqlite/`. Ни строки SQL за его пределами (кроме `stats` и фикстур `platform/dbtest`).
- Соединение открывает только `platform/db`: `foreign_keys=ON`, `journal_mode=WAL`, `busy_timeout=5000`. Пишущие транзакции — `BEGIN IMMEDIATE`.
- Транзакция открывается в сервисе через порт `Transactor`; репозитории берут её из контекста (`db.FromContext`), сами транзакций не открывают.
- **Переносимый SQL** (SQLite → PostgreSQL):
  - ID — UUIDv7 строкой (`TEXT`), генерирует сервис;
  - время — `TEXT`, формирует Go одной функцией `clock.Format` (RFC 3339, UTC, миллисекунды, `Z`); SQL время не вычисляет (`datetime('now')` запрещён);
  - перечисления — `TEXT` + `CHECK (status IN (...))`;
  - upsert — `INSERT ... ON CONFLICT (...) DO UPDATE ... WHERE ...`; `INSERT OR REPLACE/IGNORE` запрещены;
  - без `AUTOINCREMENT`, JSON-функций, `ROWID`;
  - конкурентные изменения — условием в `UPDATE ... WHERE status = 'proposed'` и проверкой `RowsAffected`, а не «прочитать → сравнить в Go → записать».
- Миграции: `goose`, по одной на изменение, всегда с `-- +goose Down`. Применённую миграцию не редактируем — пишем новую.
- Удаление участника — обнуление персональных полей, не `DELETE` (статистика должна сохраниться).

## 6. Время

- `time.Now()` в сервисах и домене запрещён. Время берём из порта `Clock` (`platform/clock`); в тестах — фиксированные часы.
- Хранение и API — UTC. `Europe/Berlin` — только для недель в статистике и дат в текстах уведомлений.

## 7. Безопасность и приватность

- Токены (ссылка, сессии): 32 байта из `crypto/rand`, base64url; в БД — только SHA-256 (`platform/token`). Сравнение — по хешу, поиск — по индексу.
- Что видит участник о партнёре, решает **сервис** по статусу пары (spec matching, «Раскрытие контактов»). Числовая оценка никогда не попадает в ответы `/api/p/*`.
- В логи **не** пишем: имя, Telegram username, «о себе», комментарии жалоб, токены, тела запросов. Пишем ID, коды и длительность.
- Rate limit — в `httpx`, на анкету и вход администратора.

## 8. Ошибки и логи

- Оборачиваем с контекстом: `fmt.Errorf("matching: accept offer: %w", err)`. Проверка — `errors.Is/As`.
- Доменные ошибки — типизированные значения в `internal/domain` (или `platform/errs`), не строки.
- Логирует только верхний уровень (middleware и фоновые задачи). Сервисы и репозитории возвращают ошибку, а не пишут её в лог.
- `panic` — только для ошибок программиста при старте (неверная конфигурация).

## 9. Фоновые задачи

- Регистрируются в `app` как `Runner` с методом `Run(ctx)`; останавливаются по отмене контекста.
- Каждый шаг идемпотентен: повторный запуск ничего не ломает (истечение 72 ч, напоминания, итоги встреч).

## 10. Тесты

- **Сценарий из spec — это тест.** Имя: `Test<Сущность>_<Сценарий>`, над тестом — название сценария из spec:
  ```go
  // spec matching — Scenario: Оба согласны
  func TestMatch_BothAccept(t *testing.T) { ... }
  ```
- Домен и `score` — табличные unit-тесты без БД.
- Хранилище — набор `storetest.Run(t, newRepo)`, который запускается для `sqlite` (позже и для `postgres`).
- HTTP — через `httptest` на собранном роутере (`app.New`) с временной SQLite (`platform/dbtest`) и фиксированными часами.
- Моки руками (структуры-фейки), без фреймворков моков.
- Запуск: `make test-slice S=matching` во время работы, `make test` перед PR.

## 11. Комментарии и документация (экономия токенов)

- Комментарий объясняет **почему**, а не что. 1–3 строки. Очевидное не комментируем.
- Не пересказываем specs и design в коде. Ссылка — одной строкой: `// spec safety: Не сводить снова`.
- Doc-комментарий пакета — 1–2 предложения.
- Отдельных документов на фичу (DESIGN_*, PLANNING_*) **не заводим**: это делают changes в OpenSpec. ADR — только для решений на весь проект, не длиннее страницы, в `docs/adr/`.
- README сервера — только запуск, переменные окружения и команды.

## 12. Чек-лист перед завершением задачи

- [ ] `make gen` не даёт diff (если менялся `api/openapi.yaml`)
- [ ] `make lint` без ошибок
- [ ] `make test-slice S=<срез>` зелёный; новые сценарии spec покрыты тестами
- [ ] Нет SQL вне `store/`, нет `time.Now()` в сервисе, нет PII в логах
- [ ] Чекбокс задачи отмечен в `tasks.md`
