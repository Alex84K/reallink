# Frontend Conventions (web)

Правила для всех, кто пишет код в `frontend/` (приложение участника) и `adminpanel/` (админ-панель), включая AI-агентов. Особенности админ-панели — в `adminpanel/CONVENTIONS.md`, всё остальное общее. Если правило мешает, его сначала меняют в этом файле (через PR), а потом нарушают.

Стек: React + TypeScript (Vite, шаблон vite-template-redux), TanStack React Query, Bootstrap 5 (+ react-bootstrap для компонентов), react-router, react-hook-form + zod, react-i18next, openapi-fetch. Линтер и форматтер — oxlint и oxfmt. Redux Toolkit подключён шаблоном, но используется только по правилу из раздела 2.

---

Версии, которые держим намеренно:

- **TypeScript 6**, не 7 — `openapi-typescript` пока официально поддерживает только TS 5; в `package.json` стоит `overrides`, генерация на TS 6 проверена. Переходим на TS 7, когда генератор его поддержит.
- **msw 2.x**, не 3 — Vitest 5 (`@vitest/mocker`) поддерживает msw `^2`.
- **react-router 8** — импорты только из `react-router`; пакет `react-router-dom` не используем.
- **Node 24 LTS** (`.nvmrc`), минимум 22.22 (`engines`).

## 1. Mobile first

Основной пользователь открывает продукт с телефона по ссылке из Telegram-чата. Сначала верстаем экран для узкого телефона, потом расширяем на большие экраны.

**Правила**

1. Базовые классы и стили — для самого узкого экрана. Брейкпоинты (`sm`, `md`, `lg`, `xl`) только **добавляют** поведение для экранов шире.
2. В собственном CSS — только `@media (min-width: …)` через миксин `media-breakpoint-up()`. `max-width` и `media-breakpoint-down()` не используем.
3. Целевые ширины для проверки: **360 px** (основная), 768 px, 1280 px. Горизонтальной прокрутки страницы на 360 px быть не должно.
4. Интерактивные элементы — не меньше **44×44 px**. Основная кнопка действия на мобильном — на всю ширину: кнопка `w-100` внутри `col-12 col-md-auto`, так с `md` она сжимается до своей ширины.
5. Формы — в одну колонку на мобильном. Поля с правильным `type` и `inputMode` (`tel`, `email`, `numeric`), чтобы открывалась нужная клавиатура.
6. Таблицы — в `.table-responsive`. Если колонок больше 4, на мобильном показываем список карточек, а таблицу — с `md`.
7. Никаких hover-only взаимодействий: всё, что открывается при наведении, должно открываться и по нажатию.
8. Изображения и иконки — с фиксированными размерами, чтобы не прыгала вёрстка. Тяжёлых картинок нет (фото в MVP не используются).

**Пример**

```tsx
// Мобильный: карточки друг под другом. С md — две колонки, с xl — три.
<div className="container py-3">
  <div className="row g-3">
    {candidates.map(c => (
      <div key={c.id} className="col-12 col-md-6 col-xl-4">
        <CandidateCard candidate={c} />
      </div>
    ))}
  </div>
</div>
```

```scss
// Правильно: от мобильного вверх
.offer-actions {
  display: flex;
  flex-direction: column;
  gap: $spacer * 0.5;

  @include media-breakpoint-up(md) {
    flex-direction: row;
  }
}
```

---

## 2. Состояние: React Query по умолчанию, RTK — по исключению

### 2.1 React Query — всё, что пришло с сервера

Любые данные из API (анкета, страница участника, пары, очереди, места, жалобы, статистика) живут **только** в кеше React Query. Их не копируют в `useState`, контекст или Redux.

