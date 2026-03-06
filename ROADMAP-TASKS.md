# ROADMAP-TASKS.md — Sea Patrol

Источник: `ROADMAP.md`

## Принципы
- Задачи отсортированы от самых базовых и блокирующих к более поздним.
- Каждая задача должна давать маленький, проверяемый инкремент.
- Предпочтение отдано задачам, которые уменьшают риск и упрощают последующие изменения.
- Если задача затрагивает контракт между frontend и backend, сначала фиксируется контракт, затем реализация.

## Формат задачи
- `ID` — уникальный номер задачи.
- `Priority`:
  - `P0` — блокирует дальнейшее развитие или первый playable loop.
  - `P1` — нужен для MVP, но не первый блокер.
  - `P2` — нужен после базового цикла.
- `Track` — `Backend`, `Frontend`, `Shared`.
- `Depends on` — список обязательных предыдущих задач.
- `Goal` — что именно добавляет задача.
- `Scope` — очень узкий инкремент.
- `Acceptance` — минимальные критерии готовности.

---

## Wave 0 — Контракты и каркас

### TASK-001 - done
- `Priority`: `P0`
- `Track`: `Shared`
- `Depends on`: `-`
- `Goal`: Зафиксировать канонический auth contract для MVP.
- `Scope`: Уточнить `login/signup/error` contract в orchestration docs и выровнять ожидания frontend/backend.
- `Acceptance`:
  - В `API.md` нет двусмысленности по `signup`, `login`, `errors[]`.
  - В roadmap и API нет противоречий по auth response.

### TASK-002 - done
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-001`
- `Goal`: Стабилизировать backend auth error contract.
- `Scope`: Привести backend responses и тесты auth к зафиксированному формату ошибок.
- `Acceptance`:
  - `signup/login` возвращают ожидаемый JSON.
  - Ошибки auth отдаются в одном формате.
  - Есть backend tests на success/failure cases.

### TASK-003 - done
- `Priority`: `P0`
- `Track`: `Frontend`
- `Depends on`: `TASK-001`
- `Goal`: Научить frontend корректно разбирать backend auth errors.
- `Scope`: Обновить auth API/client state так, чтобы UI показывал сообщения из backend format.
- `Acceptance`:
  - Login/signup UI показывает содержательные ошибки.
  - Нет падения из-за отсутствия `userId`.
  - Есть frontend tests на parsing ошибок.

### TASK-004
- `Priority`: `P0`
- `Track`: `Shared`
- `Depends on`: `TASK-001`
- `Goal`: Зафиксировать базовый room/lobby contract.
- `Scope`: Уточнить `GET /api/v1/rooms`, `POST /api/v1/rooms`, `POST /api/v1/rooms/{roomId}/join`, `ROOMS_UPDATED`, `ROOM_JOINED`, `ROOM_JOIN_REJECTED`, `SPAWN_ASSIGNED`.
- `Acceptance`:
  - В `API.md` есть минимальные request/response payloads.
  - Не осталось альтернативных flow для join/spawn.

### TASK-005
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-004`
- `Goal`: Вынести room/tick limits в конфигурацию.
- `Scope`: Перенести `tick`, `maxRooms`, `maxPlayersPerRoom`, reconnect grace и related defaults в config properties.
- `Acceptance`:
  - Конфигурация не захардкожена в core logic.
  - Есть дефолтные значения для dev/MVP.

### TASK-006
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-004`
- `Goal`: Ввести single-session policy.
- `Scope`: Один пользователь = одна игровая сессия; duplicate login запрещён, кроме reconnect window.
- `Acceptance`:
  - Повторная параллельная сессия отклоняется.
  - reconnect в течение 30 секунд возможен.
  - Есть тесты на duplicate login/reconnect.

### TASK-007
- `Priority`: `P0`
- `Track`: `Frontend`
- `Depends on`: `TASK-004`
- `Goal`: Подготовить `GameUiShell` и базовую `UI mode model`.
- `Scope`: Создать отдельный UI shell поверх сцены и ввести состояния `LOADING`, `LOBBY`, `SAILING`, `CHAT_FOCUS`, `WINDOW_FOCUS`, `MENU_OPEN`, `RECONNECTING`, `RESPAWN`.
- `Acceptance`:
  - UI-слой отделён от 3D canvas.
  - Есть единая модель режимов UI.
  - Hotkeys не размазаны по разным компонентам.

---

## Wave 1 — Лобби и жизненный цикл комнат

### TASK-008
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-005`
- `Goal`: Ввести `RoomRegistry` как явный реестр активных комнат.
- `Scope`: Убрать неявный `computeIfAbsent` flow и сделать управляемый lifecycle комнат.
- `Acceptance`:
  - Комнаты создаются/получаются через один сервис.
  - Есть room registry abstraction.
  - Есть тест на создание и удаление комнаты.

