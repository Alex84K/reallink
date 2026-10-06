# reallink server

Go-бэкенд Real Connections: API для `frontend` и `adminpanel`, фоновые задачи, раздача собранных SPA.
Правила кода — [CONVENTIONS.md](CONVENTIONS.md). Поведение — [`openspec/`](../openspec/).

## Запуск

```sh
cp .env.example .env        # заполнить значения
make migrate                # применить миграции к DB_PATH
go run ./cmd/admin create --email you@example.com   # первый администратор
make run                    # http://localhost:8080
```

## Команды

| Команда | Что делает |
|---------|-----------|
| `make gen` | Генерирует `internal/apigen` из `../api/openapi.yaml` |
| `make run` | Запускает сервер |
| `make test` | Все тесты (`-race`) |
| `make test-slice S=matching` | Тесты одного среза, печатает только падения |
| `make lint` | golangci-lint |
| `make migrate` / `make migrate-new N=add_x` | Применить миграции / создать новую |
| `make build` | Бинарник `bin/server` |

## Переменные окружения

См. [.env.example](.env.example). Обязательные без значений по умолчанию: `DB_PATH`, `APP_BASE_URL`.
