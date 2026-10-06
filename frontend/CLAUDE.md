# frontend/ — инструкции для агентов

- Перед работой прочитай `CONVENTIONS.md`. Визуальный стиль — `../docs/design/design-system.md`.
- Главное: mobile first (360 px); серверные данные — только React Query; Redux Toolkit — только по §2.3; сетка Bootstrap — классами, без `Container`/`Row`/`Col`; токен — только через `app/auth/tokenStorage.ts`.
- Не читай `src/api/schema.d.ts` (генерация). Типы API бери из него через импорт, не переписывай.
- Проверка: `npm run type-check`, `npm run lint`, `npm run test`.