### TASK-009
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-008`
- `Goal`: Реализовать `GET /api/v1/rooms`.
- `Scope`: Отдавать room catalog с `roomId`, `current/maxPlayers`, `mapId`, `mapName`, `status`.
- `Acceptance`:
  - REST endpoint возвращает список комнат.
  - Пустой каталог и заполненный каталог корректно обрабатываются.
  - Есть integration tests.

### TASK-010
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-008`
- `Goal`: Реализовать `POST /api/v1/rooms`.
- `Scope`: Создавать комнату с room limits и опциональным `mapId`.
- `Acceptance`:
  - Нельзя превысить `maxRooms`.
  - Комната создаётся с корректным `mapId`.
  - Ответ соответствует contract.

### TASK-011
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-008`, `TASK-006`
- `Goal`: Реализовать `POST /api/v1/rooms/{roomId}/join`.
- `Scope`: Backend валидирует admission и переводит активную WS-сессию из lobby в room streams.
- `Acceptance`:
  - Join невозможен без активной lobby WS session.
  - `FULL`/`NOT_FOUND` cases отдаются корректно.
  - После success сервер переключает session binding на room.

### TASK-012
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-008`
- `Goal`: Реализовать room cleanup policy.
- `Scope`: Комната закрывается, когда нет активных игроков и нет игроков в reconnect grace.
- `Acceptance`:
  - Пустая комната удаляется из registry.
  - Комната остаётся живой, пока есть игрок в 30-секундном grace.
  - Есть tests на cleanup behavior.

### TASK-013
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-008`
- `Goal`: Реализовать lobby room updates через WS.
- `Scope`: Отправлять `ROOMS_UPDATED` при создании комнаты, join/leave и cleanup.
- `Acceptance`:
  - Lobby clients получают обновление без polling.
  - Payload соответствует contract.

### TASK-014
- `Priority`: `P0`
- `Track`: `Frontend`
- `Depends on`: `TASK-004`, `TASK-007`
- `Goal`: Сделать страницу лобби.
- `Scope`: Экран лобби с загрузкой `GET /api/v1/rooms` и базовым empty/loading/error state.
- `Acceptance`:
  - При открытии страницы выполняется rooms request.
  - Комнаты отображаются списком.
  - Empty state читаем и понятен.

### TASK-015
- `Priority`: `P0`
- `Track`: `Frontend`
- `Depends on`: `TASK-014`
- `Goal`: Подключить WS на странице лобби.
- `Scope`: На экране лобби поднимать WS для чата и `ROOMS_UPDATED`.
- `Acceptance`:
  - Лобби подключает WS отдельно от игровой комнаты.
  - Изменения списка комнат обновляют UI.
  - Есть reconnect/status UI.

### TASK-016
- `Priority`: `P0`
- `Track`: `Frontend`
- `Depends on`: `TASK-014`, `TASK-015`, `TASK-011`
- `Goal`: Реализовать room join flow на фронте.
- `Scope`: По нажатию `join` вызывать REST `POST /api/v1/rooms/{roomId}/join` и переводить UI в room loading state.
- `Acceptance`:
  - Join работает из lobby UI.
  - Ошибки join показываются пользователю.
  - После success UI переходит к room init flow.

### TASK-017
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-015`
- `Goal`: Реализовать create room flow на фронте.
- `Scope`: Форма создания комнаты с выбором дефолтной карты/карты по `mapId`.
- `Acceptance`:
  - Create room вызывает backend endpoint.
  - UI обновляет room catalog после success.

---

## Wave 2 — Spawn, reconnect, room init

### TASK-018
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-011`
- `Goal`: Реализовать spawn calculation на backend.
- `Scope`: Вычислять initial spawn около `(0,0)` с random offset и server-side validation.
- `Acceptance`:
  - Координаты вычисляет только backend.
  - Spawn лежит в допустимых bounds.
  - Есть тест на диапазон координат.

### TASK-019
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-018`
- `Goal`: Реализовать `SPAWN_ASSIGNED`.
- `Scope`: Передавать отдельное сообщение с `roomId`, `reason`, `x`, `z`, `angle` для initial spawn и respawn.
- `Acceptance`:
  - Сообщение уходит при initial spawn.
  - Сообщение уходит при respawn.
  - Payload соответствует API contract.

