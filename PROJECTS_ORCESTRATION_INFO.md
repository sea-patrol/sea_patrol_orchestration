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
Auth contract для MVP зафиксирован по фактической backend-реализации:
- `POST /api/v1/auth/signup` -> `200 OK` + `{ username }`
- `POST /api/v1/auth/login` -> `{ username, token, issuedAt, expiresAt }`
- auth/validation/security errors -> `{ errors: [{ code, message }] }`
- backend single-session policy дополняет этот контракт: active game WS-session блокирует повторный `login` и второе параллельное WS-подключение тем же пользователем

Room/lobby contract для MVP теперь тоже зафиксирован на уровне orchestration:
- lobby screen использует гибридный flow: первичный `GET /api/v1/rooms` + lobby WebSocket для live-обновлений;
- `GET /api/v1/rooms` и WS-события `ROOMS_SNAPSHOT` / `ROOMS_UPDATED` используют один и тот же полный snapshot payload;
- `POST /api/v1/rooms` принимает опциональные `name` и `mapId`, а в ответе room catalog обязательно содержит `mapId` и `mapName`;
- единственный канонический room enter flow для MVP: `POST /api/v1/rooms/{roomId}/join` -> `ROOM_JOINED` -> `SPAWN_ASSIGNED` -> `INIT_GAME_STATE`;
- альтернативного WS-only flow для `join` и альтернативного client-side spawn flow в канонике MVP нет.

Что это означает для следующих runtime-задач:
1. Backend room endpoints и WS events должны реализовываться ровно под этот contract.
2. Frontend lobby UI должен считать `ROOMS_UPDATED` полным snapshot, а не delta-патчем.
3. Frontend room init должен ждать `SPAWN_ASSIGNED` как authoritative spawn source.
4. Frontend reconnect UI нельзя строить на предположении, что backend уже умеет full room resume: сейчас реализован только single-session admission + reconnect grace window.

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





