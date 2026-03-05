# API.md (Sea Patrol — unified)

## Назначение
Сводный документ по контрактам **между** `sea_patrol_frontend` и `sea_patrol_backend`.

Каноника по реализации:
- Backend: `sea_patrol_backend/ai-docs/API_INFO.md`
- Frontend: `sea_patrol_frontend/ai-docs/API_INFO.md` (ожидания/адаптеры клиента)

Если возникают расхождения, приоритет у фактической реализации backend + того, что реально обрабатывает frontend-код.

## Base URLs (локально)
- HTTP: `http://localhost:8080`
- WebSocket: `ws://localhost:8080`

## REST: Auth (`/api/v1/auth/*`)

### `POST /api/v1/auth/signup`
Request JSON:
```json
{ "username": "alice", "password": "secret", "email": "alice@example.com" }
```

Текущее поведение backend (по `sea_patrol_backend/ai-docs/API_INFO.md`):
- `200 OK`
- Response JSON: `{ "username": "alice" }`

Frontend-код (`sea_patrol_frontend/src/shared/api/authApi.js`) воспринимает любой `2xx` как успех и возвращает `data` как есть.

### `POST /api/v1/auth/login`
Request JSON:
```json
{ "username": "alice", "password": "secret" }
```

Текущее поведение backend:
- `200 OK`
- Response JSON:
```json
{ "username": "alice", "token": "<jwt>", "issuedAt": "...", "expiresAt": "..." }
```

Frontend-ожидания:
- `token` используется для сессии и WebSocket.
- `username` используется для user state.
- `userId` в текущем frontend-коде опционален (может быть `undefined`).

### Ошибки (текущее расхождение)
Backend возвращает ошибки в виде:
```json
{ "errors": [{ "code": "SEAPATROL_...", "message": "..." }] }
```

Frontend сейчас пытается читать `data.message` и при backend-формате часто покажет общий текст `Request failed`.

Рекомендация для унификации (как отдельная задача):
- либо привести backend к `{ "message": "..." }` (минимум) в auth-ошибках,
- либо научить frontend извлекать сообщение из `errors[0].message` (и при необходимости код).

## Rooms (planned MVP)

Требование: в лобби уже есть WebSocket (чат), поэтому список комнат и их заполненность удобнее получать через WS без polling.

### WebSocket: rooms state (planned MVP)

Server → Client:
- `ROOMS_SNAPSHOT` отправляется автоматически при WS‑подключении (пользователь в состоянии `lobby`).
- `ROOMS_SNAPSHOT` payload (пример):
```json
{
  "maxRooms": 5,
  "maxPlayersPerRoom": 100,
  "rooms": [
    { "id": "sandbox-1", "name": "Sandbox 1", "currentPlayers": 10, "maxPlayers": 100, "status": "OPEN" },
    { "id": "sandbox-2", "name": "Sandbox 2", "currentPlayers": 100, "maxPlayers": 100, "status": "FULL" }
  ]
}
```
- `ROOMS_UPDATED` payload: либо полный список (как `ROOMS_SNAPSHOT`), либо delta-патч (определим позже; для MVP можно слать полный список).

Client → Server (опционально, если не шлём snapshot автоматически):
```json
["ROOMS_LIST", {}]
```

### REST: create room (planned MVP)

Создание комнаты оставляем через REST (удобно для кнопки "Create" и стандартной авторизации).

Аутентификация: требуется `Authorization: Bearer <jwt>`.

#### `POST /api/v1/rooms`
Создаёт новую песочницу, если не достигнут `maxRooms`.

Request JSON (опционально):
```json
{ "name": "Sandbox 3" }
```

Response `201 Created` JSON (пример):
```json
{ "id": "sandbox-3", "name": "Sandbox 3", "currentPlayers": 0, "maxPlayers": 100, "status": "OPEN" }
```

Ошибки (черновик):
- `409` — достигнут `maxRooms`.

### Join flow (planned MVP)Т.к. в лобби уже есть WebSocket для чата лобби (`to="group:lobby"`), присоединение к игровой комнате планируется делать через WS-команду (см. WS section):
- REST используется для `list/create`,
- WS — для фактического перехода игрока в комнату (чтобы не разрывать соединение чата).

## WebSocket: Game (`/ws/game`)

### Подключение и авторизация
Endpoint: `ws://localhost:8080/ws/game?token=<jwt>`

