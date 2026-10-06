# Design

## Context

Применяется поверх реализованного и заархивированного `add-hannover-mvp`: монорепозиторий, Go + SQLite, контракт OpenAPI, таблица `notifications` с полем `channel`, middleware аутентификации участника, который возвращает `participantID` (см. design D7 и D8 того change). Мотивация — proposal.md.

## Goals / Non-Goals

**Goals:**
- Mini App — второй вход в тот же профиль; use cases не меняются.
- Один frontend для веба и Mini App.
- Надёжная доставка уведомлений ботом без дублей.

**Non-Goals:**
- Переписка через бота, команды кроме `/start`, inline-режим.

## Decisions

Новый срез `server/internal/telegram` (проверка initData, бот, webhook, привязка). `participant` получает от него проверенного Telegram-пользователя через порт; `notify` отправляет сообщения через порт `Sender`, который реализует `telegram`.


### T1. Аутентификация Mini App

При запуске Mini App frontend отправляет `POST /api/auth/participant/telegram { initData }`. Сервер проверяет HMAC-SHA256 ключом бота (алгоритм Telegram WebApp) и `auth_date` не старше 24 часов, находит участника по `telegram_user_id` и выдаёт такой же Bearer-токен доступа, как при входе по ссылке (design D7 в `add-hannover-mvp`). Middleware `/api/p/*` не меняется. Если участника нет, ответ «нужна регистрация» — frontend показывает анкету; регистрация отправляется вместе с `initData`, Telegram ID и username берутся из проверенных данных, а не из тела запроса, и в ответе сразу приходит токен.

Токен в Mini App хранится в sessionStorage: при каждом запуске Telegram передаёт свежий `initData`, долгий вход не нужен.

Альтернатива — передавать `initData` в каждом запросе: без отдельного обмена, но вторая схема аутентификации в middleware и повторная проверка подписи на каждый запрос.

### T2. Бот

- Один бот. Mini App открывается кнопкой меню и ссылкой `t.me/<bot>/app?startapp=<src>`.
- Обновления — webhook `/telegram/webhook/{secret}` с проверкой заголовка `X-Telegram-Bot-Api-Secret-Token`. В dev-режиме — long polling по флагу.
- `/start <code>` привязывает Telegram к веб-профилю (одноразовый код, хеш в `telegram_link_codes`, 30 минут).

### T3. Доставка уведомлений

Воркер (в существующем планировщике) выбирает `pending` уведомления получателей с `telegram_user_id`, отправляет через `sendMessage` с кнопкой `web_app`, повторяет с экспоненциальной задержкой до 24 часов. Ответ 403 (бот заблокирован) или исчерпанные повторы → `channel = manual`, уведомление остаётся в ручной очереди с причиной. Rate limit Telegram (около 30 сообщений в секунду) при нашем объёме не достигается, но воркер отправляет последовательно.

Альтернатива — отправлять из хендлера: потеря сообщений при сбоях Telegram и дубли при повторах.

### T4. Frontend

Слой `src/telegram`: определение режима по `window.Telegram.WebApp.initData`, обмен `initData` на токен при запуске, тема Telegram (`--tg-theme-*` → CSS-переменные Bootstrap), `BackButton` и `MainButton`, язык из `initDataUnsafe.user.language_code`. Режим (web или tma) хранится в контексте `app/`, а не в Redux (см. `frontend/CONVENTIONS.md` §2.2). Маршрут `/me` — страница участника в Mini App. Компоненты фич не знают режима, кроме кнопок навигации.

### T5. Данные

Миграция: `participants.telegram_user_id` (уникальный, nullable), `participants.channel` (`web` по умолчанию для существующих), таблица `telegram_link_codes`, у `notifications` — `attempts`, `next_attempt_at`, `fail_reason`. При переходе на PostgreSQL `telegram_user_id` — `bigint`.

## Risks / Trade-offs

- [Утечка токена бота даёт подделку initData] → токен только в переменной окружения, ротация через BotFather при подозрении.
- [Пользователь без username] → Mini App объясняет, как его задать (spec telegram-access).
- [Telegram недоступен] → веб работает без Telegram, уведомления падают в ручную очередь.
