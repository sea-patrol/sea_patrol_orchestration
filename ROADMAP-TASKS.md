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

### TASK-004 - done
- `Priority`: `P0`
- `Track`: `Shared`
- `Depends on`: `TASK-001`
- `Goal`: Зафиксировать базовый room/lobby contract.
- `Scope`: Уточнить `GET /api/v1/rooms`, `POST /api/v1/rooms`, `POST /api/v1/rooms/{roomId}/join`, `ROOMS_UPDATED`, `ROOM_JOINED`, `ROOM_JOIN_REJECTED`, `SPAWN_ASSIGNED`.
- `Acceptance`:
  - В `API.md` есть минимальные request/response payloads.
  - Не осталось альтернативных flow для join/spawn.

### TASK-005 - done
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-004`
- `Goal`: Вынести room/tick limits в конфигурацию.
- `Scope`: Перенести `tick`, `maxRooms`, `maxPlayersPerRoom`, reconnect grace и related defaults в config properties.
- `Acceptance`:
  - Конфигурация не захардкожена в core logic.
  - Есть дефолтные значения для dev/MVP.

### TASK-006 - done
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-004`
- `Goal`: Ввести single-session policy.
- `Scope`: Один пользователь = одна игровая сессия; duplicate login запрещён, кроме reconnect window.
- `Acceptance`:
  - Повторная параллельная сессия отклоняется.
  - reconnect в течение 15 секунд возможен.
  - Есть тесты на duplicate login/reconnect.

### TASK-007 - done
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

### TASK-008 - done
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-005`
- `Goal`: Ввести `RoomRegistry` как явный реестр активных комнат.
- `Scope`: Убрать неявный `computeIfAbsent` flow и сделать управляемый lifecycle комнат.
- `Acceptance`:
  - Комнаты создаются/получаются через один сервис.
  - Есть room registry abstraction.
  - Есть тест на создание и удаление комнаты.

### TASK-009 - done
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-008`
- `Goal`: Реализовать `GET /api/v1/rooms`.
- `Scope`: Отдавать room catalog с `roomId`, `current/maxPlayers`, `mapId`, `mapName`, `status`.
- `Acceptance`:
  - REST endpoint возвращает список комнат.
  - Пустой каталог и заполненный каталог корректно обрабатываются.
  - Есть integration tests.

### TASK-010 - done
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-008`
- `Goal`: Реализовать `POST /api/v1/rooms`.
- `Scope`: Создавать комнату с room limits и опциональным `mapId`.
- `Acceptance`:
  - Нельзя превысить `maxRooms`.
  - Комната создаётся с корректным `mapId`.
  - Ответ соответствует contract.

### TASK-011 - done
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-008`, `TASK-006`
- `Goal`: Реализовать `POST /api/v1/rooms/{roomId}/join`.
- `Scope`: Backend валидирует admission и переводит активную WS-сессию из lobby в room streams.
- `Acceptance`:
  - Join невозможен без активной lobby WS session.
  - `FULL`/`NOT_FOUND` cases отдаются корректно.
  - После success сервер переключает session binding на room.

### TASK-012 - done
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-008`
- `Goal`: Реализовать room cleanup policy.
- `Scope`: Комната закрывается, когда нет активных игроков и нет игроков в reconnect grace.
- `Acceptance`:
  - Пустая комната удаляется из registry.
  - Комната остаётся живой, пока есть игрок в 15-секундном grace.
  - Есть tests на cleanup behavior.

### TASK-013 - done
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-008`
- `Goal`: Реализовать lobby room updates через WS.
- `Scope`: Отправлять `ROOMS_UPDATED` при создании комнаты, join/leave и cleanup.
- `Acceptance`:
  - Lobby clients получают обновление без polling.
  - Payload соответствует contract.

### TASK-014 - done
- `Priority`: `P0`
- `Track`: `Frontend`
- `Depends on`: `TASK-004`, `TASK-007`
- `Goal`: Сделать страницу лобби.
- `Scope`: Экран лобби с загрузкой `GET /api/v1/rooms` и базовым empty/loading/error state.
- `Acceptance`:
  - При открытии страницы выполняется rooms request.
  - Комнаты отображаются списком.
  - Empty state читаем и понятен.

### TASK-015 - done
- `Priority`: `P0`
- `Track`: `Frontend`
- `Depends on`: `TASK-014`
- `Goal`: Подключить WS на странице лобби.
- `Scope`: На экране лобби поднимать WS для чата и `ROOMS_UPDATED`.
- `Acceptance`:
  - Лобби подключает WS отдельно от игровой комнаты.
  - Изменения списка комнат обновляют UI.
  - Есть reconnect/status UI.