### Форматы сообщений
Frontend умеет **получать** оба формата:
1) Tuple: `["TYPE", payload]`
2) Object: `{ "type": "TYPE", "payload": payload }`

Frontend **отправляет** tuple.

Backend (по документации) ожидает входящие tuple и отправляет object.

### Message types (пересечение по текущим описаниям)
- Chat: `CHAT_MESSAGE`, `CHAT_JOIN`, `CHAT_LEAVE`
- Game: `PLAYER_INPUT`, `PLAYER_JOIN`, `PLAYER_LEAVE`, `INIT_GAME_STATE`, `UPDATE_GAME_STATE`

### Planned (MVP): Rooms / lobby + join

Требование: в лобби уже есть WebSocket для чата, поэтому подключение к WS происходит **до** выбора комнаты.

Планируемое поведение сервера:
- при подключении WS пользователь находится в состоянии `lobby` (чат доступен, игровая комната не выбрана);
- после выбора комнаты в лобби (REST) клиент отправляет WS-команду `ROOM_JOIN`.

Client → Server (tuple):
```json
["ROOM_JOIN", { "roomId": "sandbox-1" }]
```

Server → Client:
- `ROOM_JOINED` payload: `{ roomId, currentPlayers, maxPlayers }` (после этого сервер шлёт `INIT_GAME_STATE` выбранной комнаты).
- `ROOM_JOIN_REJECTED` payload: `{ roomId, reason }` (`FULL`, `NOT_FOUND`, `ERROR`).

Чаты (MVP):
- Чат лобби (изолирован от комнат): `to="group:lobby"`.
  - сервер автоматически добавляет пользователя в `group:lobby` при WS‑подключении.
- Чат комнаты (изолирован по roomId): `to="group:room:<roomId>"`.
  - при `ROOM_JOINED` сервер переводит пользователя из `group:lobby` в `group:room:<roomId>`.
  - опционально (план): команда `ROOM_LEAVE` возвращает игрока в лобби.

### Ключевые payload (сводно)
- `PLAYER_INPUT`: `{ left: boolean, right: boolean, up: boolean, down: boolean }`
- `CHAT_MESSAGE`:
  - frontend → backend: `{ to: "group:lobby|group:room:<roomId>|group:<id>|user:<username>", text: string }`
  - backend → frontend: минимум `{ from: string, text: string }`
- `INIT_GAME_STATE`: room/wind/players (backend-описание шире; frontend применяет минимум `players[]`)
- `UPDATE_GAME_STATE`: patch-обновления по игрокам (frontend применяет только присутствующие поля)


### Planned (MVP): Cannon fire / projectiles

Каноничное решение для MVP: **server-authoritative**, снаряд симулируется в плоскости XZ как логическая сущность; попадания вычисляются через Box2D **ray/segment cast** на отрезке `prevPos → nextPos` (берётся первое пересечение, без "пробития").

#### Client → Server
Tuple:
```json
["FIRE_CANNON", { "side": "PORT|STARBOARD", "aimOffsetRad": 0.12 }]
```
- `side`: левый/правый борт относительно курса корабля.
- `aimOffsetRad`: отклонение от строго борта (перпендикуляра к курсу); `aimOffsetRad > 0` = «чуть к носу (вперёд по курсу)». Сервер клампит и добавляет разброс.
- Дальность/скорость/разброс/кол-во ядер в залпе задаёт **сервер** (включая джиттер скорости — "порох").

#### Server → Clients
События (без стриминга траектории):
- `PROJECTILES_SPAWN` — залп (массив снарядов), включает `origin/dir/speed/ttlMs/spawnAt`.
- `PROJECTILE_HIT` — подтверждение попадания (`hitPos`, цель, урон).
- `PROJECTILE_DESPAWN` — исчезновение по TTL (если не попало).

Координата Y в MVP: сервер её не считает; клиент рисует косметическую дугу и обрывает по `PROJECTILE_HIT`.

## Контрольные ссылки (где править, если меняется контракт)
- Backend REST/WS каноника: `sea_patrol_backend/ai-docs/API_INFO.md`
- Frontend ожидания/адаптеры: `sea_patrol_frontend/ai-docs/API_INFO.md`, `sea_patrol_frontend/src/shared/ws/messageAdapter.js`, `sea_patrol_frontend/src/features/auth/model/AuthContext.jsx`










