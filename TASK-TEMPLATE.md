# TASK-<N> - Шаблон общей задачи

## Метаданные
- **ID:** `TASK-<N>`
- **Тип:** `feature` / `bugfix` / `chore` / `docs` / `spike`
- **Статус:** `Todo` / `In Progress` / `Review` / `Done`
- **Приоритет:** `High` / `Medium` / `Low`
- **Дата создания:** `YYYY-MM-DD`
- **Автор:** `<name>`
- **Связанный roadmap item:** `<milestone / section>`
- **Затронутые репозитории:** `sea_patrol_backend` / `sea_patrol_frontend` / `sea_patrol_orchestration`
- **Связанные задачи / PR:** `<links or IDs>`

## Контекст
<!-- Коротко: какая проблема решается, почему это важно, какой пользовательский или технический эффект ожидается -->

## Цель
<!-- Одно-два предложения с ожидаемым конечным результатом -->

## Source of Truth
<!-- Перечислить код и документы, на которые опирается задача -->
- Код / тесты:
  - `...`
- Документация:
  - `sea_patrol_orchestration/PROJECTS_ORCESTRATION_INFO.md`
  - `sea_patrol_orchestration/API.md`
  - `...`

## Acceptance Criteria
- [ ] Критерий 1
- [ ] Критерий 2
- [ ] Критерий 3

## Scope
**Включает:**
- ...

**Не включает (out of scope):**
- ...

## Предпосылки и зависимости
- ...

## Технический подход
<!-- Коротко описать решение, ключевые модули, инварианты и ограничения -->

## Изменения по репозиториям

### `sea_patrol_orchestration`
- [ ] Не требуется
- [ ] Обновить `API.md`
- [ ] Обновить `PROJECTS_ORCESTRATION_INFO.md`
- [ ] Обновить `ROADMAP.md` / `ROADMAP-TASKS.md`
- [ ] Обновить другие документы: `...`

### `sea_patrol_backend`
- [ ] Не требуется
- [ ] Изменения в runtime / API / persistence
- [ ] Обновить `ai-docs/PROJECT_INFO.md`
- [ ] Обновить `ai-docs/API_INFO.md`
- [ ] Добавить или обновить тесты

### `sea_patrol_frontend`
- [ ] Не требуется
- [ ] Изменения в UI / state / API client / WS
- [ ] Обновить `ai-docs/PROJECT_INFO.md`
- [ ] Обновить `ai-docs/API_INFO.md`
- [ ] Добавить или обновить тесты

## Контракты и данные
<!-- Заполнять, если задача меняет интеграцию, DTO, WS message types, persistence model или конфигурацию -->

### API / WebSocket
- ...

### Данные / конфигурация / миграции
- ...

## Риски и меры контроля
| Риск | Почему это риск | Мера контроля |
|------|-----------------|---------------|
| ... | ... | ... |

## План реализации
1. ...
2. ...
3. ...

## Проверки

### Автоматические проверки
| Репозиторий | Команда | Зачем | Статус |
|-------------|---------|-------|--------|
| `sea_patrol_backend` | `.\gradlew.bat test` | Backend unit/integration tests | `Not Run / Passed / Failed / N/A` |
| `sea_patrol_backend` | `.\gradlew.bat clean build` | Полная сборка backend | `Not Run / Passed / Failed / N/A` |
| `sea_patrol_frontend` | `npm run test:run` | Frontend tests | `Not Run / Passed / Failed / N/A` |
| `sea_patrol_frontend` | `npm run build` | Production build frontend | `Not Run / Passed / Failed / N/A` |

### Ручная / интеграционная проверка
- [ ] Проверен основной сценарий задачи
- [ ] Проверены смежные сценарии / регрессия
- [ ] Проверена интеграция frontend/backend
- [ ] Проверена документация и актуальность контрактов

**Заметки по проверке:**
- ...

## Реализация
<!-- Заполнять по факту выполнения -->

### Измененные файлы
1. `path/to/file` - что изменено и зачем
2. `path/to/file` - что изменено и зачем

### Незапланированные находки
- ...

## QA / Review

### QA
- [ ] Все acceptance criteria подтверждены
- [ ] Критические сценарии пройдены
- [ ] Регрессии не обнаружены

**QA статус:** `Passed / Failed / Blocked`

### Code Review
| Приоритет | Комментарий | Статус |
|-----------|-------------|--------|
| `High / Medium / Low` | ... | `Open / Resolved / Won't Fix` |

**Review решение:** `Approve / Changes Requested / Pending`

## Финализация
- [ ] Изменения смержены
- [ ] Документация синхронизирована
- [ ] Задача перенесена в выполненные / архив
- [ ] Следующие задачи или follow-up зафиксированы

## Ссылки
- PR: `<link or #N>`
- Commit: `<hash>`
- Related docs: `...`