### TASK-016 - done
- `Priority`: `P0`
- `Track`: `Frontend`
- `Depends on`: `TASK-014`, `TASK-015`, `TASK-011`
- `Goal`: Реализовать room join flow на фронте.
- `Scope`: По нажатию `join` вызывать REST `POST /api/v1/rooms/{roomId}/join` и переводить UI в room loading state.
- `Acceptance`:
  - Join работает из lobby UI.
  - Ошибки join показываются пользователю.
  - После success UI переходит к room init flow.

### TASK-017 - done
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-015`
- `Goal`: Реализовать create room flow на фронте.
- `Scope`: Форма создания комнаты с выбором дефолтной карты/карты по `mapId`.
- `Acceptance`:
  - Create room вызывает backend endpoint.
  - UI обновляет room catalog после success.

### TASK-017A - done
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-011`, `TASK-013`
- `Goal`: Довести chat isolation до явного runtime behavior на backend.
- `Scope`: Зафиксировать и при необходимости исправить маршрутизацию сообщений между `group:lobby` и `group:room:<roomId>`, исключить утечки сообщений между комнатами и добавить интеграционные тесты на scoped chat.
- `Acceptance`:
  - Игрок в `lobby` получает только lobby chat.
  - Игрок в комнате получает только chat своей комнаты.
  - Сообщения из одной комнаты не попадают в другую комнату.
  - Есть backend tests на lobby chat и room chat isolation.

### TASK-017B - done
- `Priority`: `P0`
- `Track`: `Frontend`
- `Depends on`: `TASK-015`, `TASK-016`, `TASK-017A`
- `Goal`: Переключить chat UI на явные `Lobby` и `Room` scope.
- `Scope`: Убрать ощущение одного глобального чата; `ChatPanel` должен показывать текущую область общения, читать lobby chat только в лобби, а room chat только после входа в комнату.
- `Acceptance`:
  - В лобби пользователь видит и отправляет сообщения только в lobby chat.
  - После `join` пользователь видит и отправляет сообщения только в chat своей комнаты.
  - UI явно показывает текущий chat scope: `Lobby` или `Room <roomId/name>`.
  - История lobby chat и room chat не смешивается в одном списке сообщений.

### TASK-017C - done
- `Priority`: `P0`
- `Track`: `Frontend`
- `Depends on`: `TASK-014`, `TASK-015`, `TASK-017`
- `Goal`: Вынести лобби в отдельную лёгкую HTML-first страницу без 3D-сцены.
- `Scope`: Собрать отдельный lobby shell/page без `Canvas`, R3F и загрузки gameplay scene; оставить только список комнат, create/join actions, статус соединения и lobby chat.
- `Acceptance`:
  - При открытии лобби не монтируется 3D-сцена.
  - Lobby page остаётся простой и лаконичной HTML-страницей.
  - На экране лобби есть room list, create/join actions, connection status и lobby chat.
  - Переход в gameplay scene происходит только после успешного room join flow.

### TASK-017D - done
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-016`, `TASK-017C`
- `Goal`: Развести navigation flow `Home -> Lobby -> Room` и отделить lobby shell от gameplay shell.
- `Scope`: Явно разделить домашнюю страницу, лобби и экран комнаты/игры, чтобы пользователь не оказывался в лобби поверх уже поднятой игровой сцены.
- `Acceptance`:
  - Переход на lobby не поднимает gameplay scene.
  - Переход в room/game происходит только после завершения join/init flow.
  - Пользователю понятны состояния `Home`, `Lobby`, `Room` и переходы между ними.

---

## Wave 2 — Spawn, reconnect, room init

### TASK-018 - done
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-011`
- `Goal`: Реализовать spawn calculation на backend.
- `Scope`: Вычислять initial spawn около `(0,0)` с random offset и server-side validation.
- `Acceptance`:
  - Координаты вычисляет только backend.
  - Spawn лежит в допустимых bounds.
  - Есть тест на диапазон координат.

