# TASK-030 - Канонический wind contract для MVP

## Метаданные
- **ID:** `TASK-030`
- **Тип:** `docs`
- **Статус:** `Done`
- **Приоритет:** `High`
- **Дата создания:** `2026-03-12`
- **Автор:** `Codex`
- **Связанный roadmap item:** `Wave 4 — Ветер, паруса и движение корабля`
- **Затронутые репозитории:** `sea_patrol_orchestration`, `sea_patrol_backend`, `sea_patrol_frontend`
- **Связанные задачи / PR:** `TASK-031`, `TASK-032`, `TASK-035`

## Контекст
После `TASK-026` backend уже начал отдавать `wind` в room bootstrap, но в документации ещё не было одной явной каноники по payload shape, единицам измерения угла и тому, как клиент должен трактовать `INIT_GAME_STATE` и `UPDATE_GAME_STATE`.

## Цель
Зафиксировать единый `wind` contract для backend и frontend до начала runtime-задач по authoritative wind state и wind-driven ship movement.

## Source of Truth
- Код / тесты:
  - `sea_patrol_backend/src/main/java/ru/sea/patrol/service/game/Wind.java`
  - `sea_patrol_backend/src/main/java/ru/sea/patrol/service/game/GameRoom.java`
  - `sea_patrol_backend/src/main/resources/worlds/caribbean-01/manifest.json`
  - `sea_patrol_backend/src/main/resources/worlds/test-sandbox-01/manifest.json`
- Документация:
  - `sea_patrol_orchestration/API.md`
  - `sea_patrol_orchestration/PROJECTS_ORCESTRATION_INFO.md`
  - `sea_patrol_orchestration/ROADMAP-TASKS.md`
  - `sea_patrol_backend/ai-docs/API_INFO.md`
  - `sea_patrol_frontend/ai-docs/API_INFO.md`

## Acceptance Criteria
- [x] В orchestration docs нет двусмысленности по `wind` payload.
- [x] Backend/frontend docs одинаково трактуют `angle`, `speed` и место доставки `wind`.
- [x] Отдельно зафиксирована граница между текущим contract-level соглашением и будущей runtime policy из `TASK-035`.

## Scope
**Включает:**
- фиксацию payload shape `{ angle, speed }`;
- фиксацию семантики угла в плоскости `XZ`;
- фиксацию того, что `wind` приходит и в `INIT_GAME_STATE`, и в `UPDATE_GAME_STATE`;
- фиксацию того, что клиент применяет backend wind как authoritative snapshot.

**Не включает (out of scope):**
- изменение backend runtime поведения ветра;
- clockwise rotation implementation;
- применение ветра к движению корабля или HUD.

## Предпосылки и зависимости
- `TASK-026` уже добавил map-driven initial wind defaults и transport path `roomMeta + wind`.
- `TASK-031`, `TASK-032` и `TASK-035` опираются на эту канонику.

## Технический подход
Каноника выводится из текущего backend runtime:
- угол ветра хранится в радианах;
- направление вычисляется как `cos(angle)` / `sin(angle)` в плоскости `XZ`;
- `wind` уже является room-level transport field и должен документироваться как authoritative snapshot, а не как client-side derived state.

Clockwise policy вынесена как отдельное follow-up runtime behavior в `TASK-035`, чтобы не смешивать transport contract и будущую симуляцию.

## Изменения по репозиториям

### `sea_patrol_orchestration`
- [x] Обновить `API.md`
- [x] Обновить `PROJECTS_ORCESTRATION_INFO.md`
- [x] Обновить `ROADMAP-TASKS.md`
- [x] Обновить другие документы: `ai-docs/tasks/TASK-030.md`

### `sea_patrol_backend`
- [x] Обновить `ai-docs/API_INFO.md`

### `sea_patrol_frontend`
- [x] Обновить `ai-docs/API_INFO.md`

## Контракты и данные

### API / WebSocket
- `wind` transport shape: `{ "angle": number, "speed": number }`
- `angle` в радианах, плоскость `XZ`, `0 -> +X`, `PI / 2 -> +Z`
- `INIT_GAME_STATE.wind` — initial authoritative room snapshot
- `UPDATE_GAME_STATE.wind` — текущий authoritative room snapshot того же shape

### Данные / конфигурация / миграции
- Не менялись

## Риски и меры контроля
| Риск | Почему это риск | Мера контроля |
|------|-----------------|---------------|
| Clockwise policy будет воспринята как уже реализованная | В roadmap она ещё отдельной задачей | В docs явно указано, что runtime policy приходит только в `TASK-035` |
| Frontend начнёт трактовать угол иначе | Неправильная ориентация wind indicator и движения | Семантика `0 -> +X`, `PI / 2 -> +Z` зафиксирована во всех API docs |

## План реализации
1. Сверить текущую реализацию `Wind`/`GameRoom` и существующие примеры в docs.
2. Зафиксировать канонику в orchestration docs.
3. Синхронизировать backend/frontend API notes и отметить задачу выполненной.

## Проверки

### Автоматические проверки
| Репозиторий | Команда | Зачем | Статус |
|-------------|---------|-------|--------|
| `sea_patrol_backend` | `.\gradlew.bat test` | Backend unit/integration tests | `N/A` |
| `sea_patrol_backend` | `.\gradlew.bat clean build` | Полная сборка backend | `N/A` |
| `sea_patrol_frontend` | `npm run test:run` | Frontend tests | `N/A` |
| `sea_patrol_frontend` | `npm run build` | Production build frontend | `N/A` |

### Ручная / интеграционная проверка
- [x] Проверена документация и актуальность контрактов

**Заметки по проверке:**
- Задача была документационной; runtime-код и тесты не менялись.

## Реализация

### Измененные файлы
1. `sea_patrol_orchestration/API.md` - добавлена каноника `wind` payload и update semantics.
2. `sea_patrol_orchestration/PROJECTS_ORCESTRATION_INFO.md` - синхронизирована краткая интеграционная сводка.
3. `sea_patrol_orchestration/ROADMAP-TASKS.md` - `TASK-030` помечена как выполненная.
4. `sea_patrol_backend/ai-docs/API_INFO.md` - уточнена backend-семантика `wind` и примеры.
5. `sea_patrol_frontend/ai-docs/API_INFO.md` - уточнены frontend-ожидания по `wind`.

### Незапланированные находки
- `TASK-035` нужно держать как отдельную runtime-задачу, потому что текущая backend-реализация ещё не совпадает с желаемой продуктовой policy «ветер вращается по часовой стрелке».

## QA / Review

### QA
- [x] Все acceptance criteria подтверждены
- [x] Критические сценарии пройдены
- [x] Регрессии не обнаружены

**QA статус:** `Passed`

### Code Review
| Приоритет | Комментарий | Статус |
|-----------|-------------|--------|
| `Low` | Clockwise wind policy намеренно оставлена follow-up задачей `TASK-035`, чтобы не смешивать contract и runtime behavior | `Resolved` |

**Review решение:** `Approve`

## Финализация
- [x] Документация синхронизирована
- [x] Задача перенесена в выполненные / архив
- [x] Следующие задачи или follow-up зафиксированы

## Ссылки
- Related docs: `sea_patrol_orchestration/API.md`, `sea_patrol_backend/ai-docs/API_INFO.md`, `sea_patrol_frontend/ai-docs/API_INFO.md`
