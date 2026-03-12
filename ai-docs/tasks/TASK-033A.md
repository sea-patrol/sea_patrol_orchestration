# TASK-033A - Канонический sailLevel contract для MVP

## Метаданные
- **ID:** `TASK-033A`
- **Тип:** `docs`
- **Статус:** `Done`
- **Приоритет:** `High`
- **Дата создания:** `2026-03-12`
- **Автор:** `Codex`
- **Связанный roadmap item:** `Wave 4 — Ветер, паруса и движение корабля`
- **Затронутые репозитории:** `sea_patrol_orchestration`, `sea_patrol_backend`, `sea_patrol_frontend`
- **Связанные задачи / PR:** `TASK-033B`, `TASK-033C`

## Контекст
В `ROADMAP.md` уже была зафиксирована идея дискретного управления парусами, но до этой задачи она не была оформлена как явный кросс-репозиторный контракт. Оставалась двусмысленность, считаются ли `PLAYER_INPUT.up/down` throttle/brake или управлением уровнем парусов.

## Цель
Зафиксировать единую канонику `sailLevel` для backend и frontend до начала runtime-реализации.

## Source of Truth
- Документация:
  - `sea_patrol_orchestration/ROADMAP.md`
  - `sea_patrol_orchestration/API.md`
  - `sea_patrol_orchestration/PROJECTS_ORCESTRATION_INFO.md`
  - `sea_patrol_backend/ai-docs/API_INFO.md`
  - `sea_patrol_frontend/ai-docs/API_INFO.md`

## Acceptance Criteria
- [x] В orchestration docs нет двусмысленности по `sailLevel 0..3`.
- [x] Ясно описано, что `PLAYER_INPUT.up/down` управляют именно уровнем парусов по rising-edge.
- [x] Зафиксировано, что `sailLevel` синхронизируется как часть player state в `INIT_GAME_STATE` / `UPDATE_GAME_STATE`.

## Scope
**Включает:**
- фиксацию `sailLevel` как server-authoritative дискретного состояния;
- фиксацию диапазона `0..3` и default `3`;
- фиксацию rising-edge semantics для `PLAYER_INPUT.up/down`;
- фиксацию места синхронизации `sailLevel` в room messages.

**Не включает (out of scope):**
- backend runtime реализацию `sailLevel`;
- frontend HUD/runtime реализацию `sailLevel`;
- изменение physics-модели движения.

## Технический подход
Контракт зафиксирован так, чтобы не смешивать transport и реализацию:
- `sailLevel` — часть player state;
- authoritative source — backend;
- `up/down` меняют уровень парусов ступенчато, а не дают непрерывный throttle;
- реальная runtime-поддержка разнесена в `TASK-033B` и `TASK-033C`.

## Изменения по репозиториям

### `sea_patrol_orchestration`
- [x] Обновить `API.md`
- [x] Обновить `PROJECTS_ORCESTRATION_INFO.md`
- [x] Обновить `ROADMAP-TASKS.md`
- [x] Обновить другие документы: `ai-docs/tasks/TASK-033A.md`

### `sea_patrol_backend`
- [x] Обновить `ai-docs/API_INFO.md`

### `sea_patrol_frontend`
- [x] Обновить `ai-docs/API_INFO.md`

## Контракты и данные

### API / WebSocket
- `sailLevel` — `0 | 1 | 2 | 3`
- `0` = паруса полностью убраны
- `3` = все паруса подняты
- default для нового room player: `3`
- `PLAYER_INPUT.up/down` — rising-edge команды `+1/-1`
- `sailLevel` синхронизируется как часть player state в `INIT_GAME_STATE` / `UPDATE_GAME_STATE`

## Проверки

### Автоматические проверки
| Репозиторий | Команда | Зачем | Статус |
|-------------|---------|-------|--------|
| `sea_patrol_backend` | `.\gradlew.bat test` | Backend tests | `N/A` |
| `sea_patrol_frontend` | `npm run test:run` | Frontend tests | `N/A` |

### Ручная / интеграционная проверка
- [x] Проверена документация и актуальность контрактов

**Заметки по проверке:**
- Задача документационная; runtime-код и тесты не менялись.

## QA / Review

### QA
- [x] Все acceptance criteria подтверждены
- [x] Документация синхронизирована

**QA статус:** `Passed`

### Code Review
| Приоритет | Комментарий | Статус |
|-----------|-------------|--------|
| `Low` | Runtime implementation сознательно вынесена в `TASK-033B` и `TASK-033C` | `Resolved` |

**Review решение:** `Approve`

## Финализация
- [x] Документация синхронизирована
- [x] Задача перенесена в выполненные / архив
- [x] Следующие runtime-задачи зафиксированы