### TASK-020
- `Priority`: `P0`
- `Track`: `Frontend`
- `Depends on`: `TASK-019`, `TASK-016`
- `Goal`: Обработать `SPAWN_ASSIGNED` на фронте.
- `Scope`: Переставлять локальный корабль только по authoritative spawn coordinates.
- `Acceptance`:
  - Клиент не рандомит spawn самостоятельно.
  - Initial spawn и respawn применяются корректно.

### TASK-021
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-006`, `TASK-011`
- `Goal`: Реализовать reconnect grace state.
- `Scope`: После disconnect удерживать session 30 секунд и позволять resume той же комнаты.
- `Acceptance`:
  - Session не удаляется мгновенно.
  - Reconnect within grace работает.
  - После timeout session удаляется.

### TASK-022
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-021`, `TASK-020`
- `Goal`: Реализовать reconnect UI flow.
- `Scope`: Показывать `RECONNECTING` state, пробовать восстановить room session и возвращать пользователя в лобби после timeout.
- `Acceptance`:
  - Пользователь видит reconnect status.
  - После success происходит возврат в room.
  - После fail UI переходит в lobby/new session flow.

---

## Wave 3 — Карты и статические данные

### TASK-023
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-005`
- `Goal`: Подключить `Liquibase` как единственный механизм миграций.
- `Scope`: Добавить базовую структуру changelog для schema/reference/dev seeds.
- `Acceptance`:
  - Backend поднимает БД через Liquibase.
  - Changelog структура заведена и документирована.

### TASK-024
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-023`
- `Goal`: Создать базовую schema для MVP persistence.
- `Scope`: Завести `users`, `ships`, `user_state`, `ship_cargo`, `quests`, `map_templates`.
- `Acceptance`:
  - Таблицы создаются Liquibase changelog'ом.
  - Есть техническая проверка на startup.