### TASK-019 - done
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-018`
- `Goal`: Реализовать `SPAWN_ASSIGNED`.
- `Scope`: Передавать отдельное сообщение с `roomId`, `reason`, `x`, `z`, `angle` для initial spawn и respawn.
- `Acceptance`:
  - Сообщение уходит при initial spawn.
  - Сообщение уходит при respawn.
  - Payload соответствует API contract.

### TASK-020 - done
- `Priority`: `P0`
- `Track`: `Frontend`
- `Depends on`: `TASK-019`, `TASK-016`
- `Goal`: Обработать `SPAWN_ASSIGNED` на фронте.
- `Scope`: Переставлять локальный корабль только по authoritative spawn coordinates.
- `Acceptance`:
  - Клиент не рандомит spawn самостоятельно.
  - Initial spawn и respawn применяются корректно.

### TASK-021 - done
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-006`, `TASK-011`
- `Goal`: Реализовать reconnect grace state.
- `Scope`: После disconnect удерживать session 15 секунд и позволять resume той же комнаты.
- `Acceptance`:
  - Session не удаляется мгновенно.
  - Reconnect within grace работает.
  - После timeout session удаляется.

### TASK-022 - done
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

### TASK-022A - done
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-022`, `TASK-017D`
- `Goal`: Сделать room-resume-first entry flow с домашней страницы.
- `Scope`: Переименовать главный CTA на `/` обратно в `Play` и при наличии аутентифицированного пользователя вести не в обычное лобби по умолчанию, а в flow попытки восстановления активной room session; если room resume не подтверждён, frontend уже затем откатывается в `/lobby`.
- `Acceptance`:
  - Кнопка на домашней странице снова называется `Play`.
  - После закрытия вкладки и повторного открытия приложения пользователь с ещё живой room session может через `Play` вернуться сразу в room flow, а не только в lobby.
  - Если активной room session уже нет, `Play` не ломает сценарий и приводит пользователя в обычный lobby/new session flow.

### TASK-022B - done
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-012`, `TASK-021`
- `Goal`: Ввести empty-room idle timeout и довести cleanup пустых комнат до предсказуемого поведения.
- `Scope`: Комната с `0` активных игроков и без игроков в reconnect grace должна не висеть бесконечно: нужно зафиксировать и реализовать отдельный idle timeout для пустой комнаты, включая комнаты, которые были созданы, но в них никто не зашёл, и комнаты, из которых все окончательно вышли.
- `Acceptance`:
  - Комната, созданная и оставшаяся пустой, автоматически закрывается после idle timeout.
  - Комната, из которой ушли все игроки и закончился reconnect grace, автоматически закрывается после idle timeout.
  - Закрытие комнаты обновляет room catalog и не оставляет «мертвые» комнаты в lobby list.

### TASK-022C - done
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-017`, `TASK-016`, `TASK-022B`
- `Goal`: Перевести create room UX в create-and-join flow.
- `Scope`: После успешного создания комнаты frontend не оставляет пользователя в лобби, а сразу запускает join/init flow для только что созданной комнаты и переводит игрока в неё так же, как при обычном `Join room`.
- `Acceptance`:
  - После `Create room` пользователь автоматически входит в только что созданную комнату.
  - UI показывает ошибки как create-, так и последующего join-шага, если один из них не прошёл.
  - Create-and-join flow не оставляет пустую комнату висеть бесконечно, если auto-join не завершился.

## Wave 3 — Карты, шаблоны мира и in-memory каталоги

### TASK-023 - done
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-010`
- `Goal`: Ввести `MapTemplateRegistry` как in-memory реестр карт.
- `Scope`: Загружать world templates из `src/main/resources/worlds/*`, валидировать их и держать в памяти без БД.
- `Acceptance`:
  - Registry умеет `list/get/default map`.
  - Битая карта не попадает в available maps.
  - Для работы registry не нужен `H2`.

### TASK-024 - done
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-023`
- `Goal`: Добавить первую рабочую карту `caribbean-01`.
- `Scope`: Подготовить минимальный пакет файлов карты: `manifest`, `bounds/colliders`, `spawn-points`, `poi`, `minimap` metadata, default wind settings.
- `Acceptance`:
  - Карта проходит validation.
  - Комнату можно создать с `mapId = caribbean-01`.
  - У карты есть enough metadata для лобби и room bootstrap.

### TASK-025 - done
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-023`
- `Goal`: Добавить техническую карту `test-sandbox-01`.
- `Scope`: Простая dev/debug карта для проверки spawn, POI, ветра, интерактивных объектов и боевых сценариев.
- `Acceptance`:
  - Карта зарегистрирована в `MapTemplateRegistry`.
  - Её можно использовать для dev/debug комнат.

