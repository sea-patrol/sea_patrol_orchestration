# TASK-035A - Канонический room leave / menu contract

## Метаданные
- **ID:** `TASK-035A`
- **Тип:** `docs`
- **Статус:** `Done`
- **Приоритет:** `High`
- **Дата создания:** `2026-03-13`
- **Автор:** `Codex`
- **Связанный roadmap item:** `Wave 4.5 / TASK-035A`
- **Затронутые репозитории:** `sea_patrol_orchestration`, `sea_patrol_backend`, `sea_patrol_frontend`
- **Связанные задачи / PR:** `TASK-035B`, `TASK-035C`, `TASK-035D`

## Контекст
После `Wave 1`..`Wave 4` у нас уже есть канонический flow входа в комнату, reconnect и lobby/game navigation, но не было такой же однозначной договорённости по штатному выходу из комнаты через игровое меню обратно в лобби.

## Цель
Зафиксировать единый `leave room -> lobby` contract до runtime-реализации, чтобы backend и frontend одинаково понимали endpoint, response shape, WS rebinding и ожидаемое поведение `RoomSession`.

## Source of Truth
- Документация:
  - `sea_patrol_orchestration/API.md`
  - `sea_patrol_orchestration/PROJECTS_ORCESTRATION_INFO.md`
  - `sea_patrol_orchestration/ROADMAP-TASKS.md`
  - `sea_patrol_backend/ai-docs/API_INFO.md`
  - `sea_patrol_frontend/ai-docs/API_INFO.md`

## Acceptance Criteria
- [x] В orchestration docs зафиксирован канонический `POST /api/v1/rooms/{roomId}/leave`.
- [x] Нет двусмысленности по тому, остаётся ли та же auth/WS session.
- [x] Ясно указано, что success/failure остаётся REST-authoritative, а первый lobby snapshot после leave приходит как `ROOMS_SNAPSHOT`.
- [x] Backend/frontend docs синхронизированы и честно помечают, что это ещё не runtime-реализация.

## Scope
**Включает:**
- фиксацию REST endpoint и response shape;
- фиксацию room->lobby WS rebinding semantics;
- фиксацию ожидаемого поведения chat scope и `RoomSession`;
- синхронизацию orchestration/backend/frontend API docs;
- пометку roadmap item как выполненного contract-first шага.

**Не включает (out of scope):**
- backend runtime implementation leave-room flow;
- frontend menu modal implementation;
- debug toggle implementation;
- новые WS message types beyond already agreed `ROOMS_SNAPSHOT`.

## Каноника
### REST
`POST /api/v1/rooms/{roomId}/leave`

Request:
```json
{}
```

Response `200 OK`:
```json
{
  "roomId": "sandbox-1",
  "status": "LEFT",
  "nextState": "LOBBY"
}
```

Ошибки:
- `404` -> `ROOM_NOT_FOUND`
- `409` -> `ROOM_SESSION_REQUIRED`
- `409` -> `ROOM_SESSION_MISMATCH`

### WS / session semantics
- auth session не рвётся;
- та же активная WS-сессия переводится из room binding обратно в `lobby`;
- chat membership меняется из `group:room:<roomId>` обратно в `group:lobby`;
- backend отправляет `ROOMS_SNAPSHOT` как первый authoritative lobby snapshot после успешного leave;
- отдельный `ROOM_LEFT` message type в MVP не требуется.

### Frontend semantics
- menu action `Exit` не является logout;
- после REST success frontend очищает stale room state и переводит пользователя на `/lobby`;
- lobby runtime продолжает жить на той же auth/WS session и подтверждается первым `ROOMS_SNAPSHOT`.

## Проверки
### Автоматические проверки
| Репозиторий | Команда | Зачем | Статус |
|-------------|---------|-------|--------|
| `sea_patrol_backend` | `.\gradlew.bat test` | Не требовалось: runtime-код не менялся | `N/A` |
| `sea_patrol_frontend` | `npm run test:run` | Не требовалось: runtime-код не менялся | `N/A` |

### Ручная проверка
- [x] Документация синхронизирована
- [x] Явно отделён contract-first шаг от следующих runtime задач

## QA / Review
### QA
- [x] Acceptance criteria подтверждены
- [x] Документы не противоречат текущему join/reconnect contract

**QA статус:** `Passed`

### Code Review
| Приоритет | Комментарий | Статус |
|-----------|-------------|--------|
| `Low` | Оставили REST-authoritative success path без нового `ROOM_LEFT`, чтобы не плодить лишний protocol surface перед runtime-реализацией | `Resolved` |

**Review решение:** `Approve`

## Финализация
- [x] Contract leave-room flow зафиксирован
- [x] Backend/frontend/orchestration docs синхронизированы
- [x] Задача помечена как выполненная в roadmap