- Каждая фича описывает ключи в одном месте — фабрика `xxxKeys` в `features/<feature>/api/keys.ts`.
- Запросы и мутации оборачиваются в хуки: `useParticipantQuery`, `useAcceptOfferMutation`. Компоненты не вызывают `fetch` и API-клиент напрямую.
- После мутации — `invalidateQueries` по ключам фичи. `setQueryData` — только для оптимистичных обновлений, с откатом в `onError`.
- Значения по умолчанию задаются в `QueryClient` (`staleTime: 30_000`, `retry: 1`, без `refetchOnWindowFocus` в админке). Переопределяем в хуке, только если есть причина, и пишем её в комментарии.
- Состояния загрузки и ошибки обрабатываются в каждом экране: скелетон или спиннер, понятное сообщение и кнопка «Повторить».

```ts
// features/participant/api/keys.ts
export const participantKeys = {
  all: ["participant"] as const,
  me: () => [...participantKeys.all, "me"] as const,
  offer: (matchId: string) =>
    [...participantKeys.all, "offer", matchId] as const,
}

// features/participant/api/useAcceptOfferMutation.ts
export function useAcceptOfferMutation() {
  const qc = useQueryClient()
  return useMutation({
    mutationFn: (matchId: string) =>
      api.POST("/api/p/matches/{id}/accept", {
        params: { path: { id: matchId } },
      }),
    onSuccess: () => qc.invalidateQueries({ queryKey: participantKeys.all }),
  })
}
```

### 2.2 Где хранить клиентское состояние (без Redux)

| Что                                                      | Где                                                                                                                                          |
| -------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| Состояние одного компонента (открыт ли модал, шаг формы) | `useState` / `useReducer`                                                                                                                    |
| Значения формы и валидация                               | react-hook-form                                                                                                                              |
| Фильтры, сортировка, пагинация, вкладки списков          | Параметры URL (`useSearchParams`) — чтобы ссылкой можно было поделиться и не терялось при обновлении                                         |
| Язык интерфейса                                          | i18next (`i18n.changeLanguage`)                                                                                                              |
| Токен доступа                                            | `localStorage` / `sessionStorage` через модуль `app/auth/tokenStorage.ts` (см. §3a); статус «вошёл / не вошёл» — React Context в `app/auth/` |

### 2.3 Когда можно Redux Toolkit

RTK добавляется, только если выполнены **оба** условия:

1. Клиентское (не серверное) состояние нужно **нескольким несвязанным частям экрана** или нескольким маршрутам, и передача через props или контекст становится запутанной;
2. У этого состояния **сложная логика переходов** (много действий, зависимые изменения, отмена и повтор), которую удобнее описать редьюсером и тестировать отдельно.

Примеры, где RTK оправдан: конструктор пары в админке, если в нём одновременно меняются выбранные участники, сценарий, место, время и черновик сообщения; многошаговый мастер со сложными переходами между шагами.

Примеры, где RTK **не нужен**: список с фильтрами (URL), форма (react-hook-form), данные с сервера (React Query), флаг «модал открыт» (`useState`).

Правила, если RTK всё-таки нужен:

- Store из шаблона (`app/store.ts`) остаётся пустым, пока не появится первый слайс. Первый слайс добавляется с обоснованием в описании PR. Демо-фичи шаблона (`counter`, `quotes`) удаляются.
- Слайс живёт в фиче: `features/<feature>/model/<name>Slice.ts`. Глобальных «общих» слайсов нет.
- В store **не** кладём данные с сервера. RTK Query не используем — серверным состоянием владеет только React Query.
- Типизированные хуки `useAppSelector` и `useAppDispatch` — из `app/store.ts`.

---

## 3. Bootstrap: сетка классами, а не компонентами

### 3.1 Сетка и раскладка — только классы

Для сетки и раскладки используем классы Bootstrap на обычных элементах. Компоненты `Container`, `Row`, `Col`, `Stack` из react-bootstrap **не используем**.

```tsx
// Правильно
<div className="container">
  <div className="row g-3 align-items-center">
    <div className="col-12 col-md-8">…</div>
    <div className="col-12 col-md-4">…</div>
  </div>
</div>

// Неправильно
<Container>
  <Row>
    <Col xs={12} md={8}>…</Col>
  </Row>
</Container>
```