### TASK-026 - done
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-023`, `TASK-008`
- `Goal`: Привязать room bootstrap к map template.
- `Scope`: При создании комнаты брать из карты world bounds, spawn rules, room metadata и initial wind defaults.
- `Acceptance`:
  - Room runtime знает, из какого map template он создан.
  - `INIT_GAME_STATE`/room metadata опираются на данные карты.
  - Карта становится реальным source of truth для room bootstrap.

### TASK-027 - done
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-023`
- `Goal`: Ввести in-memory static catalogs без БД.
- `Scope`: Загружать из backend resources ship classes, item catalog, merchant catalog, quest definitions и другие static definitions, не подключая `H2`.
- `Acceptance`:
  - Static catalogs доступны через runtime registry/services.
  - Пустой стенд поднимается и работает только на in-memory данных + resource files.
  - Нет зависимости на `Liquibase`/`H2` в основных игровых flow.

### TASK-028 - done
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-024`, `TASK-014`
- `Goal`: Показывать map metadata в лобби.
- `Scope`: Отображать `mapName`, `region` и preview placeholder в room list и create room UI.
- `Acceptance`:
  - Пользователь видит, на какой карте создана комната.
  - Lobby UI не скрывает map context за общим названием комнаты.

### TASK-029 - done
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-026`, `TASK-016`, `TASK-028`
- `Goal`: Добавить room loading summary перед входом в море.
- `Scope`: На `ROOM_LOADING`/join-init этапе показывать карту, room name, текущий статус входа и базовую room metadata до монтирования gameplay scene.
- `Acceptance`:
  - При входе в комнату пользователь видит понятный loading summary.
  - Room loading screen использует map/room metadata из backend contract.

---

## Wave 4 — Ветер, паруса и движение корабля

### TASK-030 - done
- `Priority`: `P0`
- `Track`: `Shared`
- `Depends on`: `TASK-026`
- `Goal`: Зафиксировать канонический wind contract для MVP.
- `Scope`: Уточнить в orchestration docs payload и семантику `wind` (`angle`, `speed`), место доставки в `INIT_GAME_STATE` / `UPDATE_GAME_STATE` и простую policy изменения ветра по часовой стрелке.
- `Acceptance`:
  - В `API.md` нет двусмысленности по wind payload.
  - Frontend/backend одинаково понимают угол, скорость и update semantics.

### TASK-031 - done
- `Priority`: `P0`
- `Track`: `Backend`
- `Depends on`: `TASK-030`, `TASK-026`
- `Goal`: Ввести authoritative wind state в комнате.
- `Scope`: Каждая комната хранит текущий `wind.angle` и `wind.speed`, а backend включает эти данные в `INIT_GAME_STATE` и `UPDATE_GAME_STATE`.
- `Acceptance`:
  - Клиент получает wind state из backend.
  - Один и тот же wind state согласован для всех игроков комнаты.

### TASK-032 - done
- `Priority`: `P0`
- `Track`: `Frontend`
- `Depends on`: `TASK-031`
- `Goal`: Научить frontend применять backend wind state.
- `Scope`: Поднять `wind` в game state клиента и убрать локальные/заглушечные источники ветра из runtime path.
- `Acceptance`:
  - Frontend использует только backend wind state.
  - Изменение ветра реально доходит до UI/runtime state.

### TASK-033 - done
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-031`
- `Goal`: Перевести логику движения парусника на зависимость от ветра.
- `Scope`: Простая server-authoritative модель: тяга и ускорение судна зависят от силы ветра, направления ветра и курса корабля; пока без сложной симуляции и без продвинутой аэродинамики.
- `Acceptance`:
  - Судно движется по-разному при попутном, боковом и встречном ветре.
  - Модель остаётся предсказуемой и тестируемой.

### TASK-033A - done
- `Priority`: `P1`
- `Track`: `Shared`
- `Depends on`: `TASK-030`, `TASK-033`
- `Goal`: Зафиксировать канонический `sailLevel` contract для MVP.
- `Scope`: Уточнить в orchestration docs, что `sailLevel` является server-authoritative дискретным состоянием корабля (`0..3`), стартует с `3`, меняется по rising-edge от `PLAYER_INPUT.up/down` (`+1/-1`) и синхронизируется клиенту в `INIT_GAME_STATE` / `UPDATE_GAME_STATE` как часть player state.
- `Acceptance`:
  - В `API.md` и связанных docs нет двусмысленности по уровням парусов `0..3`.
  - Ясно описано, что `0` = паруса полностью убраны, `3` = все паруса подняты.
  - Ясно описано, что `PLAYER_INPUT.up/down` не являются «газом/тормозом», а управляют уровнем парусов по rising-edge.

### TASK-033B - done
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-033A`, `TASK-033`
- `Goal`: Реализовать server-authoritative `sailLevel` на backend.
- `Scope`: Добавить в runtime корабля дискретный `sailLevel` (`0..3`), стартовое значение `3`, обработку `PLAYER_INPUT.up/down` как `+1/-1` уровня по rising-edge, использование `sailLevel` в формуле тяги и синхронизацию этого состояния в room messages.
- `Acceptance`:
  - При `sailLevel = 0` корабль не набирает ход от парусов.
  - При `sailLevel = 1..3` тяга от ветра и парусов возрастает ступенчато.
  - Уровень парусов не выходит за границы `0..3`.
  - Есть backend tests на edge handling и влияние `sailLevel` на движение.

