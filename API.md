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

Канонический auth contract для MVP фиксируется по фактической backend-реализации и текущим backend tests.

### `POST /api/v1/auth/signup`
Request JSON:
```json
{ "username": "alice", "password": "secret", "email": "alice@example.com" }
```

Response `200 OK`:
```json
{ "username": "alice" }
```

Правила для каноники MVP:
- `201 Created` c `{ id, username, email }` не является текущей каноникой.
- ответ содержит только `username`;
- duplicate username policy пока не зафиксирована как часть канонического MVP contract и будет уточняться отдельно в backend task.

### `POST /api/v1/auth/login`
Request JSON:
```json
{ "username": "alice", "password": "secret" }
```

Response `200 OK`:
```json
{ "username": "alice", "token": "<jwt>", "issuedAt": "...", "expiresAt": "..." }
```

Правила для каноники MVP:
- `token` используется для HTTP session и WebSocket подключения;
- `username` используется как минимальный user identity field;
- `userId` в канонический ответ MVP не входит и не должен считаться обязательным на frontend.

### Ошибки auth / validation / security
Канонический формат ошибок для auth-related сценариев:
```json
{ "errors": [{ "code": "SEAPATROL_INVALID_PASSWORD", "message": "Invalid password" }] }
```

Применение:
- `401` — auth / unauthorized / invalid credentials / invalid JWT;
- `400` — validation errors и прочие `ApiException` bad request cases.

Дополнительное правило для текущего backend runtime:
- `POST /api/v1/auth/login` может вернуть `401` + `{ "errors": [{ "code": "SEAPATROL_DUPLICATE_SESSION", "message": "Active game session already exists" }] }`, если у пользователя уже есть активная игровая WebSocket-сессия.

Правила для каноники MVP:
- корневой `message` без `errors[]` не является каноническим форматом auth-ошибок;
- frontend должен уметь извлекать человекочитаемое сообщение из `errors[0].message`;
- backend code и backend tests считаются source of truth для текущего формата до отдельных согласованных изменений.

## Rooms (MVP contract)

Требование: лобби использует гибридную схему загрузки.
- При открытии страницы лобби frontend делает первичный REST-запрос за текущим списком комнат.
- Параллельно frontend подключает WebSocket для чата лобби и дальнейших live-обновлений списка комнат.
- После первичной загрузки все изменения room catalog приходят через WS без polling.

### Room summary payload (каноника MVP)

Минимальная модель комнаты для каталога и create response:
```json
{
  "id": "sandbox-1",
  "name": "Sandbox 1",
  "mapId": "caribbean-01",
  "mapName": "Caribbean Sea",
  "currentPlayers": 10,
  "maxPlayers": 100,
  "status": "OPEN"
}
```

Правила:
- `mapId` и `mapName` обязательны в room catalog contract;
- для room catalog в MVP достаточно статусов `OPEN` и `FULL`;
- `ROOMS_SNAPSHOT`, `ROOMS_UPDATED` и `GET /api/v1/rooms` используют один и тот же payload shape.

### REST: rooms list

Аутентификация: требуется `Authorization: Bearer <jwt>`.

#### `GET /api/v1/rooms`
Возвращает актуальный список комнат на момент открытия страницы лобби. Этот endpoint уже реализован на backend.

Response `200 OK` JSON (пример):
```json
{
  "maxRooms": 5,
  "maxPlayersPerRoom": 100,
  "rooms": [
    {
      "id": "sandbox-1",
      "name": "Sandbox 1",
      "mapId": "caribbean-01",
      "mapName": "Caribbean Sea",
      "currentPlayers": 10,
      "maxPlayers": 100,
      "status": "OPEN"
    },
    {
      "id": "sandbox-2",
      "name": "Sandbox 2",
      "mapId": "caribbean-01",
      "mapName": "Caribbean Sea",
      "currentPlayers": 100,
      "maxPlayers": 100,
      "status": "FULL"
    }
  ]
}
```

Назначение:
- используется frontend при открытии страницы лобби как первичный snapshot;
- не заменяет WebSocket-обновления;
- нужен для устойчивого первого рендера lobby UI и для сценариев reconnect/reload страницы.

Текущее backend-ограничение:
- до `TASK-025` room catalog использует временное default map metadata `caribbean-01` / `Caribbean Sea` для всех активных комнат.

### WebSocket: rooms state (planned MVP)