### TASK-025
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-023`
- `Goal`: Ввести `MapTemplateRegistry`.
- `Scope`: Загружать world templates из `src/main/resources/worlds/*` и держать in-memory registry.
- `Acceptance`:
  - Registry умеет list/get/default map.
  - Битая карта не попадает в available maps.

### TASK-026
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-025`
- `Goal`: Добавить первую рабочую карту `caribbean-01`.
- `Scope`: Создать минимальный пакет файлов карты: `manifest`, `colliders`, `spawn-points`, `poi`, `minimap`.
- `Acceptance`:
  - Карта проходит validation.
  - Комнату можно создать с `mapId = caribbean-01`.

### TASK-027
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-025`
- `Goal`: Добавить техническую карту `test-sandbox-01`.
- `Scope`: Простая тестовая карта с несколькими объектами для spawn/POI/trade/loot testing.
- `Acceptance`:
  - Карта зарегистрирована.
  - Её можно использовать для dev/debug комнат.

### TASK-028
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-023`
- `Goal`: Засидировать static catalogs.
- `Scope`: Seed'ы для ship classes, item catalog, map templates, merchant catalog, quest definitions.
- `Acceptance`:
  - Пустая БД после старта пригодна для MVP flow.
  - Seed idempotent.

### TASK-029
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-014`, `TASK-026`
- `Goal`: Показывать информацию о карте в лобби.
- `Scope`: Отображать `mapName`/region/preview placeholder в списке комнат и create room UI.
- `Acceptance`:
  - Пользователь видит, на какой карте создана комната.
  - Выбор карты не ломает create room flow.

---

## Wave 4 — Базовый room simulation и HUD foundation

### TASK-030
- `Priority`: `P0`
- `Track`: `Frontend`
- `Depends on`: `TASK-007`, `TASK-020`
- `Goal`: Собрать базовый in-game HUD layout.
- `Scope`: Разместить `PlayerStatusCard`, `ChatPanel`, `GameMenuButton`, `CompassMiniMapCluster`, inventory/journal buttons.
- `Acceptance`:
  - HUD живёт поверх canvas.
  - Layout работает в `SAILING` mode.

### TASK-031
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-030`
- `Goal`: Реализовать `CHAT_FOCUS` и `WINDOW_FOCUS` behavior.
- `Scope`: Горячие клавиши и блокировка ship input при chat/window focus.
- `Acceptance`:
  - Chat input не конфликтует с ship controls.
  - Большие окна блокируют ship input.

### TASK-032
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-008`
- `Goal`: Ввести base entity transport model.
- `Scope`: Зафиксировать минимальные server events для entity spawn/update/despawn внутри комнаты.
- `Acceptance`:
  - Есть единый подход для world entities beyond players.
  - Contract задокументирован и применим к crates/NPC/projectiles.

### TASK-033
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-032`, `TASK-026`
- `Goal`: Добавить тестовые world objects на карте.
- `Scope`: Поднять на тестовой/основной карте несколько POI и интерактивных сущностей с `CIRCLE/RECTANGLE` geometry.
- `Acceptance`:
  - Объекты появляются в комнате.
  - Их geometry загружается из template data.

---

## Wave 5 — Боевка минимального цикла

### TASK-034
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-032`
- `Goal`: Ввести authoritative ship combat state.
- `Scope`: HP, max HP, reload state, minimal death/respawn transition.
- `Acceptance`:
  - Ship combat state существует на backend.
  - Потопление переводит игрока в respawn flow.

### TASK-035
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-034`
- `Goal`: Реализовать collision damage.
- `Scope`: Простая модель урона от столкновения кораблей.
- `Acceptance`:
  - Столкновение наносит урон.
  - Есть тест на damage application.

### TASK-036
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-034`
- `Goal`: Реализовать projectile subsystem skeleton.
- `Scope`: `FIRE_CANNON`, projectile lifecycle, hit/despawn events без сложного баланса.
- `Acceptance`:
  - Projectile spawn/hit/despawn работает.
  - Сервер остаётся authoritative.

### TASK-037
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-034`, `TASK-030`
- `Goal`: Показать combat state в HUD.
- `Scope`: HP, reload, боезапас, respawn/reconnect indicators.
- `Acceptance`:
  - PlayerStatusCard отражает combat state.
  - Потопление и respawn видны пользователю.

### TASK-038
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-036`
- `Goal`: Добавить projectile visuals.
- `Scope`: Косметическая визуализация залпа, попадания и despawn.
- `Acceptance`:
  - Выстрелы видны в клиенте.
  - Клиент следует authoritative backend events.

---

## Wave 6 — Лут, cargo, торговля

### TASK-039
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-028`, `TASK-032`
- `Goal`: Ввести runtime cargo model.
- `Scope`: Грузовые слоты/вместимость и операции add/remove item для ship cargo.
- `Acceptance`:
  - Cargo считается на backend.
  - Есть unit tests на inventory operations.

### TASK-040
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-032`, `TASK-033`, `TASK-039`
- `Goal`: Реализовать floating crates.
- `Scope`: Spawn, pickup, despawn, respawn по минимальному lifecycle.
- `Acceptance`:
  - Ящики появляются на карте.
  - Игрок может поднять их server-authoritatively.

### TASK-041
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-040`, `TASK-030`
- `Goal`: Добавить inventory window skeleton.
- `Scope`: Окно инвентаря/трюма и отображение предметов/количеств.
- `Acceptance`:
  - Inventory window открывается по hotkey/button.
  - Cargo state отображается в UI.

### TASK-042
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-040`
- `Goal`: Показать prompts взаимодействия с loot.
- `Scope`: Подсказки/индикаторы при возможности pickup.
- `Acceptance`:
  - Игрок понимает, когда может подобрать объект.

### TASK-043
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-028`, `TASK-033`, `TASK-039`
- `Goal`: Реализовать минимального торговца.
- `Scope`: `TRADE_OPEN`, buy/sell validation, fixed prices, merchant POI.
- `Acceptance`:
  - Торговля работает только у торговой точки.
  - Покупка/продажа валидируется сервером.

### TASK-044
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-043`, `TASK-041`
- `Goal`: Сделать trade UI.
- `Scope`: Окно покупки/продажи с balance/cargo feedback.
- `Acceptance`:
  - Игрок может купить и продать базовые ресурсы.
  - Ошибки торговли отображаются корректно.

---

## Wave 7 — NPC, рыбалка, квесты

### TASK-045
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-028`, `TASK-032`
- `Goal`: Добавить базовый pirate NPC archetype.
- `Scope`: Spawn, patrol/agro skeleton, simple attack behavior.
- `Acceptance`:
  - Пиратский NPC появляется в комнате.
  - Может вступать в базовый бой.

### TASK-046
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-028`, `TASK-032`
- `Goal`: Добавить рыбу/рыбалку как простой interaction loop.
- `Scope`: Fish entities, catch interaction, loot into cargo.
- `Acceptance`:
  - Игрок может получить `FISH` через world interaction.

### TASK-047
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-028`, `TASK-039`, `TASK-043`, `TASK-045`, `TASK-046`
- `Goal`: Реализовать минимальную quest system.
- `Scope`: Один активный квест, progress tracking, rewards, completion.
- `Acceptance`:
  - Работают `deliver-wood-01`, `catch-fish-01`, `sink-pirate-01`.
  - Награда начисляется сервером.

### TASK-048
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-047`, `TASK-030`
- `Goal`: Добавить journal window skeleton.
- `Scope`: Окно журнала с активным квестом и кратким прогрессом.
- `Acceptance`:
  - Journal открывается по hotkey/button.
  - Виден активный квест и его progress.

---

## Wave 8 — Persistence MVP

### TASK-049
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-024`, `TASK-039`
- `Goal`: Ввести repositories для user/profile/cargo persistence.
- `Scope`: Реализовать persistence interfaces поверх H2/R2DBC.
- `Acceptance`:
  - User/profile/cargo можно сохранить и восстановить.

### TASK-050
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-049`, `TASK-021`
- `Goal`: Реализовать save policy MVP.
- `Scope`: Периодическое сохранение + save on disconnect + save on meaningful events.
- `Acceptance`:
  - Данные игрока не теряются после disconnect/server restart в ожидаемом MVP сценарии.

### TASK-051
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-049`, `TASK-047`
- `Goal`: Сохранять quest progress и currency.
- `Scope`: Persistence для квестов, денег и active ship state.
- `Acceptance`:
  - Квест и баланс переживают restart.

### TASK-052
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-050`, `TASK-051`
- `Goal`: Восстанавливать profile/session state на фронте.
- `Scope`: После повторного входа UI показывает актуальные cargo, balance, active quest, ship profile.
- `Acceptance`:
  - Пользователь после relogin видит сохранённый прогресс.

---

## Wave 9 — Карта и polish MVP

### TASK-053
- `Priority`: `P2`
- `Track`: `Frontend`
- `Depends on`: `TASK-030`, `TASK-026`
- `Goal`: Сделать minimap placeholder.
- `Scope`: Простой compass/minimap cluster с положением игрока и базовыми POI markers.
- `Acceptance`:
  - Миникарта видна в HUD.
  - Позиция игрока и базовые markers отображаются.

### TASK-054
- `Priority`: `P2`
- `Track`: `Frontend`
- `Depends on`: `TASK-053`
- `Goal`: Сделать полноэкранное окно карты.
- `Scope`: `MapWindow` с room bounds, player position и POI.
- `Acceptance`:
  - `M` открывает карту.
  - Карта использует данные текущей комнаты/шаблона карты.

### TASK-055
- `Priority`: `P2`
- `Track`: `Backend`
- `Depends on`: `TASK-032`, `TASK-040`, `TASK-045`, `TASK-046`
- `Goal`: Добавить entity validation/test suite.
- `Scope`: Минимальные тесты на lifecycle world entities и map template validation.
- `Acceptance`:
  - Карты и сущности можно валидировать отдельным прогоном.

### TASK-056
- `Priority`: `P2`
- `Track`: `Frontend`
- `Depends on`: `TASK-030`, `TASK-031`, `TASK-048`, `TASK-054`
- `Goal`: Дополировать игровые UI modes.
- `Scope`: Финальные переходы между `SAILING`, `CHAT_FOCUS`, `WINDOW_FOCUS`, `MENU_OPEN`, `RECONNECTING`, `RESPAWN`.
- `Acceptance`:
  - UI modes не конфликтуют друг с другом.
  - Управление и окна ведут себя предсказуемо.

### TASK-057
- `Priority`: `P2`
- `Track`: `Backend`
- `Depends on`: `TASK-050`
- `Goal`: Добавить structured metrics/logging для MVP эксплуатации.
- `Scope`: Tick lag, room count, entity count, reconnects, room cleanup, trade/quest events.
- `Acceptance`:
  - Ключевые runtime события видны в логах/метриках.

---

## MVP Cut Line
Если нужен самый ранний playable MVP без лишнего полиша, минимальная линия отсечения такая:
- `TASK-001` … `TASK-022`
- `TASK-023` … `TASK-030`
- `TASK-034` … `TASK-044`
- `TASK-047`
- `TASK-049` … `TASK-052`

Это даст:
- auth;
- lobby;
- create/join room;
- spawn/reconnect basics;
- одну рабочую карту;
- базовую боёвку;
- loot/cargo/trade;
- один базовый quest loop;
- persistence player progress.