### TASK-033C - done
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-033A`, `TASK-032`, `TASK-033B`
- `Goal`: Научить frontend отображать и использовать backend `sailLevel`.
- `Scope`: Поднять `sailLevel` в клиентский game state как часть player state, показывать игроку текущий уровень парусов (`0..3`) и объяснять связь между клавишами `вперед/назад` и поднятием/опусканием парусов без введения локального authoritative sail state.
- `Acceptance`:
  - Frontend получает `sailLevel` из backend `INIT_GAME_STATE` / `UPDATE_GAME_STATE`.
  - Игрок видит текущий уровень парусов в HUD.
  - UI не создаёт отдельную локальную модель парусов, противоречащую backend state.

### TASK-034 - done
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-032`, `TASK-033`
- `Goal`: Показать игроку состояние ветра и отклик движения.
- `Scope`: Добавить в HUD минимальный wind indicator и понятный feedback о том, как ветер влияет на ход корабля.
- `Acceptance`:
  - Игрок видит направление и силу ветра.
  - Изменение поведения судна не выглядит «магическим» без UI-подсказки.

### TASK-035 - done
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-031`, `TASK-033`
- `Goal`: Реализовать простое изменение ветра по часовой стрелке.
- `Scope`: Room runtime постепенно вращает `wind.angle` по часовой стрелке с фиксированным шагом/скоростью и рассылает обновления клиентам.
- `Acceptance`:
  - Ветер меняется предсказуемо и одинаково для всех игроков комнаты.
  - Клиент видит, что направление ветра постепенно поворачивается по часовой стрелке.

---

## Wave 4.5 — Меню комнаты, выход в лобби и debug toggle

### TASK-035A - done
- `Priority`: `P1`
- `Track`: `Shared`
- `Depends on`: `TASK-011`, `TASK-017D`
- `Goal`: Зафиксировать канонический room menu / leave-room contract.
- `Scope`: Уточнить в orchestration docs, как игрок выходит из комнаты обратно в лобби через меню: какой endpoint/flow используется, как backend переводит active WS session из room обратно в `lobby`, какие room/chat/catalog события считаются authoritative и что должен сделать frontend с `RoomSession`.
- `Acceptance`:
  - В `API.md` нет двусмысленности по flow `Room -> Menu -> Exit -> Lobby`.
  - Ясно зафиксировано, остаётся ли тот же WS connect и как меняется session binding.
  - Понятно, какие события/ответы должен ждать frontend после успешного leave.

### TASK-035B - done
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-035A`, `TASK-021`
- `Goal`: Реализовать backend leave-room flow без разрыва всей игровой сессии.
- `Scope`: Добавить server-authoritative room leave endpoint/flow, который убирает игрока из runtime комнаты, переводит его chat/session binding обратно в `lobby`, обновляет room catalog и не требует полного relogin/rehydration.
- `Acceptance`:
  - Игрок может штатно выйти из комнаты, не закрывая всю auth/WS session.
  - После leave backend переводит пользователя обратно в `group:lobby` и lobby room stream.
  - Room cleanup/counter logic корректно отрабатывает после leave.
  - Есть backend tests на `room -> lobby` transition.

### TASK-035C - done
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-035A`, `TASK-035B`, `TASK-036`
- `Goal`: Довести menu modal до реального room-exit UX.
- `Scope`: В `MENU_OPEN` показать модальное окно с кнопкой `Выйти`, которая запускает authoritative leave-room flow и возвращает игрока в `/lobby`, очищая stale room UI state.
- `Acceptance`:
  - В меню комнаты есть явная кнопка `Выйти`.
  - После успешного leave пользователь оказывается в рабочем lobby flow, а не в сломанном промежуточном состоянии.
  - Ошибки leave показываются в UI, не оставляя пользователя на пустом экране.

### TASK-035D - done
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-036`
- `Goal`: Добавить dev-only toggle для debug UI из меню комнаты.
- `Scope`: В `MENU_OPEN` показывать кнопку `Дебаг` только в debug/dev режиме и дать ей возможность включать/выключать overlay/debug layer runtime-переключателем без отдельной production-сборки.
- `Acceptance`:
  - Кнопка `Дебаг` не видна в production UI.
  - В debug/dev режиме игрок может включить и выключить debug layer прямо из меню.
  - Переключение не требует пересборки frontend и не ломает обычный gameplay HUD.