То же для утилит: отступы (`p-3`, `mb-2`, `gap-2`), flex (`d-flex`, `flex-column`, `flex-md-row`), выравнивание, видимость (`d-none d-md-block`), типографика (`fs-5`, `fw-semibold`, `text-muted`) — классами.

### 3.2 Компоненты

- Компоненты с поведением (Modal, Offcanvas, Dropdown, Toast, Tooltip, Accordion, Tabs) — из react-bootstrap: они дают управление фокусом, клавиатуру и ARIA.
- Элементы форм — react-bootstrap `Form.*` вместе с react-hook-form, чтобы единообразно показывать ошибки (`isInvalid`, `Form.Control.Feedback`).
- Простые элементы без поведения (кнопки, карточки, бейджи, алерты, списки) — можно и классами, и компонентами react-bootstrap. В одной фиче придерживаемся одного варианта.

### 3.3 Стили

- Цвета, шрифты, отступы и компоненты описаны в [`docs/design/design-system.md`](../docs/design/design-system.md). Тема — через переменные Bootstrap в `src/styles/_variables.scss` (цвета, шрифты, радиусы, отступы) до подключения Bootstrap. Значения Bootstrap не переопределяем каскадом с `!important`.
- Свои стили — только если нет утилиты Bootstrap. Файл рядом с компонентом: `Component.module.scss`.
- Инлайн-стили (`style={{…}}`) не используем, кроме динамических значений (например, ширина прогресс-бара).
- Цвета только из переменных темы, без жёстких hex в компонентах.

---

## 3a. Аутентификация и API-клиент

Работаем через **Bearer-токен** в заголовке `Authorization`. Cookie не используем.

**Хранение**

| Приложение                   | Где                                                   | Ключ                         | Почему                                              |
| ---------------------------- | ----------------------------------------------------- | ---------------------------- | --------------------------------------------------- |
| frontend (участник)          | `localStorage`                                        | `rl.participant.accessToken` | Участник возвращается на сайт без ссылки до 90 дней |
| frontend в Telegram Mini App | `sessionStorage`                                      | `rl.participant.accessToken` | Telegram даёт свежий `initData` при каждом запуске  |
| adminpanel                   | `sessionStorage`; с «Запомнить меня» — `localStorage` | `rl.admin.accessToken`       | Закрыл вкладку — вышел                              |

**Правила**

1. Доступ к хранилищу — **только** через `app/auth/tokenStorage.ts` (`getToken`, `setToken`, `clearToken`). Прямых вызовов `localStorage`/`sessionStorage` в фичах нет.
2. Каждое обращение к хранилищу — в `try/catch`: в приватном режиме Safari и при запрете cookies/storage оно бросает исключение. Тогда токен живёт только в памяти до закрытия вкладки.
3. Токен добавляет **один** middleware API-клиента (openapi-fetch). Компоненты и хуки заголовок не трогают.
4. Ответ `401` от любого запроса → `clearToken()`, `queryClient.clear()`, переход на экран входа (участник: «откройте свою персональную ссылку»; админ: `/admin/login` с возвратом на исходную страницу).
5. Персональная ссылка `/p/:linkToken` сразу обменивается на токен доступа (`POST /api/auth/participant/link`), после чего адрес заменяется на `/me` через `navigate('/me', { replace: true })` — токен ссылки не остаётся в истории браузера.
6. Выход — запрос `POST /api/auth/logout`, затем `clearToken()` и `queryClient.clear()`, даже если запрос не удался.
7. Токен **никогда** не попадает в URL (кроме самой ссылки), логи, `console`, сообщения об ошибках и аналитику.
8. Поскольку токен доступен JavaScript, защищаемся от XSS: никакого `dangerouslySetInnerHTML`, никаких сторонних скриптов и CDN, строгий CSP настраивается на сервере.