Server -> Client:
- `ROOMS_SNAPSHOT` может отправляться автоматически при WS-подключении пользователя в состоянии `lobby`.
- `ROOMS_SNAPSHOT` допустимо использовать как повторную синхронизацию после подключения или реконнекта.
- `ROOMS_UPDATED` используется для live-обновлений room catalog после `create`, `join`, `leave`, cleanup.
- Для MVP `ROOMS_UPDATED` не является delta-патчем: payload всегда равен полному snapshot той же формы, что и `GET /api/v1/rooms`.

`ROOMS_SNAPSHOT` / `ROOMS_UPDATED` payload (пример):
```json
{
  "maxRooms": 5,
  "maxPlayersPerRoom": 100,
  "rooms": [
    {
      "id": "sandbox-1",
      "name": "Sandbox 1",
      "mapId": "caribbean-01",
      "mapName": "Caribbean Sea",
      "currentPlayers": 10,
      "maxPlayers": 100,
      "status": "OPEN"
    }
  ]
}
```

Client -> Server (опционально, если snapshot не шлётся автоматически):
```json
["ROOMS_LIST", {}]
```

### REST: create room

Создание комнаты остаётся через REST. Этот endpoint уже реализован на backend.

Аутентификация: требуется `Authorization: Bearer <jwt>`.

#### `POST /api/v1/rooms`
Создаёт новую песочницу, если не достигнут `maxRooms`.

Request JSON (все поля опциональны):
```json
{ "name": "Sandbox 3", "mapId": "caribbean-01" }
```

Правила:
- если `name` не передан, backend генерирует следующий `sandbox-N` и display name `Sandbox N`;
- если `name` передан, backend строит `id` как slugified-форму имени, а `name` оставляет display label;
- если `mapId` не передан, backend использует дефолтную карту MVP;
- до `TASK-025` backend принимает только `mapId = caribbean-01` и валидирует остальные значения как `INVALID_MAP_ID`.

Response `201 Created` JSON (пример):
```json
{
  "id": "sandbox-3",
  "name": "Sandbox 3",
  "mapId": "caribbean-01",
  "mapName": "Caribbean Sea",
  "currentPlayers": 0,
  "maxPlayers": 100,
  "status": "OPEN"
}
```

Ошибки (MVP):
- `400` -> `{ "errors": [{ "code": "INVALID_MAP_ID", "message": "Unknown mapId" }] }`
- `409` -> `{ "errors": [{ "code": "MAX_ROOMS_REACHED", "message": "Maximum number of rooms reached" }] }`

## WebSocket: Game (`/ws/game`)

### Подключение и авторизация
Endpoint: `ws://localhost:8080/ws/game?token=<jwt>`

Session policy:
- backend допускает только одну активную игровую WS-сессию на пользователя;
- параллельное второе подключение отклоняется закрытием `POLICY_VIOLATION`, reason содержит `SEAPATROL_DUPLICATE_SESSION`;
- reconnect в течение `game.room.reconnect-grace-period` допускается на уровне session admission;
- полный resume room state не входит в текущую канонику `TASK-006` и остается отдельной backend задачей.

### Форматы сообщений
Frontend умеет **получать** оба формата:
1) Tuple: `["TYPE", payload]`
2) Object: `{ "type": "TYPE", "payload": payload }`

Frontend **отправляет** tuple.

Backend (по документации) ожидает входящие tuple и отправляет object.

### Message types (пересечение по текущим описаниям)
- Chat: `CHAT_MESSAGE`, `CHAT_JOIN`, `CHAT_LEAVE`
- Lobby/rooms: `ROOMS_SNAPSHOT`, `ROOMS_UPDATED`, `ROOM_JOINED`, `ROOM_JOIN_REJECTED`
- Game: `PLAYER_INPUT`, `PLAYER_JOIN`, `PLAYER_LEAVE`, `SPAWN_ASSIGNED`, `INIT_GAME_STATE`, `UPDATE_GAME_STATE`

### Rooms / lobby + join

Требование: в лобби уже есть WebSocket для чата, поэтому подключение к WS происходит **до** выбора комнаты.

Текущее backend-поведение:
- при подключении WS пользователь находится в состоянии `lobby` (чат доступен, игровая комната не выбрана);
- lobby chat использует `to="group:lobby"`;
- после выбора комнаты клиент делает только REST-запрос `POST /api/v1/rooms/{roomId}/join`;
- backend валидирует room admission и наличие активной lobby WS-session;
- после успешного `join` backend переводит текущую WS-сессию пользователя из `group:lobby` в `group:room:<roomId>`.