---

## Wave 5 — Игровой экран, HUD и базовые интеракции в комнате

### TASK-036 - done
- `Priority`: `P0`
- `Track`: `Frontend`
- `Depends on`: `TASK-007`, `TASK-020`, `TASK-032`
- `Goal`: Собрать понятный базовый in-game HUD.
- `Scope`: Разместить player status, room chat, wind indicator, menu button и quick-access кнопки поверх canvas.
- `Acceptance`:
  - HUD живёт поверх gameplay scene.
  - В `SAILING` mode пользователь видит основной игровой интерфейс.

### TASK-037 - done
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-036`
- `Goal`: Довести поведение `CHAT_FOCUS` и `WINDOW_FOCUS`.
- `Scope`: Горячие клавиши, блокировка ship input и переключение фокуса между chat/window/gameplay без конфликтов.
- `Acceptance`:
  - Chat input не конфликтует с ship controls.
  - Большие окна блокируют обычное sailing input.

### TASK-038 - done
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-026`
- `Goal`: Ввести базовую transport model для world entities.
- `Scope`: Зафиксировать и реализовать server events `spawn/update/despawn` для объектов внутри комнаты beyond players.
- `Acceptance`:
  - Есть единый transport path для POI, crates, NPC и future projectiles.
  - У сущностей есть стабильный `entityId` в рамках жизни комнаты.

### TASK-039
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-038`, `TASK-024`
- `Goal`: Добавить первые world objects и POI на карту.
- `Scope`: Поднять на карте несколько статичных/интерактивных объектов с геометрией и runtime state, достаточных для будущих trade/loot/fishing сценариев.
- `Acceptance`:
  - Объекты реально появляются в комнате.
  - Их metadata и geometry приходят из map template/resources.

### TASK-040
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-038`, `TASK-039`
- `Goal`: Показать игроку базовые room interactions.
- `Scope`: Ввести prompts/markers для nearby POI и интерактивных объектов без полноценных окон конкретных механик.
- `Acceptance`:
  - Игрок понимает, что рядом есть интерактивный объект.
  - UI не требует догадок о том, где доступно действие.

### TASK-041
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-036`
- `Goal`: Подготовить крупные игровые окна.
- `Scope`: Сделать shell-окна `Inventory`, `Journal`, `Map` с открытием/закрытием, layout и hotkeys, пока без полного наполнения данными.
- `Acceptance`:
  - Окна открываются и закрываются предсказуемо.
  - `WINDOW_FOCUS` работает с ними как с полноценными UI-режимами.

---

## Wave 6 — Боевка и respawn loop

### TASK-042
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-033`, `TASK-038`
- `Goal`: Ввести authoritative ship combat state.
- `Scope`: Добавить `HP`, `maxHP`, reload/cooldown state и минимальный death/sunk transition на backend.
- `Acceptance`:
  - Combat state существует на backend как часть authoritative ship runtime.
  - Потопление переводит игрока в respawn flow.

### TASK-043
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-042`
- `Goal`: Реализовать collision damage.
- `Scope`: Простая модель урона от столкновения кораблей/объектов без сложной физики повреждений.
- `Acceptance`:
  - Столкновение наносит урон.
  - Есть тест на damage application.

### TASK-044
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-042`, `TASK-038`
- `Goal`: Реализовать skeleton projectile subsystem.
- `Scope`: `FIRE_CANNON`, projectile lifecycle, `spawn/hit/despawn` events и минимальная server-authoritative логика попаданий.
- `Acceptance`:
  - Projectile spawn/hit/despawn работает.
  - Сервер остаётся authoritative по факту попадания.

### TASK-045
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-036`, `TASK-042`
- `Goal`: Показать combat state в HUD.
- `Scope`: Отобразить HP, reload, ammo/боезапас и базовые combat alerts в in-game UI.
- `Acceptance`:
  - Игрок видит своё боевое состояние.
  - HUD обновляется по authoritative backend data.

### TASK-046
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-044`
- `Goal`: Добавить projectile visuals и hit feedback.
- `Scope`: Косметическая визуализация залпа, попадания и despawn, не нарушающая server-authoritative contract.
- `Acceptance`:
  - Выстрелы и попадания видны игроку.
  - Клиент следует backend events, а не симулирует исход боя сам.

