# server/ — инструкции для агентов

- Перед работой прочитай `CONVENTIONS.md` (правила Go-кода) и spec своей capability в `openspec/`.
- Работай в одном срезе `internal/<slice>/`. Таблица «срез → capability» — CONVENTIONS §2.1.
- Не читай `internal/apigen/*.gen.go` — это генерация из `api/openapi.yaml`. Нужен эндпоинт — меняй `api/openapi.yaml`, затем `make gen`.
- Проверка своей работы: `make test-slice S=<slice>` (печатает только падения), затем `make lint`.
- Комментарии — только «почему», 1–3 строки. Документы на фичу не создавай.
