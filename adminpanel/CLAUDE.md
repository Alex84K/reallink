# adminpanel/ — инструкции для агентов

- Перед работой прочитай `../frontend/CONVENTIONS.md` (общие правила) и `CONVENTIONS.md` (отличия админки). Визуальный стиль — `../docs/design/design-system.md`.
- Приложение работает под `/admin/`; токен — в sessionStorage («Запомнить меня» → localStorage); на 401 — на `/admin/login`.
- Не читай `src/api/schema.d.ts` (генерация).