Канонический initial join flow для MVP:
1. Клиент открывает лобби: `GET /api/v1/rooms` и lobby WS могут стартовать параллельно.
2. Пользователь нажимает `join` в UI лобби.
3. Клиент вызывает `POST /api/v1/rooms/{roomId}/join`.
4. Backend валидирует комнату и lobby WS binding.
5. При успехе backend возвращает REST `200 OK`, затем по уже открытому WS отправляет `ROOM_JOINED`.
6. Затем backend вычисляет spawn и отправляет `SPAWN_ASSIGNED`.
7. Затем backend отправляет `INIT_GAME_STATE` комнаты и дальнейшие room updates.

Альтернативного WS-only flow для `join` в канонике MVP нет.

### REST: room join

Аутентификация: требуется `Authorization: Bearer <jwt>`.

#### `POST /api/v1/rooms/{roomId}/join`
Этот endpoint уже реализован на backend.
Подтверждает вход игрока в комнату и переключает его активную WS-сессию из лобби в потоки комнаты.

Request body:
```json
{}
```

Response `200 OK` JSON (пример):
```json
{
  "roomId": "sandbox-1",
  "mapId": "caribbean-01",
  "mapName": "Caribbean Sea",
  "currentPlayers": 11,
  "maxPlayers": 100,
  "status": "JOINED"
}
```

Ошибки (MVP):
- `404` -> `{ "errors": [{ "code": "ROOM_NOT_FOUND", "message": "Room not found" }] }`
- `409` -> `{ "errors": [{ "code": "ROOM_FULL", "message": "Room is full" }] }`
- `409` -> `{ "errors": [{ "code": "LOBBY_SESSION_REQUIRED", "message": "Active lobby WebSocket session is required" }] }`

Server -> Client по уже открытому WS:
- `ROOM_JOINED` payload повторяет успешный REST response:
```json
{
  "roomId": "sandbox-1",
  "mapId": "caribbean-01",
  "mapName": "Caribbean Sea",
  "currentPlayers": 11,
  "maxPlayers": 100,
  "status": "JOINED"
}
```
- `ROOM_JOIN_REJECTED` payload (пример, тип зарезервирован в протоколе):
```json
{ "roomId": "sandbox-1", "reason": "FULL" }
```

Правила для `ROOM_JOIN_REJECTED`:
- событие используется только как WS-уведомление о несогласованности или отказе текущей сессии;
- для MVP допустимые `reason`: `FULL`, `NOT_FOUND`, `LOBBY_SESSION_REQUIRED`;
- в текущей backend-реализации `TASK-011` authoritative отказ приходит через REST error response, а отдельное WS-событие `ROOM_JOIN_REJECTED` пока не отправляется.

Чаты (MVP):
- Чат лобби (изолирован от комнат): `to="group:lobby"`.
  - сервер автоматически добавляет пользователя в `group:lobby` при WS-подключении.
- Чат комнаты (изолирован по `roomId`): `to="group:room:<roomId>"`.
  - после успешного REST `join` сервер переводит пользователя из `group:lobby` в `group:room:<roomId>`.
  - отдельный `leave` flow не входит в канонику `TASK-004`.

### Spawn assignment

Server -> Client:
- `SPAWN_ASSIGNED`

Payload (пример):
```json
{
  "roomId": "sandbox-1",
  "reason": "INITIAL",
  "x": 12.5,
  "z": -8.0,
  "angle": 0.0
}
```

Правила:
- spawn/respawn координаты определяет только backend;
- клиент не вычисляет spawn самостоятельно;
- `reason` в MVP фиксируется как `INITIAL | RESPAWN`;
- initial spawn flow фиксируется как `ROOM_JOINED -> SPAWN_ASSIGNED -> INIT_GAME_STATE`;
- respawn flow фиксируется как `SPAWN_ASSIGNED(reason=RESPAWN) -> дальнейший room snapshot/update`;
- клиент не должен переводить локальный корабль в игровую комнату до получения `SPAWN_ASSIGNED`;
- в текущем backend runtime initial `SPAWN_ASSIGNED` отправляется с placeholder coordinates `(0.0, 0.0, 0.0)` до отдельных задач по spawn logic.

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