### TASK-047
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-045`, `TASK-019`
- `Goal`: Довести respawn UX.
- `Scope`: Показать явный `RESPAWN` screen/mode и корректный возврат в `SAILING` после authoritative respawn/init flow.
- `Acceptance`:
  - Потопление и ожидание respawn понятны пользователю.
  - Возврат в игру идёт только после backend respawn/init signals.

---

## Wave 7 — Трюм, лут и торговля

### TASK-048
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-027`, `TASK-038`
- `Goal`: Ввести runtime cargo model.
- `Scope`: Грузовые слоты/вместимость, типы предметов и операции add/remove для ship cargo в памяти процесса.
- `Acceptance`:
  - Cargo считается на backend.
  - Есть unit tests на inventory operations.

### TASK-049
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-039`, `TASK-048`
- `Goal`: Реализовать floating crates и базовый resource pickup.
- `Scope`: Spawn, pickup, despawn, respawn и server-authoritative перенос ресурсов в cargo.
- `Acceptance`:
  - Ящики/ресурсы появляются на карте.
  - Игрок может поднять их server-authoritatively.

### TASK-050
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-041`, `TASK-048`
- `Goal`: Наполнить inventory window реальными данными трюма.
- `Scope`: Отображать cargo state, вместимость и базовые типы предметов в UI.
- `Acceptance`:
  - Inventory window показывает реальный cargo state.
  - Пользователь видит состав трюма без dev/debug инструментов.

### TASK-051
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-049`, `TASK-040`
- `Goal`: Довести UX подбора лута.
- `Scope`: Подсказки, success/error feedback и обновление инвентаря после pickup.
- `Acceptance`:
  - Игрок понимает, когда может подобрать объект.
  - После pickup UI сразу отражает результат.

### TASK-052
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-027`, `TASK-039`, `TASK-048`
- `Goal`: Реализовать минимального торговца и backend trade rules.
- `Scope`: `TRADE_OPEN`, buy/sell validation, fixed prices и merchant POI без динамической экономики.
- `Acceptance`:
  - Торговля работает только у торговой точки.
  - Покупка/продажа валидируется сервером.

### TASK-053
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-050`, `TASK-052`
- `Goal`: Сделать trade UI.
- `Scope`: Окно покупки/продажи с balance/cargo feedback и понятными ошибками.
- `Acceptance`:
  - Игрок может купить и продать базовые ресурсы.
  - Ошибки торговли отображаются корректно.

---

## Wave 8 — NPC, рыбалка и квестовый loop

### TASK-054
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-038`, `TASK-042`
- `Goal`: Добавить базовый pirate NPC archetype.
- `Scope`: Spawn, patrol/agro skeleton и простое боевое поведение пирата.
- `Acceptance`:
  - Пиратский NPC появляется в комнате.
  - Он может вступать в базовый бой с игроком.

