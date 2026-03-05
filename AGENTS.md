# AGENTS.md (sea_patrol_orchestration)

## Назначение
Этот репозиторий — «оркестрация» для двух проектов Sea Patrol:
- `sea_patrol_backend` — backend браузерной игры (HTTP + WebSocket).
- `sea_patrol_frontend` — frontend (React + Three.js/R3F).

Цель оркестрации: хранить единые сводные документы, договорённости и материалы для планирования развития системы в целом.

## Каноничные документы оркестрации
- Единый обзор (архитектура/запуск/интеграция): `PROJECTS_ORCESTRATION_INFO.md`
- Единый свод контрактов (REST + WebSocket): `API.md`

## Каноничные документы внутри репозиториев
При работе *внутри* соответствующих подпроектов сначала читать их документы:
- Backend: `sea_patrol_backend/AGENTS.md`, `sea_patrol_backend/ai-docs/PROJECT_INFO.md`, `sea_patrol_backend/ai-docs/API_INFO.md`
- Frontend: `sea_patrol_frontend/AGENTS.md`, `sea_patrol_frontend/ai-docs/PROJECT_INFO.md`, `sea_patrol_frontend/ai-docs/API_INFO.md`

Если сводные документы оркестрации расходятся с подпроектом, приоритет у кода подпроекта и его `ai-docs/*`, затем нужно синхронизировать сводные документы.

## Правила работы (в оркестрации)
1. Отвечать на русском (код/идентификаторы — как есть).
2. Не менять файлы внутри `sea_patrol_backend/` и `sea_patrol_frontend/`, если это явно не запрошено пользователем.
3. Если задача затрагивает контракты или интеграцию фронта/бэка, обновлять сводные документы `PROJECTS_ORCESTRATION_INFO.md` и/или `API.md` (и при необходимости — соответствующие `ai-docs/*` в подпроектах).
4. Не добавлять секреты/токены в репозиторий; конфигурацию фиксировать через переменные окружения и документацию.