```ts
// app/auth/tokenStorage.ts — единственное место работы с хранилищем
const KEY = "rl.participant.accessToken"
let memoryToken: string | null = null

export function getToken(): string | null {
  try {
    return localStorage.getItem(KEY) ?? memoryToken
  } catch {
    return memoryToken
  }
}
export function setToken(token: string): void {
  memoryToken = token
  try {
    localStorage.setItem(KEY, token)
  } catch {
    /* только память */
  }
}
export function clearToken(): void {
  memoryToken = null
  try {
    localStorage.removeItem(KEY)
  } catch {
    /* нечего чистить */
  }
}

// api/client.ts
const client = createClient<paths>({ baseUrl: "/" })
client.use({
  onRequest({ request }) {
    const token = getToken()
    if (token) request.headers.set("Authorization", `Bearer ${token}`)
    return request
  },
  onResponse({ response }) {
    if (response.status === 401) onUnauthorized() // clearToken + queryClient.clear + redirect
    return response
  },
})
```

## 4. Структура проекта

```
src/
├── app/            # корень: провайдеры (QueryClient, i18n, Router), auth/ (tokenStorage, контекст), store.ts (пустой), маршруты, layout
├── api/            # schema.d.ts — сгенерирован (руками не править); client.ts — клиент с Bearer-middleware
├── features/
│   └── <feature>/{api,components,pages,model,lib}   # frontend: registration, participant
├── shared/         # переиспользуемые UI-компоненты, хуки, утилиты без привязки к фиче
├── i18n/           # ресурсы ru, uk, ro, de
└── styles/         # _variables.scss, main.scss
```

- Фича не импортирует внутренности другой фичи. Общее — переносим в `shared/`.
- Страницы (`pages/`) собирают экран из компонентов и хуков; бизнес-логики в JSX нет.

## 5. TypeScript и API

- `strict: true`, без `any` и `@ts-ignore` (при крайней необходимости — `@ts-expect-error` с комментарием).
- Типы запросов и ответов — **только** из сгенерированного `src/api` (из `api/openapi.yaml`). Свои DTO не пишем. Нужен новый эндпоинт — сначала меняем OpenAPI, потом `make gen`.
- Схемы zod повторяют ограничения из specs (длины, количество интересов, форматы) и лежат в `features/<feature>/lib/schema.ts`.

## 6. Тексты, языки, даты

- Ни одной строки интерфейса в JSX — только `t('…')`. Ключи с пространством имён фичи: `registration.fields.district`.
- Все 4 языка (ru, uk, ro, de) обновляются в том же PR. Если перевода нет — ключ добавляется во все файлы с пометкой `TODO(i18n)`, CI это ловит.
- Строки не склеиваем; для чисел и множественного числа — интерполяция и plural-формы i18next.
- Даты и время — через `Intl.DateTimeFormat` с `timeZone: 'Europe/Berlin'` и текущей локалью. С сервера приходит UTC.

## 7. Доступность

- У каждого поля — видимый `<label>`; у иконок-кнопок — `aria-label`.
- Ошибки форм связаны с полем (`aria-describedby` через `Form.Control.Feedback`).
- Контраст текста — не ниже WCAG AA; фокус всегда видим.

## 8. Приватность в коде

- Токен — только через `tokenStorage` (§3a), никогда в `console`, URL, ошибки и аналитику.
- Ничего, что получено с сервера, не логируем в консоль в production.
- Данные партнёра показываем ровно так, как их отдал сервер; не «достраиваем» скрытое на клиенте.

## 9. Тесты и качество

- Vitest + Testing Library, моки API — MSW. Тестируем поведение глазами пользователя (роли, тексты), а не внутренности компонентов.
- Хук запроса или мутации, форма и каждый экран со сложной логикой получают тест.
- oxlint (`npm run lint`) и oxfmt (`npm run format:check`), `npm run type-check` и `npm run test` должны проходить перед PR. Конфиги — `.oxlintrc.json` и `.oxfmtrc.json` из шаблона.

## 10. Именование

- Компоненты — `PascalCase.tsx`, один компонент на файл.
- Хуки — `useXxx.ts`; хуки React Query — `useXxxQuery` и `useXxxMutation`.
- Обработчики — `handleXxx` внутри компонента, `onXxx` в props.