### TASK-055
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-038`, `TASK-048`
- `Goal`: Добавить рыбу/рыбалку как простой interaction loop.
- `Scope`: Fish entities или fishing spots, catch interaction и награда ресурсом в cargo.
- `Acceptance`:
  - Игрок может получить `FISH` через world interaction.
  - Flow не требует persistence или БД.

### TASK-056
- `Priority`: `P1`
- `Track`: `Backend`
- `Depends on`: `TASK-027`, `TASK-048`, `TASK-052`, `TASK-054`, `TASK-055`
- `Goal`: Реализовать минимальную quest system.
- `Scope`: Один активный квест, progress tracking, rewards и completion для нескольких простых quest archetypes.
- `Acceptance`:
  - Работают минимум `deliver-wood-01`, `catch-fish-01`, `sink-pirate-01`.
  - Награда начисляется сервером.

### TASK-057
- `Priority`: `P1`
- `Track`: `Frontend`
- `Depends on`: `TASK-041`, `TASK-056`
- `Goal`: Наполнить journal window квестовыми данными.
- `Scope`: Показывать активный квест, краткий progress и completion/reward feedback.
- `Acceptance`:
  - Journal показывает текущий квест и его прогресс.
  - Игроку понятен текущий PvE loop.

---

## Wave 9 — Навигационная карта, polish и наблюдаемость

### TASK-058
- `Priority`: `P2`
- `Track`: `Frontend`
- `Depends on`: `TASK-036`, `TASK-026`
- `Goal`: Сделать minimap/compass cluster.
- `Scope`: Показать положение игрока, направление движения и базовые POI markers на компактной карте HUD.
- `Acceptance`:
  - Миникарта видна в HUD.
  - Позиция игрока и базовые markers отображаются.

### TASK-059
- `Priority`: `P2`
- `Track`: `Frontend`
- `Depends on`: `TASK-058`, `TASK-041`
- `Goal`: Сделать полноэкранное окно карты.
- `Scope`: `MapWindow` с room bounds, player position, POI и текущим направлением ветра/курса.
- `Acceptance`:
  - `M` открывает карту.
  - Карта использует данные текущей комнаты/шаблона карты.

### TASK-060
- `Priority`: `P2`
- `Track`: `Backend`
- `Depends on`: `TASK-038`, `TASK-049`, `TASK-054`, `TASK-055`
- `Goal`: Добавить validation/test suite для карт и сущностей.
- `Scope`: Минимальные проверки lifecycle world entities, map template validation и sanity-check тесты интерактивных объектов.
- `Acceptance`:
  - Карты и сущности можно валидировать отдельным прогоном.
  - Broken data ловится до ручного тестирования в браузере.

### TASK-061
- `Priority`: `P2`
- `Track`: `Frontend`
- `Depends on`: `TASK-036`, `TASK-037`, `TASK-041`, `TASK-059`
- `Goal`: Дополировать игровые UI modes.
- `Scope`: Финальные переходы между `SAILING`, `CHAT_FOCUS`, `WINDOW_FOCUS`, `MENU_OPEN`, `RECONNECTING`, `RESPAWN` и большими окнами.
- `Acceptance`:
  - UI modes не конфликтуют друг с другом.
  - Управление и окна ведут себя предсказуемо.

### TASK-062
- `Priority`: `P2`
- `Track`: `Backend`
- `Depends on`: `TASK-035`, `TASK-052`, `TASK-056`
- `Goal`: Добавить structured metrics/logging для MVP эксплуатации.
- `Scope`: Логировать и/или метрифицировать tick lag, wind changes, room count, entity count, reconnects, combat/trade/quest events.
- `Acceptance`:
  - Ключевые runtime события видны в логах/метриках.
  - Проблемы room/wind/combat loop проще диагностировать.

---

## Wave 10 — Persistence и H2 (самая последняя волна)

### TASK-063
- `Priority`: `P2`
- `Track`: `Backend`
- `Depends on`: `TASK-048`, `TASK-056`
- `Goal`: Подготовить persistence boundaries после стабилизации in-memory MVP.
- `Scope`: Ввести repository/store interfaces для профиля, корабля, cargo и quest progress, не ломая текущий in-memory runtime.
- `Acceptance`:
  - Runtime-код не жёстко привязан к конкретной БД.
  - In-memory реализация остаётся основной до подключения `H2`.

### TASK-064
- `Priority`: `P2`
- `Track`: `Backend`
- `Depends on`: `TASK-063`
- `Goal`: Подключить `H2` и `Liquibase` только в самом конце.
- `Scope`: Добавить схему, changelog'и и минимальный persistence adapter поверх `H2`/`R2DBC` для уже существующих contracts.
- `Acceptance`:
  - Backend поднимает БД через `Liquibase`.
  - Схема данных соответствует уже сложившимся in-memory моделям.

### TASK-065
- `Priority`: `P2`
- `Track`: `Backend`
- `Depends on`: `TASK-064`, `TASK-021`
- `Goal`: Реализовать save policy MVP.
- `Scope`: Периодическое сохранение + save on disconnect + save on meaningful events для игрока, трюма и квестов.
- `Acceptance`:
  - Данные игрока не теряются после disconnect/server restart в ожидаемом сценарии.
  - Save policy не ломает realtime/gameplay loop.

### TASK-066
- `Priority`: `P2`
- `Track`: `Frontend`
- `Depends on`: `TASK-065`
- `Goal`: Восстанавливать persistent profile/session state на фронте.
- `Scope`: После повторного входа UI показывает актуальные cargo, balance, active quest, active ship state и last-known profile data.
- `Acceptance`:
  - Пользователь после relogin видит сохранённый прогресс.
  - Persistence не ломает текущие auth/lobby/room flows.

---

## MVP Cut Line
Если нужен самый ранний playable MVP без БД и без лишнего полиша, минимальная линия отсечения такая:
- `TASK-001` … `TASK-057`

Опциональные поздние волны:
- `TASK-058` … `TASK-062` — карта, UX polish и observability
- `TASK-063` … `TASK-066` — persistence и `H2`

Это даст:
- auth;
- lobby;
- create/join room;
- spawn/reconnect basics;
- карты и map templates из resource files;
- wind-driven sailing loop;
- базовый in-game HUD;
- боёвку и respawn;
- loot/cargo/trade;
- NPC, рыбалку и базовый quest loop;
- при этом все игровые структуры данных до последней волны остаются in-memory.





