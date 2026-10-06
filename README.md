# Real Connections (reallink)

Платформа, которая подбирает совместимых людей поблизости и переводит знакомство в реальные встречи.
Этап 1 — релоканты и диаспоры в Ганновере: языковые партнёры и «свои» люди рядом. Решения и дорожная карта — [docs/product/product-brief.md](docs/product/product-brief.md).

## Структура

| Путь | Что там |
|------|---------|
| `openspec/specs/` | Действующие спецификации (что система делает сейчас) |
| `openspec/changes/` | Предлагаемые и текущие изменения: proposal, design, tasks, delta specs |
| `docs/product/` | Концепция owner (`idea.md`), product brief, concierge-план, питч |
| `docs/design/` | Design system (цвета, шрифты, компоненты) и макеты |
| `api/openapi.yaml` | Контракт HTTP API *(появится в change add-hannover-mvp)* |
| `server/` | Go + SQLite (позже PostgreSQL), срезы по фичам — [CONVENTIONS](server/CONVENTIONS.md) |
| `frontend/` | Приложение участника: React + TypeScript, React Query, Bootstrap — [CONVENTIONS](frontend/CONVENTIONS.md) |
| `adminpanel/` | Админ-панель (отдельное SPA, отдаётся на `/admin/`) — [CONVENTIONS](adminpanel/CONVENTIONS.md) |

## Процесс (OpenSpec)

1. `/opsx:propose "<идея>"` — создать change с proposal, specs, design и tasks.
2. Обсудить и поправить артефакты, затем `openspec validate <change> --strict`.
3. `/opsx:apply` — реализация по tasks.md.
4. `/opsx:archive` — после релиза delta specs переносятся в `openspec/specs/`.

CLI: `npm i -g @fission-ai/openspec`, затем `openspec list`, `openspec show <change>`, `openspec view`.

## Текущие changes

1. `add-hannover-mvp` — этап 1, веб: анкета, Language и Home Connection, подбор пар, встречи, безопасность, ручная очередь уведомлений, статистика.
2. `add-telegram-mini-app` — Mini App и бот (после архивации первого change).
# reallink
# reallink
