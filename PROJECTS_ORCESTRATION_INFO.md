# PROJECTS_ORCESTRATION_INFO.md

## Назначение
Этот документ объединяет знания о **двух репозиториях** Sea Patrol внутри оркестрации (`sea_patrol_backend` и `sea_patrol_frontend`): что где лежит, как запускать локально и где находятся каноничные описания.

> Примечание по имени файла: используется написание `ORCESTRATION` (как в запросе/истории проекта).

## Состав системы

### Backend — `sea_patrol_backend`
- Стек: Spring Boot 4.0.3 + WebFlux + Spring Security + JWT, Java 25, Gradle (Kotlin DSL).
- Реалтайм: WebSocket endpoint `/ws/game`.
- Игровая логика/физика: LibGDX + Box2D + gdx-ai.
- Хранилище пользователей: in-memory.
- Может отдавать собранный фронтенд из `src/main/resources/static`.

Каноничные документы backend:
- `sea_patrol_backend/AGENTS.md`
- `sea_patrol_backend/ai-docs/PROJECT_INFO.md`
- `sea_patrol_backend/ai-docs/API_INFO.md`

### Frontend — `sea_patrol_frontend`
- Стек: React 19 + Vite 7 + Three.js + React Three Fiber (+ Drei).
- Реалтайм: WebSocket клиент + контекст.
- Тесты: Vitest + React Testing Library + MSW + mock-socket.
- PWA: vite-plugin-pwa.

Каноничные документы frontend:
- `sea_patrol_frontend/AGENTS.md`
- `sea_patrol_frontend/ai-docs/PROJECT_INFO.md`
- `sea_patrol_frontend/ai-docs/API_INFO.md`

## Локальный запуск (рекомендуемый dev-режим)

### 1) Backend (порт 8080)
Backend требует секрет для JWT.

Переменные окружения (достаточно одной):
- `JWT_SECRET` — raw строка (рекомендуется >= 32 байта)
- `JWT_SECRET_BASE64` — base64/base64url (после декодирования >= 32 байта)

Команды (Windows):
- `cd sea_patrol_backend`
- `.\gradlew.bat bootRun`

### 2) Frontend (порт 5173)
Команды:
- `cd sea_patrol_frontend`
- `npm install`
- `npm run dev`

По умолчанию фронтенд пытается достучаться до backend на текущем hostname + порт `8080`.

Опционально можно задать:
- `VITE_API_BASE_URL` (пример: `http://localhost:8080`)
- `VITE_WS_BASE_URL` (пример: `ws://localhost:8080`)

## Интеграционные точки (что должно совпадать)

### HTTP
- Base URL (локально): `http://localhost:8080`
- Auth endpoints: `POST /api/v1/auth/signup`, `POST /api/v1/auth/login`

### WebSocket
- Endpoint: `/ws/game`
- Авторизация: query-параметр `token`, пример: `ws://localhost:8080/ws/game?token=<jwt>`
- Форматы сообщений:
  - frontend → backend: tuple `['TYPE', payload]`
  - backend → frontend: object `{ type, payload }` (frontend также умеет принимать tuple)

### CORS
Backend (по текущей конфигурации) разрешает origins для dev:
- `http://localhost:5173`
- `http://localhost:4173`

## Статус согласованности документации (важно для планирования)
Ниже — расхождения между описаниями (полезно учитывать, пока не приведём к единому контракту):

1) **Формат ошибок REST**
- Backend документирует формат `{ errors: [{ code, message }] }`.
- Frontend сейчас читает `data.message` и при backend-формате показывает общий текст `Request failed`.

2) **Login response**
- Backend: `{ username, token, issuedAt, expiresAt }`.
- Frontend ожидает минимум `token` и `username`, а также пытается прочитать `userId` (может быть `undefined`).

3) **Signup response / status**
- Backend: успешный ответ описан как `200` и `{ username }`.
- Frontend docs описывают `201` и `{ id, username, email }`.

Эти пункты — хорошие кандидаты для первых задач по унификации контрактов.

## Где в коде смотреть интеграцию
Frontend:
- REST auth: `sea_patrol_frontend/src/shared/api/authApi.js`
- Auth state: `sea_patrol_frontend/src/features/auth/model/AuthContext.jsx`
- WS адаптер: `sea_patrol_frontend/src/shared/ws/messageAdapter.js`
- WS client: `sea_patrol_frontend/src/shared/ws/wsClient.js`

Backend:
- Security/public routes: `sea_patrol_backend/src/main/java/ru/sea/patrol/config/WebSecurityConfig.java`
- WebSocket handler: `sea_patrol_backend/src/main/java/ru/sea/patrol/ws/game/GameWebSocketHandler.java`
- WS types/DTO: `sea_patrol_backend/src/main/java/ru/sea/patrol/ws/protocol/MessageType.java`

