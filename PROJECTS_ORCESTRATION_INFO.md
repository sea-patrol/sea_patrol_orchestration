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
- `GET /api/v1/rooms` уже реализован на backend как защищённый snapshot endpoint;
- `GET /api/v1/rooms` и WS-события `ROOMS_SNAPSHOT` / `ROOMS_UPDATED` используют один и тот же полный snapshot payload;
- backend уже отправляет `ROOMS_SNAPSHOT` при lobby WS-connect и `ROOMS_UPDATED` после `create`, `join`, `leave`, cleanup без polling;
- пустая комната удаляется из room catalog не сразу: после того, как в ней не остаётся активных игроков и завершается reconnect grace последнего room-bound пользователя, backend ещё выдерживает отдельный empty-room idle timeout;
- `POST /api/v1/rooms` уже реализован на backend и принимает опциональные `name` и `mapId`, а в ответе room catalog обязательно содержит `mapId` и `mapName`;
- текущий frontend UX может сразу после successful `POST /api/v1/rooms` запускать `POST /api/v1/rooms/{roomId}/join`, поэтому create room и room entry уже могут быть одной пользовательской операцией без отдельного промежуточного шага в UI;
- если имя не передано, backend генерирует `sandbox-N` / `Sandbox N`; если имя передано, `id` slugify-ится, а `name` остаётся display label;
- `mapId` и `mapName` уже резолвятся через in-memory `MapTemplateRegistry`, который валидирует полный world package из `src/main/resources/worlds/*`;
- в текущем production bundle зарегистрированы `caribbean-01` и `test-sandbox-01`; первая остаётся default map, а вторая уже доступна для dev/debug room creation без изменения внешнего room contract;
- `POST /api/v1/rooms/{roomId}/join` уже реализован на backend как защищённый room admission endpoint;
- backend стартует `/ws/game` в состоянии `lobby`, автоматически добавляет пользователя в `group:lobby`, после успешного REST join переводит сессию в `group:room:<roomId>` и server-authoritatively удерживает public chat isolation между lobby и rooms;
- единственный канонический room enter flow для MVP: `POST /api/v1/rooms/{roomId}/join` -> `ROOM_JOINED` -> `SPAWN_ASSIGNED` -> `INIT_GAME_STATE`;
- room-menu flow уже зафиксирован и реализован на backend: `POST /api/v1/rooms/{roomId}/leave` -> REST `200 { roomId, status: LEFT, nextState: LOBBY }` -> backend rebinding той же WS-сессии обратно в `lobby` -> `ROOMS_SNAPSHOT`;
- по состоянию на `TASK-035C` frontend тоже уже использует этот flow из room menu: после confirmed REST success он сразу очищает stale room state и переводит пользователя на `/lobby` без logout, а `LobbyPage` дополнительно не пытается резюмить старую комнату, если пользователь пришёл туда через explicit `Exit to lobby`;
- альтернативного WS-only flow для `join` и альтернативного client-side spawn flow в канонике MVP нет;
- initial spawn для current runtime уже вычисляется backend'ом из `MapTemplate.spawnPoints` и `spawnRules.playerSpawnRadius`, проверяется по `MapTemplate.bounds`, а transport shape `SPAWN_ASSIGNED` переиспользуется и для server-side respawn path с `reason=RESPAWN`.
- `INIT_GAME_STATE` теперь несёт не только legacy `room`, но и map-driven `roomMeta`, собранный из room runtime и `MapTemplate`.
- для `Wave 4` каноника `wind` теперь тоже фиксируется на уровне orchestration: payload имеет вид `{ angle, speed }`, `angle` хранится в радианах в плоскости `XZ` (`0 -> +X`, `PI / 2 -> +Z`), `wind` приходит и в `INIT_GAME_STATE`, и в `UPDATE_GAME_STATE`, а клиент не должен симулировать собственный authoritative ветер.
- по состоянию на `TASK-031` backend уже держит этот `wind` как room-level authoritative state в `GameRoom` и рассылает один и тот же snapshot всем игрокам комнаты через `INIT_GAME_STATE` и `UPDATE_GAME_STATE`.
- по состоянию на `TASK-032` frontend уже поднимает backend `wind` в `GameStateContext`, поэтому transport реально доходит до клиентского runtime state, а не остаётся неиспользованным полем в payload.
- по состоянию на `TASK-033` backend ship movement тоже уже зависит от room wind state: ускорение судна рассчитывается server-authoritatively с учётом силы ветра и относительного угла между курсом корабля и направлением ветра.
- по состоянию на `TASK-033B` backend уже реализует канонику `sailLevel`: это server-authoritative дискретное состояние `0..3`, где `0` означает полностью убранные паруса, `3` — полностью поднятые, а `PLAYER_INPUT.up/down` управляют именно уровнем парусов по rising-edge, а не прямым throttle/brake.
- `sailLevel` уже приходит клиенту как часть player state в `INIT_GAME_STATE` и `UPDATE_GAME_STATE`, а по состоянию на `TASK-033C` frontend уже поднимает это поле в `GameStateContext` и показывает его в HUD без отдельной локальной authoritative модели парусов.
- по состоянию на `TASK-034` frontend также уже даёт игроку более понятный wind HUD feedback: показываются сила и направление ветра, относительное положение ветра к текущему курсу корабля и краткая подсказка, почему ход судна сейчас выглядит сильным или слабым.
- по состоянию на `TASK-035` предсказуемое вращение ветра по часовой стрелке уже стало backend-authoritative runtime behavior: угол меняется в комнате с фиксированной скоростью из backend config, а фронт по-прежнему просто применяет последний snapshot, пришедший от backend.
- Backend также уже держит in-memory static catalogs из `src/main/resources/catalogs/*.json`: `ship classes`, `items`, `merchants`, `quests`; это заготовка под cargo/trade/quest flows без отдельной БД.
- По состоянию на `TASK-027` основные игровые flow всё ещё сознательно работают без `Liquibase`/`H2`, только на process memory + resource files.

Что это означает для следующих runtime-задач:
1. Backend room endpoints и WS events должны реализовываться ровно под этот contract.
2. Frontend lobby UI должен считать `ROOMS_UPDATED` полным snapshot, а не delta-патчем.
3. Frontend room init должен ждать `SPAWN_ASSIGNED` как authoritative spawn source.
4. Frontend reconnect UI уже может опираться на backend room resume в пределах `game.room.reconnect-grace-period` (MVP default: `15s`): при reconnect backend восстанавливает ту же room binding и повторно шлёт `ROOM_JOINED` + `INIT_GAME_STATE`, а после истечения окна переводит пользователя в новый lobby session flow.
5. Room menu leave flow должен оставаться REST-authoritative: frontend не должен «угадывать» выход из комнаты локально без успешного `POST /api/v1/rooms/{roomId}/leave`.

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
















