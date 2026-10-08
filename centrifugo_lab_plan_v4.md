# Лаба: «Centrifugo с нуля» — итоговый план (v4)

Все факты ниже сверены с официальной докой Centrifugo v6, `types.ts` и README `centrifuge-js`, исходниками пакета `denis660/laravel-centrifugo` (`Centrifugo.php`, `CentrifugoBroadcaster.php`), докой k6. То, что можно проверить только запуском, вынесено в явные эксперименты E1–E8 (приложение C) с ожидаемым результатом.

**Сквозной проект:** Realtime Workspace (Laravel + Centrifugo + Vue). Centrifugo в Docker.
**Объём:** 21 сессия, ~62 часа, каждая сессия ≤ 3.5 ч (кроме финала). В сессии 5–7 шагов. После каждой сессии мини-задание на 15–30 минут (всего ещё около 7 часов).
**Формат шага:** проба → проверяемый результат → вопрос на понимание. Ссылки на доку в заголовке сессии.
**Репозиторий:** отдельный standalone-репозиторий, не зависит от других лаб.
**Версии:** Centrifugo ≥ 6.9.0 (`getState` нужен ≥ 6.8.0, `emulated_headers` ≥ 6.9.0). `centrifuge-js` v5.x (совместим с Centrifugo v6). Laravel 13 / PHP 8.4. Redis ≥ 6.2. PostgreSQL. Пакет `denis660/laravel-centrifugo` заявляет проверку до Centrifugo v6.8.0, поэтому в сессии 11 есть smoke-тест совместимости (эксперимент E4).
**Вне лабы (PRO):** CEL-выражения, capabilities, token revocation, Connections API, channel patterns, labels, per-namespace engines.

---

## Уровень и сложность

**Итоговая сложность лабы: 3 из 5.** Целевой уровень: от уверенного junior до начала middle. Сложность растёт по ходу: первые сессии лёгкие, пик в сессиях 8B, 10, 13B, 15B и в финале.

### Что нужно знать до старта (минимум)
- **HTTP и JSON:** запрос и ответ, заголовки, коды ответа, `curl`.
- **JavaScript:** `async/await`, промисы, события, `fetch`.
- **Laravel (базовый уровень):** маршруты, контроллеры, `.env`, миграции. Events, очереди, Policy и транзакции объясняются по ходу.
- **Docker из практики:** `docker compose up`, порты, переменные окружения, логи контейнера. Устройство Docker изнутри не нужно.
- **SQL:** `INSERT`, `SELECT`, транзакция (`BEGIN`/`COMMIT`/`ROLLBACK`). Нужно для сессий 5B и 13B.
- **JWT на уровне идеи:** подписанный токен с данными внутри. Детали в сессии 7.

### Что знать не нужно (лаба даёт по ходу или вне лабы)
- **Vue:** хватает `ref`, `onMounted`, `onUnmounted`. Pinia, composables и роутинг не нужны.
- **TypeScript:** не нужен, примеры на обычном JS.
- **OOP глубже обычных классов:** паттерны и SOLID не требуются.
- **Вне лабы:** WebSocket-протокол изнутри, Redis изнутри, Kubernetes, Kafka, Protobuf, PRO-функции Centrifugo. Nginx достаточно скопировать из приложения D и понять каждую директиву.

### Сложность по сессиям (1 — легко, 5 — самое тяжёлое)

| Сессия | Тема | Сложность | Что сложного |
|---|---|---|---|
| 1 | Что такое realtime | 1 | ничего, знакомство |
| 2 | Конфиг, API, namespaces | 2 | много новых названий опций |
| 3 | centrifuge-js и Vue | 2 | жизненный цикл клиента и подписок |
| 4 | Laravel → Centrifugo | 2 | ответ `200` с ошибкой внутри |
| 5A | Event → Listener → Queue | 2 | различать очередь и канал |
| 5B | Надёжная публикация | 3 | транзакции и `afterCommit` |
| 6 | Протокол сообщений | 2 | дисциплина проектирования |
| 7 | Connection JWT | 3 | claims и обновление токена |
| 8A | Subscription tokens | 3 | порядок проверки прав |
| 8B | Proxy и выбор механизма | 4 | пять механизмов, надо выбирать между ними |
| 9 | Presence и History | 3 | граница «БД против history» |
| 10 | Recovery и reconnect | 4 | `epoch`, `offset`, `recovered`, гонки |
| 11 | HTTP API и пакет | 3 | чтение чужого кода и ловушки пакета |
| 12A | Уведомления и прогресс | 2 | в основном сборка из уже изученного |
| 12B | Typing и доска | 3 | client publish и права |
| 13A | Аварии | 2 | наблюдение, а не разработка |
| 13B | Outbox | 4 | триггеры Postgres, LISTEN/NOTIFY, at-least-once |
| 14 | Тесты и безопасность | 2 | рутинная работа |
| 15A | Production | 3 | прокси и compose |
| 15B | Масштаб и нагрузка | 4 | лимиты ОС, k6, снятие кадров протокола |
| 16 | Финальный проект | 5 | всё вместе, без пошаговых подсказок |

### Как проходить
- Идти по порядку. Сессии 1–3 не пропускать: на них держится всё остальное.
- Если сессия слишком тяжёлая, оставить шаги с проверяемым результатом. Вопросы на понимание можно отложить, но не выкидывать.
- Самые «умственные» сессии: 8B (держать рядом матрицу из 8B.7) и 10 (схема `epoch`/`offset`/`recovered` на бумаге).

---

## Главная модель

```
Postgres (источник истины) ← Laravel (права, бизнес-логика)
        │ commit → Event → Queue (или outbox) → HTTP API publish
        ▼
   Centrifugo (доставка: connections, namespaces, presence, history/recovery)
        │ WebSocket
        ▼
   Vue + centrifuge-js
```

Четыре правила на всю лабу:
1. Centrifugo **доставляет**, но не хранит истину. История — ограниченный кэш, он может быть пустым или потеряться.
2. Publish только **после commit** транзакции.
3. Redis queue ≠ Redis engine Centrifugo ≠ Centrifugo channel.
4. Опции живут на уровне **namespace**. Канал без `:` попадает в `channel.without_namespace`.

---

## Каналы проекта

| Namespace | Пример канала | Для чего | Ключевые опции |
|---|---|---|---|
| `chat` | `chat:room-general` (приватный: `$chat:room-general`) | чат, presence | `presence`, `join_leave`, `history_size`, `history_ttl`, `force_recovery`, `allow_presence_for_subscriber` |
| `tasks` | `$tasks:123` | события задачи | `history_size`, `history_ttl`, `force_recovery` |
| `notify` | `notify:#42` (автоканал пользователя) | личные уведомления | `allow_user_limited_channels`, `history_size`, `history_ttl` |
| `progress` | `$progress:job-77` | прогресс фоновой задачи | короткая история |
| `typing` | `$typing:room-general` | «печатает…» | без истории, `allow_publish_for_subscriber`, `publication_data_format: "json_object"` |

Правила имён (из доки): только ASCII, до 255 символов (`channel.max_length`). Зарезервированы `:` (namespace), `#` (user-limited), `$` (private-префикс), `/`, `*`, `&`. Имя namespace: `^[-a-zA-Z0-9_]{2,}$`. Канал в несуществующем namespace даёт `102: unknown channel`. Namespace остаётся частью имени при публикации. Для приватного канала namespace определяется после снятия `$`: `$chat:room-1` → namespace `chat`. `$` без токена на подписке даёт `103` сразу.

---

# Блок A. Фундамент

## Сессия 1. Что такое realtime (~2.5 ч)
Доки: [transports overview](https://centrifugal.dev/docs/transports/overview)

| # | Проба | Результат | Вопрос |
|---|---|---|---|
| 1.1 | Одна «лента» через polling, long polling, SSE, WebSocket | Видно разницу в запросах и задержке в DevTools | Почему polling хуже при 10 000 клиентов? |
| 1.2 | Минимальный WS-сервер на Node + браузерный клиент | Сообщения идут в обе стороны | Чем WS-соединение отличается от HTTP-запроса по времени жизни? |
| 1.3 | Закрыть сервер, наблюдать `close`, наивный reconnect | Клиент переподключается | Что будет, если 10 000 клиентов переподключатся одновременно? |
| 1.4 | Два сервера и три клиента: разослать всем | Видна проблема «клиент на другом сервере» | Кто хранит список подписчиков? |
| 1.5 | Centrifugo в Docker, admin UI | Админка открывается | Почему Laravel сам не держит WS? |

**Задание после сессии 1** (~15 мин): В `README.md` лабы нарисовать текстом две схемы: «браузер ↔ Laravel» и «браузер ↔ Centrifugo ← Laravel». Под ними записать 3 причины, почему polling не подходит для 10 000 клиентов. Положить в репозиторий `docker-compose.yml` с Centrifugo.

**Готово, когда:** `docker compose up` поднимает Centrifugo, админка открывается, в README две схемы и 3 пункта.

## Сессия 2. Конфиг, API, каналы, namespaces (~3.5 ч)
Доки: [configuration](https://centrifugal.dev/docs/server/configuration), [server API](https://centrifugal.dev/docs/server/server_api), [channels](https://centrifugal.dev/docs/server/channels)

| # | Проба | Результат | Вопрос |
|---|---|---|---|
| 2.1 | `centrifugo genconfig`, `defaultconfig`, `defaultenv`, `configdoc` | Свой `config.json`: `client.token.hmac_secret_key`, `http_api.key`, `admin.enabled` | Приоритет источников конфига? (файл < env < флаги) |
| 2.2 | `client.allowed_origins` для `http://localhost:5173` | Чужой origin отклоняется | От какой атаки защищает? (CSRF/WebSocket hijacking) |
| 2.3 | `POST /api/publish` с `X-API-Key`, смотреть в admin UI | Сообщение видно | Где передавать ключ: заголовок или `?api_key=`? |
| 2.4 | Publish в `unknown:chat` | `200 OK` с телом `error: 102`. Затем `X-Centrifugo-Error-Mode: transport` | Почему 200 при ошибке? Как учесть в Laravel? |
| 2.5 | Описать namespace `chat`, подписаться на `chat:room-general` и `chat2:x` | Первая работает, вторая даёт `102` | Что решает namespace? |
| 2.6 | Публикация в `room-general` вместо `chat:room-general` | Подписчик ничего не получил | Почему префикс — часть имени? |
| 2.7 | `history_size` без `history_ttl`, затем с ним | Только с обоими история работает | Что будет с историей при рестарте (memory engine)? |

⚠️ `client.insecure` показываем один раз как dev-режим и убираем. Env-имена вычисляем по правилу `CENTRIFUGO_` + путь через `_` и сверяем командой `centrifugo defaultenv`.

**Задание после сессии 2** (~20 мин): Сохранить в репозиторий `config.json` проекта (namespaces `chat` и `tasks` из приложения A, секреты из env) и скрипт `scripts/publish.sh {channel}`, который публикует через `curl` и печатает тело ответа.

**Готово, когда:** `publish.sh chat:room-general` печатает пустой `result`, а `publish.sh unknown:x` печатает `error` с кодом `102`.

## Сессия 3. centrifuge-js и Vue (~3 ч)
Доки: [client API spec](https://centrifugal.dev/docs/transports/client_api), [centrifuge-js](https://github.com/centrifugal/centrifuge-js), файл `types.ts` в `node_modules/centrifuge`

| # | Проба | Результат | Вопрос |
|---|---|---|---|
| 3.1 | `npm i centrifuge`, открыть `types.ts` и найти `Options` и `SubscriptionOptions` | Список опций у студента перед глазами | Какие опции у клиента, какие у подписки? |
| 3.2 | `new Centrifuge(url, {getToken})`, события `state`, `connecting`, `connected`, `disconnected`, `error` | Лог состояний | Чем `connecting` отличается от `connected`? |
| 3.3 | `newSubscription`, события `subscribing`, `subscribed`, `publication`, `unsubscribed`, `error` | Приём сообщений | Когда подписка уходит в `unsubscribed`? |
| 3.4 | Токены руками (`gensubtoken`, `jwt.io`) | Подключение по токену | Почему `sub` — строка? |
| 3.5 | `RealtimeChat.vue`: `onMounted` создаёт подписку, `onUnmounted` вызывает `unsubscribe`, `removeSubscription`, `disconnect` | Две вкладки переписываются, утечек нет | Что будет без `removeSubscription` при повторном `newSubscription` на тот же канал? (исключение) |
| 3.6 | Тайминги: `minReconnectDelay` (500 мс), `maxReconnectDelay` (20 000 мс), `timeout` (5000 мс), `maxServerPingDelay` (10 000 мс) | Понятно, где живут клиентские пинг-настройки | Чем `client.ping_interval` на сервере отличается от `maxServerPingDelay` на клиенте? |
| 3.7 | Client publish: `sub.publish` + `allow_publish_for_subscriber` | Работает только при включённой опции | Почему это опасно по умолчанию? |

Fallback-транспорты (`transports: [{transport:'websocket', endpoint}, …]`) показываем одним шагом-демо, в проекте их не используем.

**Задание после сессии 3** (~20 мин): Сделать компонент `ConnectionBadge.vue`: показывает состояние клиента цветом и текстом (`connecting`, `connected`, `disconnected`).

**Готово, когда:** После `docker stop centrifugo` бейдж переходит в `connecting`, после `docker start centrifugo` возвращается в `connected` без перезагрузки страницы.

---

# Блок B. Laravel

## Сессия 4. Laravel → Centrifugo через HTTP API (~3 ч)
Доки: [server API](https://centrifugal.dev/docs/server/server_api)

| # | Проба | Результат | Вопрос |
|---|---|---|---|
| 4.1 | Сервис `RealtimePublisher` на `Http::withHeaders(['X-API-Key'=>…])->post('/api/publish')` | Сообщение доходит до браузера | Почему сначала «голый» HTTP? |
| 4.2 | Проверка поля `error` в теле ответа, исключение при ошибке | Ошибка не теряется молча | Что вернёт API при 200 и `error`? |
| 4.3 | Event, `Dispatchable`, `InteractsWithSockets`, `Queueable`: таблица «trait → что делает» | Таблица в README | Почему `InteractsWithSockets` не делает Laravel WebSocket-сервером? |
| 4.4 | Artisan `realtime:test` отправляет 10 сообщений | 10 сообщений в двух вкладках | Что быстрее: 10 `publish` или один `broadcast`? |
| 4.5 | `broadcast` в несколько каналов и `batch` | Один запрос — несколько публикаций | Гарантирует ли batch порядок между каналами? (нет) |
| 4.6 | `broadcast` с одним неверным каналом | Ответ `200`, ошибка внутри `result.responses[]` | Почему проверять только верхний `error` недостаточно? |

**Задание после сессии 4** (~20 мин): Сделать так, чтобы `RealtimePublisher` бросал исключение, если в теле ответа есть `error`. Добавить в `realtime:test` опцию `--channel`.

**Готово, когда:** `php artisan realtime:test --channel=unknown:x` завершается с кодом 1 и печатает ошибку `102`, а с верным каналом отправляет 10 сообщений.

## Сессия 5A. Event → Listener → Queue (~2.5 ч)

| # | Проба | Результат | Вопрос |
|---|---|---|---|
| 5A.1 | События `TaskCreated`, `TaskUpdated`, `TaskDeleted` | Три события | Зачем отделять событие от публикации? |
| 5A.2 | Listener публикует в `$tasks:{id}` | Realtime при изменении | Где граница между Laravel Event и Centrifugo publication? |
| 5A.3 | `ShouldQueue`, `queue:work`, Redis queue | Публикация идёт из worker | Что быстрее отвечает пользователю? |
| 5A.4 | Схема: Redis queue ≠ Centrifugo channel | Нарисованная схема | Чем они различаются? |
| 5A.5 | Остановить worker, создать 5 задач, запустить | Все 5 событий приходят позже | Что видит пользователь, пока worker выключен? |

**Задание после сессии 5A** (~15 мин): Добавить команду `tasks:create-demo {count=5}`, которая создаёт N задач и диспатчит `TaskCreated` через очередь.

**Готово, когда:** При остановленном worker в таблице `jobs` лежит N строк, после запуска `queue:work` в браузере появляется N сообщений.

## Сессия 5B. Надёжная публикация (~2.5 ч)

| # | Проба | Результат | Вопрос |
|---|---|---|---|
| 5B.1 | Dispatch внутри транзакции, потом rollback | Клиент получил событие о несуществующей задаче | Почему это опасно? |
| 5B.2 | `ShouldDispatchAfterCommit` / `afterCommit` | Баг исчез | Что если commit прошёл, а publish упал? |
| 5B.3 | `tries`, `backoff`, `failed_jobs` | Повтор после недоступного Centrifugo | Может ли повтор дать дубль? |
| 5B.4 | `idempotency_key` в собственном сервисе (пакет такого параметра не принимает) | Дубль подавлен | Сколько живёт ключ? (окно 5 минут, per channel, Memory и Redis engine) |
| 5B.5 | Предпросмотр нативного outbox-consumer (Postgres), делаем в 13B | Понятна альтернатива | Чем outbox лучше «dispatch после commit»? |

**Задание после сессии 5B** (~20 мин): Добавить демо-маршрут `/demo/rollback`: в транзакции создаёт задачу, диспатчит событие и откатывает транзакцию. Прогнать без `afterCommit` и с ним, результат записать в README двумя строками «до» и «после».

**Готово, когда:** Без `afterCommit` клиент получил событие о несуществующей задаче, с `afterCommit` не получил ничего.

---

## Сессия 6. Протокол сообщений (~2.5 ч)

| # | Проба | Результат | Вопрос |
|---|---|---|---|
| 6.1 | Envelope: `event_id`, `type`, `version`, `occurred_at`, `data` | `PROTOCOL.md` | Зачем `version`? |
| 6.2 | Именование типов `task.updated` и реестр | Реестр типов | Чем плохи типы вроде `update2`? |
| 6.3 | Fat против thin событий | Решение по каждому типу | Когда лучше слать «сигнал обновиться»? |
| 6.4 | Дедупликация на клиенте по `event_id` | Повтор не применяется дважды | Почему событие может прийти повторно? |
| 6.5 | `version` и `version_epoch` в publish (только каналы с history, Centrifugo ≥ 6.2) | Устаревшая версия игнорируется | Когда это безопасно? (когда публикация несёт полное состояние) |
| 6.6 | Frontend dispatcher: реестр обработчиков | Один обработчик на тип | Как добавить тип без правки ядра? |
| 6.7 | `publication_data_format: "json_object"` | Невалидный payload отклонён | Что будет без этой опции? (не-JSON доходит, а JSON-клиент отключается с кодом 3506) |
| 6.8 | Драйвер Laravel кладёт имя события в `payload.event` | Сопоставление с вашим envelope | Как совместить `event` драйвера и `type` протокола? |

**Задание после сессии 6** (~25 мин): В `PROTOCOL.md` описать 5 типов событий (`task.created`, `task.updated`, `task.deleted`, `chat.message`, `notify.new`): поля envelope и пример JSON для каждого.

**Готово, когда:** Каждый из 5 примеров опубликован один раз через `curl` и обработан frontend-dispatcher без ошибок в консоли, повтор с тем же `event_id` не применяется второй раз.

---

# Блок C. Безопасность

## Сессия 7. Аутентификация: connection JWT (~3.5 ч)
Доки: [authentication](https://centrifugal.dev/docs/server/authentication)

| # | Проба | Результат | Вопрос |
|---|---|---|---|
| 7.1 | Authentication против authorization | Две схемы | Что решает токен соединения, а что — права на канал? |
| 7.2 | Разбор JWT: `sub` (строка), `exp`, `iat`, `info`, `meta`, `channels`, `subs` | Расшифрованный токен | Что будет без `exp`? (соединение не истекает) |
| 7.3 | Laravel выдаёт токен (HS256) | Эндпоинт `/centrifugo/connection-token` | Где живёт секрет? |
| 7.4 | Короткий `exp` + `getToken`, `UnauthorizedError` при 403 | Токен обновляется сам, при отказе клиент уходит в `disconnected` | Что происходит после истечения? (grace около 25 с) |
| 7.5 | Анонимы: пустой `sub`, `client.allow_anonymous_connect_without_token`, `client.disallow_anonymous_connection_tokens` | Разница между анонимом и пользователем | Когда допустим аноним? |
| 7.6 | `client.token.audience` и `issuer` | Чужой токен отклоняется | Зачем audience? |
| 7.7 | Ротация: `hmac_previous_secret_key`, `hmac_previous_secret_key_valid_until` (≥ 6.6.1) | Старые токены живут до срока | Как сменить секрет без массового разрыва? |

**Задание после сессии 7** (~20 мин): Выдавать connection-токен с `exp` через 5 минут. В README записать три строки: что лежит в токене, где живёт секрет, что происходит после истечения.

**Готово, когда:** Токен расшифровывается на jwt.io, подпись проверяется вашим секретом, клиент без токена получает отказ.

## Сессия 8A. Subscription tokens и private channels (~2.5 ч)
Доки: [permissions](https://centrifugal.dev/docs/server/channel_permissions), [channel token auth](https://centrifugal.dev/docs/server/channel_token_auth)

Порядок проверки подписки в Centrifugo: private-prefix gate (`$` без токена → отказ) → subscription token → user-limited канал → subscribe proxy → `allow_subscribe_for_client`. Побеждает первый сработавший.

| # | Проба | Результат | Вопрос |
|---|---|---|---|
| 8A.1 | Публичный namespace с `allow_subscribe_for_client` | Любой авторизованный подписывается | Почему это «публичный» namespace? |
| 8A.2 | Убрать опцию, подписаться без токена | `103: permission denied` | Чем 102 отличается от 103? |
| 8A.3 | Laravel выдаёт subscription JWT (`sub`, `channel`, `exp`) | Подписка через `getToken(ctx)` | Должен ли `sub` совпадать с connection-токеном? (да) |
| 8A.4 | Policy: пользователь 42 не получает токен на `tasks:999` | Негативный тест | Где проверяется право? |
| 8A.5 | `info` в subscription-токене | Имя в presence | Чьи данные попадают в `info`? |
| 8A.6 | `exp` и `expire_at`; пустая строка из `getToken` | Подписка продлевается или отзывается | Что значит пустой токен? (права больше нет, подписка снимается) |
| 8A.7 | `$` + `allow_subscribe_for_client` | Без токена на `$…` отказ даже в «публичном» namespace | Зачем это ограничение? |

**Задание после сессии 8A** (~20 мин): Написать негативный тест (или `curl`-сценарий): пользователь 42 просит subscription-токен на `$tasks:{id}` чужой задачи.

**Готово, когда:** Для чужой задачи Laravel отвечает `403`, для своей выдаёт токен, и подписка с ним проходит.

## Сессия 8B. Proxy, user-limited, выбор механизма (~2.5 ч)
Доки: [proxy](https://centrifugal.dev/docs/server/proxy)

| # | Проба | Результат | Вопрос |
|---|---|---|---|
| 8B.1 | User-limited канал `notify:#42` и `allow_user_limited_channels` | Подписывается только пользователь 42 | Что без опции? (`103`) |
| 8B.2 | Connect proxy: `client.proxy.connect.enabled`, `endpoint`, `timeout` (1 с), ответ `{"result":{"user":"56"}}` | Аутентификация без JWT | Что дороже при массовом reconnect: JWT или connect proxy? |
| 8B.3 | `http_headers` и `emulated_headers` (≥ 6.9.0), клиент `headers` / `setHeaders` | Laravel видит cookie/Authorization | Почему emulated-заголовки — недоверенный ввод? |
| 8B.4 | Subscribe proxy: `subscribe_proxy_enabled` в namespace | Laravel решает по каждой подписке | Каких подписок proxy не видит? (с токеном и user-limited) |
| 8B.5 | Ответ proxy: `error` (коды 400–1999) и `disconnect` (4000–4499 reconnect, 4500–4999 terminal, reason ≤ 32 байт) | Клиент либо возвращается, либо нет | Чем terminal-код отличается от reconnect-кода? |
| 8B.6 | Ошибка 500 от Laravel: `status_to_code_transforms` | Понятно поведение при сбое | Что увидит клиент при недоступном endpoint? (`100: internal server error`, временная ошибка, повтор) |
| 8B.7 | Матрица: JWT / subscription token / proxy / user-limited / server-side | Таблица «когда что» | Что выбрать для личных уведомлений, а что для приватных задач? |

**Задание после сессии 8B** (~20 мин): Заполнить в README матрицу решений для 5 каналов проекта (`chat`, `tasks`, `notify`, `progress`, `typing`): какой механизм доступа выбран и почему, одной фразой.

**Готово, когда:** В таблице 5 строк, в каждой названа причина выбора, а не просто механизм.

---

# Блок D. Состояние и надёжность

## Сессия 9. Presence и History (~3.5 ч)
Доки: [presence](https://centrifugal.dev/docs/server/presence), [history and recovery](https://centrifugal.dev/docs/server/history_and_recovery), [FAQ про масштаб presence](https://centrifugal.dev/docs/faq)

| # | Проба | Результат | Вопрос |
|---|---|---|---|
| 9.1 | `presence: true`, затем `sub.presence()` без и с `allow_presence_for_subscriber` | Без опции `103`, с опцией список `clients` | Что вернёт вызов, если presence не включён? (`108: not available`) |
| 9.2 | `presenceStats()` → `numClients`, `numUsers` | Только счётчики | Когда stats вместо полного presence? |
| 9.3 | `info` из connection JWT и `chan_info` из subscription JWT | Видны в ответе presence | Что нельзя класть в `info`? |
| 9.4 | `join_leave` + `force_push_join_leave` или клиентский `joinLeave: true` | События `join`/`leave` | Чем опасен `join_leave` в больших каналах? (N² при массовом reconnect, доставка at-most-once) |
| 9.5 | UI «Online: 3» + список, один пользователь в двух вкладках | Два client ID у одного user | Как считать «онлайн-пользователей»? |
| 9.6 | `history_size`, `history_ttl`, `sub.history({limit: …})` | Последние сообщения | Что вернёт history без `limit`? (только позиция) |
| 9.7 | Чат: история из Postgres + живые обновления из Centrifugo, пагинация из Laravel | История из БД | Почему history Centrifugo — не источник истины? |

⚠️ Centrifugo не шлёт hook-события `disconnect`/`unsubscribe` на бэкенд. «Онлайн» считаем по presence, а не по hook.

**Задание после сессии 9** (~25 мин): Добавить в чат строку «Онлайн: N клиентов, M пользователей» и список имён из presence.

**Готово, когда:** Откройте 3 вкладки: две под одним пользователем и одну под другим. Должно показать 3 клиента и 2 пользователя, после закрытия одной вкладки числа обновляются.

## Сессия 10. Recovery и reconnect (~3 ч)

| # | Проба | Результат | Вопрос |
|---|---|---|---|
| 10.1 | Lifecycle клиента, backoff с jitter | Видны попытки с растущей паузой | Зачем jitter? |
| 10.2 | `epoch` и `offset` в `publication.offset`, `streamPosition` | Понимание позиции в потоке | Что значит смена epoch? |
| 10.3 | `force_recovery` в namespace; Wi-Fi off/on во время чата | Пропущенные сообщения возвращаются | Как recovery связан с positioning? (recovery включает positioning) |
| 10.4 | Флаги `wasRecovering` и `recovered` в `subscribed` | Логика: `wasRecovering && !recovered` → перезагрузить из Laravel | Что означает `recovered: true` при нуле реплеев? |
| 10.5 | Вытеснение истории: пропустить больше `history_size` | `recovered: false` | Какие лимиты? (`client.recovery_max_publication_limit` = 300) |
| 10.6 | Клиентский `positioned`/`recoverable` вместо `force_*` | Требует права на history (`allow_history_for_subscriber`) | Что вернёт сервер без этого права? (эксперимент E7) |
| 10.7 | `getState` (Centrifugo ≥ 6.8.0): позицию читать **до** загрузки данных | SDK вызывает его на первом subscribe и при `112`, но не при успешном recovery | Почему порядок «позиция → данные» важен? |
| 10.8 | Код `3010 insufficient state` при потере на живом соединении | Reconnect и recovery | Что такое positioning? |

**Задание после сессии 10** (~25 мин): Включить в DevTools режим Offline, отправить через `curl` 5 сообщений в чат, вернуть сеть. Затем поставить `history_size` равным 3 и повторить. В README записать значения `wasRecovering` и `recovered` для обоих случаев.

**Готово, когда:** В первом случае все 5 сообщений появились без дублей, во втором `recovered: false` и интерфейс перезагрузил историю из Laravel.

## Сессия 11. HTTP API вглубь и Laravel-пакет (~3.5 ч)
Доки: [server API](https://centrifugal.dev/docs/server/server_api), [пакет denis660/laravel-centrifugo](https://github.com/denis660/laravel-centrifugo) (читаем `Centrifugo.php` и `CentrifugoBroadcaster.php`)

| # | Проба | Результат | Вопрос |
|---|---|---|---|
| 11.0 | `composer require denis660/laravel-centrifugo`, `php artisan centrifuge:install`, smoke-тест с Centrifugo ≥ 6.9 (эксперимент E4) | Публикация и токен работают | Что меняет инсталлер в `broadcasting.php` и `.env`? |
| 11.1 | Чтение кода: что пакет умеет и чего нет. Умеет: `publish(channel, array, skipHistory)`, `broadcast`, `presence`, `presenceStats`, `history`, `historyRemove`, `subscribe`, `unsubscribe`, `disconnect`, `channels`, `info`. Не умеет: `idempotency_key`, `tags`, `version`, `batch`, `refresh`, `skip_history` кроме флага | Таблица «пакет против прямого API» | Что делать для недостающего? (свой сервис из сессии 4) |
| 11.2 | Ошибки: `send()` глотает `GuzzleException` и возвращает массив с `error`; `broadcast()` драйвера проверяет только верхний `error` | Тест: unknown-namespace в `broadcast` не бросает исключение (эксперимент E2) | Как обернуть, чтобы не терять ошибки? |
| 11.3 | `ShouldBroadcast` + `PrivateChannel('chat:room-1')` → `private-chat:room-1` → `$chat:room-1`; `Broadcast::channel('chat:room-{id}')` (эксперимент E3) | Приватный канал в namespace `chat` | Почему именно `$` и как Centrifugo находит namespace? |
| 11.4 | `/broadcasting/auth`: ответ `{channels:[{channel, token, info}]}`, клиентский `getToken` | Подписка по токену через Laravel | Что приходит при отказе? (`{channel, status: 403}`) |
| 11.5 | Токены: `generateConnectionToken(userId, ttlSeconds, info, channels)` и `generatePrivateChannelToken(userId, channel, ttlSeconds, info)`. `exp` — TTL в секундах. При `0` claim `exp` **не добавляется** | Токены с TTL | Что опасного в `auth()` драйвера? (выдаёт subscription-токены с TTL 0 и пустым `info`) |
| 11.6 | Свой контроллер выдачи subscription-токенов с TTL и `info` вместо дефолтного `auth()` | Токены живут минуты, в presence видно имя | Какие каналы лучше выдавать вручную? |
| 11.7 | Что пакет не поддерживает: `PresenceChannel` не конвертируется и теряет namespace | Решение: presence делаем средствами Centrifugo | Почему не использовать `PresenceChannel`? |
| 11.8 | Бан пользователя: `disconnect` + короткий `exp` | Пользователь выброшен и не вернётся | Почему без `exp` нельзя «разлогинить» иначе, чем `disconnect`? |
| 11.9 | Правило: когда «прямой publish», когда «Event → Queue → Centrifugo» | Раздел в README лабы | Что выбрать для чата, что для прогресса? |

**Задание после сессии 11** (~25 мин): В README составить таблицу «пакет против прямого сервиса» для 5 операций: `publish`, `broadcast`, `idempotency_key`, `presence`, выдача токена. Прогнать эксперимент E2 и записать результат одной строкой.

**Готово, когда:** В таблице у каждой операции отметка «умеет / не умеет», а в строке про E2 написано, бросил ли драйвер исключение при неверном канале.

## Сессия 12A. Паттерны: уведомления, прогресс, server-side подписки (~2.5 ч)
Доки: [server-side subscriptions](https://centrifugal.dev/docs/server/server_subs)

| # | Проба | Результат | Вопрос |
|---|---|---|---|
| 12A.1 | Автоканал: `client.subscribe_to_user_personal_channel.enabled` + `personal_channel_namespace: "notify"` → канал `notify:#42` | Уведомление приходит без `subscribe()` | Что при трёх вкладках? (в каждую) |
| 12A.2 | Приём через `client.on('publication', ctx => ctx.channel)` | Работает без Subscription-объекта | Чем server-side подписка отличается от клиентской? |
| 12A.3 | Ещё способы: `channels`/`subs` в JWT, `channels` из connect proxy, API `subscribe` | Таблица способов | Что даёт `subs` по сравнению с `channels`? (`data`, `info`, `override`) |
| 12A.4 | `single_connection` (нужно `presence` в личном namespace) | Старые вкладки закрываются | Почему на него нельзя опереться в бизнес-логике? (at-most-once) |
| 12A.5 | Прогресс job: publish 25/50/75/100 в `$progress:job-77` | Прогресс-бар | Как не заспамить канал? |
| 12A.6 | `skip_history` для служебных событий | Не попадают в историю | Когда история не нужна? |
| 12A.7 | `client.presence(channel)`, `client.history(channel)` для server-side каналов | Вызовы верхнего уровня | Нужны ли права на них? (да, как для обычных каналов) |

**Задание после сессии 12A** (~15 мин): Добавить команду `notify:send {userId} {text}`, которая публикует в личный канал `notify:#{userId}`.

**Готово, когда:** При трёх открытых вкладках пользователя уведомление пришло во все три, у другого пользователя не пришло.

## Сессия 12B. Паттерны: typing, client publish, доска (~2.5 ч)

| # | Проба | Результат | Вопрос |
|---|---|---|---|
| 12B.1 | Typing в `$typing:room-general` без истории, `allow_publish_for_subscriber` | «Nikita is typing…» | Почему здесь допустим client publish? |
| 12B.2 | Debounce/throttle и авто-таймаут | Не больше 1 события в секунду | Почему не на каждый keydown? |
| 12B.3 | Publish proxy: Laravel валидирует публикацию клиента | Невалидный payload отклонён | Кто решает при включённом proxy? (proxy перебивает все `allow_publish_*`) |
| 12B.4 | Server-side publish против client-side publish | Таблица | Что выбрать для сообщений чата? (Laravel сохраняет в БД, потом publish) |
| 12B.5 | Сборка доски: задачи, онлайн, уведомления, чат, typing, прогресс | Realtime-доска | Сколько каналов у клиента? (лимит `client.channel_limit` = 128) |

**Задание после сессии 12B** (~20 мин): Добавить к typing-индикатору debounce 1 секунда и автотаймаут 3 секунды.

**Готово, когда:** За 20 быстрых нажатий уходит не больше одного события в секунду, через 3 секунды после остановки надпись «печатает…» пропадает сама.

## Сессия 13A. Reliability: аварии (~2.5 ч)

| # | Проба | Результат | Вопрос |
|---|---|---|---|
| 13A.1 | Остановить Centrifugo | Клиенты переподключаются, API-вызовы падают | Что делает Laravel? |
| 13A.2 | Остановить Laravel | Realtime идёт, новые токены не выдаются | Что при истечении токена? |
| 13A.3 | Остановить queue worker | Задержка событий | Как заметить зависший worker? |
| 13A.4 | Остановить Redis | Падает очередь (и история, если Redis engine) | Что работает без Redis? |
| 13A.5 | Потеря сети у клиента | Disconnect → reconnect → recovery | Когда recovery не хватит? |
| 13A.6 | Таблица «проблема → что видит пользователь → как восстанавливаем» | Заполненная таблица | Где остались дыры? |

**Задание после сессии 13A** (~20 мин): Заполнить таблицу аварий (Centrifugo, Laravel, worker, Redis, сеть, дубликат) по тому, что вы реально увидели в экспериментах, а не по памяти. Отдельной строкой назвать одну «дыру», которую пока нечем закрыть.

**Готово, когда:** Таблица из 6 строк с тремя столбцами, строка про «дыру» есть.

## Сессия 13B. Reliability: outbox, дубликаты, порядок (~2.5 ч)
Доки: [consumers](https://centrifugal.dev/docs/server/consumers)

| # | Проба | Результат | Вопрос |
|---|---|---|---|
| 13B.1 | Таблица `centrifugo_outbox` (`id BIGSERIAL`, `method`, `payload JSONB`, `partition`, `created_at`) | Таблица создана | Чем outbox надёжнее dispatch после commit? |
| 13B.2 | Конфиг `consumers[]`: `enabled`, `name`, `type: "postgresql"`, `postgresql.dsn`, `outbox_table_name`, `num_partitions`, `partition_select_limit`, `partition_poll_interval` | Centrifugo сам читает таблицу | Что значит «at-least-once»? (внутренние ошибки повторяются) |
| 13B.3 | В транзакции Laravel: `INSERT` задачи + `INSERT` в outbox (`method='publish'`, `payload={"channel":…,"data":…}`) | Откат убирает и событие | Что за relay-процесс заменил Centrifugo? |
| 13B.4 | `LISTEN/NOTIFY`: триггер + `partition_notification_channel` | Задержка падает | Почему PgBouncer в transaction-режиме несовместим? (нужен `partition_notification_dsn`) |
| 13B.5 | `broadcast` в outbox: при ошибке одного канала команда повторяется целиком | Дубли без ключа | Как избежать? (`idempotency_key` в payload + дедуп на клиенте) |
| 13B.6 | Порядок: внутри `partition` строгий | Понятны гарантии | Что с порядком между разными каналами? (не гарантирован) |
| 13B.7 | Читаются только команды, меняющие состояние; `batch` не поддерживается; только JSON | Знаем ограничения | Что нельзя отправить через outbox? (чтение, `batch`, бинарные данные) |

⚠️ Бонус без внедрения: экспериментальный PostgreSQL stream broker (`broker.type: "postgres"`, PG 16+, `cf_stream_publish()` внутри транзакции).

**Задание после сессии 13B** (~25 мин): Сделать создание задачи одной транзакцией: `INSERT` в `tasks` и `INSERT` в `centrifugo_outbox`. Добавить флаг для демо, который бросает исключение после обоих `INSERT`.

**Готово, когда:** В обычном режиме событие доходит до браузера, в режиме с исключением нет ни задачи, ни события.

---

# Блок E. Качество и продакшен

## Сессия 14. Тестирование и безопасность (~3 ч)

| # | Проба | Результат | Вопрос |
|---|---|---|---|
| 14.1 | `Http::fake()`: URL, `X-API-Key`, канал, payload | Feature-тест публикации | Что проверять: payload или факт вызова? |
| 14.2 | Тесты выдачи токенов и Policy | Негативные кейсы | Какие кейсы обязательны? |
| 14.3 | Тест `afterCommit` и идемпотентности | Тест на rollback | Как проверить, что при rollback ничего не ушло? |
| 14.4 | Чек-лист: секреты в env, `allowed_origins`, `http_server.internal_port` для API/admin/metrics, `client.insecure` и `http_api.insecure` выключены | Чек-лист пройден | Что нельзя публиковать в интернет? (`/api`, admin, `/metrics`, `/health`, `/debug`) |
| 14.5 | XSS в чате, лимиты размера, `channel_regex` (без префикса namespace) | Защита от мусорных каналов | Зачем `channel_regex`? |
| 14.6 | `client.user_connection_limit`, `client.connection_limit`, `client.connection_rate_limit` | Лимиты работают | Почему `user_connection_limit` действует на одну ноду? |

**Задание после сессии 14** (~25 мин): Написать 3 теста: публикация (`Http::fake()` проверяет URL, `X-API-Key`, канал и payload), отказ в subscription-токене чужому пользователю, rollback ничего не публикует.

**Готово, когда:** `php artisan test` зелёный, все 3 теста на месте.

## Сессия 15A. Production: compose, прокси, engine (~3 ч)
Доки: [load balancing](https://centrifugal.dev/docs/server/load_balancing), [engines](https://centrifugal.dev/docs/server/engines), [observability](https://centrifugal.dev/docs/server/observability)

| # | Проба | Результат | Вопрос |
|---|---|---|---|
| 15A.1 | Docker Compose: app, nginx, centrifugo, redis, postgres, queue | Всё поднимается одной командой | Что публично, что внутри? |
| 15A.2 | Nginx по конфигу из приложения D: `location /connection/` с `Upgrade`/`Connection`, `proxy_http_version 1.1`, `proxy_buffering off`, `proxy_read_timeout 60s`; `location /` для `/emulation` | WS не рвётся через минуту | Почему таймаут должен быть выше ping-интервала (25 с)? |
| 15A.3 | Чего не нужно: sticky sessions не требуются даже для SSE/HTTP-streaming (`/emulation` идёт на любую ноду) | Round-robin работает | Почему нет нужды в stickiness? |
| 15A.4 | `engine.type: "redis"`, `engine.redis.address`, `presence_ttl` | История переживает рестарт Centrifugo | Что теряет memory engine? |
| 15A.5 | `health.enabled`, `prometheus.enabled`, логи, `shutdown.timeout` (30 с), SIGHUP | Метрики и health | Что перечитывает SIGHUP? (токены и опции каналов) |
| 15A.6 | Опционально: тот же прокси на Caddy (`flush_interval -1`) или Traefik (`responseForwarding.flushInterval: "10ms"`) | Работает на привычном прокси | Что общего во всех прокси? (upgrade, no buffering, timeouts, `/emulation`) |

**Задание после сессии 15A** (~20 мин): Выполнить `docker compose down -v` и поднять стенд заново. Закрыть в nginx `/api` и `/admin` для внешнего порта (`return 404`).

**Готово, когда:** Стенд поднимается одной командой, `curl` на `/api/publish` через nginx даёт `404`, WebSocket-соединение через nginx живёт дольше 2 минут.

## Сессия 15B. Масштаб и нагрузка (~3.5 ч)
Доки: [infra tuning](https://centrifugal.dev/docs/server/infra_tuning), [client protocol](https://centrifugal.dev/docs/transports/client_protocol), [k6 WebSockets](https://grafana.com/docs/k6/latest/using-k6/protocols/websockets/), пример [proxy_load_balancing](https://github.com/centrifugal/examples/tree/master/v6/proxy_load_balancing)

| # | Проба | Результат | Вопрос |
|---|---|---|---|
| 15B.1 | Две ноды Centrifugo + Redis engine + балансировщик | Публикация на одной ноде видна на другой | Зачем Redis при двух нодах? |
| 15B.2 | Лимиты: `ulimit -n`/`LimitNOFILE` (в пакетах 65536), Docker `ulimits`, `worker_rlimit_nofile`, `worker_connections`; по 2 fd на проксированное соединение | Лимиты выставлены | Почему лимит нужен и у прокси? |
| 15B.3 | Ephemeral ports (`ip_local_port_range`), TIME_WAIT (`ss -tan state time-wait \| wc -l`), conntrack | Знаем симптом `99: Cannot assign requested address` и 502 | Как лечить? (диапазон портов, больше инстансов, виртуальные интерфейсы) |
| 15B.4 | Снять реальные кадры клиента в DevTools (Network → WS) и сохранить в `PROTOCOL-captured.md` (эксперимент E5) | Кадры `connect`, `subscribe`, `pub`, ping, pong | Чем push отличается от reply? (у push `id` нет) |
| 15B.5 | k6-скрипт на `k6/websockets` (рекомендован; `k6/experimental/websockets` устарел): JWT на VU, `connect` → `subscribe` → приём, ответ на ping пустым кадром | 100 соединений держатся | Что будет, если не отвечать на ping? (клиент считается мёртвым) |
| 15B.6 | Измерение задержки: timestamp в payload, метрика `Trend`; нагрузка 100 → 1 000 → 10 000 | Отчёт: что ломается первым | Упрёмся в CPU, память или fd? |
| 15B.7 | Parsing batched-кадров: один кадр = несколько JSON через `\n` | Скрипт считает все сообщения | Зачем batching? (меньше системных вызовов) |

**Задание после сессии 15B** (~30 мин): Снять в DevTools и сохранить в `PROTOCOL-captured.md` по одному кадру: `connect`, `subscribe`, push с публикацией, ping, pong (пустой кадр). Запустить k6 на 10 виртуальных пользователей на 30 секунд.

**Готово, когда:** В файле 5 кадров с пометкой направления, в отчёте k6 записано, сколько соединений продержалось все 30 секунд.

---

## Сессия 16. Финальный проект (~5 ч)
Студент получает ТЗ без пошагового решения: **Realtime Task Workspace**.
- **Laravel:** авторизация, задачи, события, очередь, outbox, публикация, токены с TTL, Policy, тесты.
- **Centrifugo:** connection JWT, private-каналы (subscription token), presence, history, recovery, server-side publish, личный автоканал.
- **Vue:** состояние соединения, задачи, уведомления, онлайн, чат, typing, reconnect, `getState` или логика `wasRecovering && !recovered`.

Приёмка:
1. A создаёт задачу → Laravel → outbox → Centrifugo → B видит её.
2. A отключает сеть → B продолжает → A возвращается и получает пропущенное (или перезагружает состояние).
3. Пользователь без прав не подписывается на чужой `tasks:*`.
4. Откат транзакции не создаёт событие.
5. Остановка Centrifugo на минуту не ломает данные.
6. Бан пользователя (`disconnect` + короткий `exp`) выбрасывает его и не пускает обратно.

Сдача: тесты, нагрузочный отчёт, `PROTOCOL.md`, схема архитектуры, таблица аварий.

**Задание после сессии 16** (~30 мин): Прогнать вручную 6 приёмочных сценариев из этой сессии и записать в `ACCEPTANCE.md` результат каждого: ✅ или ❌ и одна строка, что именно вы наблюдали.

**Готово, когда:** В файле 6 строк, у каждой есть наблюдение. Каждую ❌ можно превратить в задачу на доработку.

---

# Приложение A. Рабочий config (стартовая точка)

```json
{
  "client": {
    "token": { "hmac_secret_key": "<env>" },
    "allowed_origins": ["http://localhost:5173"],
    "subscribe_to_user_personal_channel": {
      "enabled": true,
      "personal_channel_namespace": "notify"
    }
  },
  "http_api": { "key": "<env>" },
  "admin": { "enabled": true, "password": "<env>", "secret": "<env>" },
  "health": { "enabled": true },
  "prometheus": { "enabled": true },
  "channel": {
    "namespaces": [
      { "name": "chat", "presence": true, "join_leave": true,
        "history_size": 100, "history_ttl": "300s", "force_recovery": true,
        "allow_presence_for_subscriber": true },
      { "name": "tasks", "history_size": 50, "history_ttl": "300s", "force_recovery": true },
      { "name": "notify", "allow_user_limited_channels": true,
        "history_size": 50, "history_ttl": "600s", "force_recovery": true },
      { "name": "progress", "history_size": 5, "history_ttl": "60s" },
      { "name": "typing", "allow_publish_for_subscriber": true,
        "publication_data_format": "json_object" }
    ]
  }
}
```

Env-имена вычисляем по правилу `CENTRIFUGO_` + путь через `_` (например `CENTRIFUGO_CLIENT_TOKEN_HMAC_SECRET_KEY`), сверяем `centrifugo defaultenv`. Namespaces одной строкой через env: `CENTRIFUGO_CHANNEL_NAMESPACES='[{"name":"chat"}]'`.

# Приложение B. Ловушки (источник указан)

| # | Ловушка | Источник |
|---|---|---|
| 1 | API отвечает `200 OK` даже при ошибке. Ошибка в `error`, либо `error_mode: transport` | доки server API |
| 2 | В `broadcast` ошибки отдельных каналов лежат в `result.responses[]`. Драйвер пакета смотрит только верхний `error` | доки + код `CentrifugoBroadcaster.php` (проверить: E2) |
| 3 | `Centrifugo::send()` пакета ловит исключения Guzzle и возвращает массив с `error`, а не бросает | код `Centrifugo.php` |
| 4 | `history_size` без `history_ttl` (и наоборот) — история выключена | доки channels |
| 5 | Канал в несуществующем namespace → `102`. Публиковать надо в точное имя с namespace | доки channels |
| 6 | `$` без токена → `103` сразу, даже при `allow_subscribe_for_client` | доки permissions |
| 7 | Presence/history с клиента требуют `allow_*_for_subscriber/client`. Если фича не включена в namespace, ответ `108`, а не `103` | доки permissions |
| 8 | Клиентский `positioned`/`recoverable` требует права на history, `joinLeave` — права на presence | доки permissions |
| 9 | Publish proxy перебивает все `allow_publish_*` | доки permissions |
| 10 | User-limited каналы работают только с `allow_user_limited_channels` | доки channels |
| 11 | Subscribe proxy не видит подписки по токену и user-limited | доки proxy |
| 12 | Без `exp` соединение не истекает. В пакете при `exp = 0` claim вообще не добавляется. Драйвер выдаёт subscription-токены с TTL 0 и пустым `info` | доки auth + код пакета |
| 13 | `exp` в пакете — TTL в секундах, не timestamp | код пакета |
| 14 | `PresenceChannel` в пакете не конвертируется (`presence-…` без `$` и namespace) | код `CentrifugoBroadcaster.php` |
| 15 | В драйвере имя события лежит в `payload.event` | код пакета |
| 16 | Connect proxy бьёт в Laravel на каждое подключение. JWT лучше при массовом reconnect | доки auth, proxy |
| 17 | `emulated_headers` приходят от клиента, им нельзя доверять без проверки | доки proxy |
| 18 | Memory engine: одна нода, история и presence теряются при рестарте. Redis engine ≥ 6.2 | доки engines |
| 19 | `idempotency_key`: окно 5 минут, per channel, Memory и Redis engine | доки server API |
| 20 | Recovery — не гарантия доставки. При `wasRecovering && !recovered` нужна перезагрузка из Laravel | доки history and recovery |
| 21 | `getState` в SDK не вызывается при успешном recovery. Позицию читать **до** данных | `types.ts`, README `centrifuge-js` |
| 22 | Centrifugo не шлёт hook-события `disconnect`/`unsubscribe` | доки proxy |
| 23 | `single_connection` — at-most-once, на него нельзя опираться в бизнес-логике | доки server subs |
| 24 | Outbox-consumer: только команды, меняющие состояние, JSON, без `batch`. Ошибка одного канала `broadcast` повторяет команду целиком | доки consumers |
| 25 | PgBouncer (transaction pooling) несовместим с LISTEN/NOTIFY, нужен отдельный DSN | доки consumers |
| 26 | Прокси: нужны `Upgrade`, `proxy_buffering off`, таймаут выше ping-интервала, доступный `/emulation` | доки load balancing |
| 27 | `client.insecure` и `http_api.insecure` — только для разработки | доки configuration |
| 28 | Клиентские пинг-настройки: `maxServerPingDelay` (10 000 мс). `client.ping_interval` (25 с) — серверная | `types.ts`, доки configuration |
| 29 | `history({limit: 0})`: по доке сервера возвращает только позицию, а комментарий в `types.ts` говорит иначе | противоречие, эксперимент E1 |

# Приложение C. Эксперименты, которые закрывают остаточные неопределённости

| # | Эксперимент | Ожидаемый результат | Сессия |
|---|---|---|---|
| E1 | `sub.history({limit: 0})` против `{limit: -1}` на канале с историей | Доки сервера: `0` — только `offset`/`epoch`, `-1` — публикации до лимита. Записать фактическое поведение | 9.6 |
| E2 | `broadcast` через драйвер на смесь верного канала и неизвестного namespace | По коду исключения не будет (смотрит только верхний `error`). Подтвердить и обернуть | 11.2 |
| E3 | `PrivateChannel('chat:room-1')` + `Broadcast::channel('chat:room-{id}')` | Канал уходит как `$chat:room-1`, авторизация находит правило (имя без `$`) | 11.3 |
| E4 | Smoke-тест пакета с Centrifugo ≥ 6.9 (publish, token, broadcast) | Работает: HTTP API стабилен. Зафиксировать версии | 11.0 |
| E5 | Снять кадры из DevTools и повторить в k6: нужен ли subprotocol `centrifuge-json`, точный вид ping и pong | JS SDK передаёт `centrifuge-json` в примере своего WS-конструктора. Подтвердить кадры реальной сессией | 15B.4 |
| E6 | Connection JWT без `exp` | Соединение не истекает. Выбросить пользователя только через `disconnect` | 7.2, 11.8 |
| E7 | Клиентский `recoverable: true` без `allow_history_for_subscriber` | Ожидается отказ, зафиксировать код | 10.6 |
| E8 | Cookie и emulated-заголовки в connect proxy (браузер) | Заголовки доходят в Laravel при перечислении в `http_headers`/`emulated_headers` | 8B.3 |

# Приложение D. Конфиг nginx (из доки Centrifugo, адаптирован)

```nginx
worker_rlimit_nofile 1048576;
events { worker_connections 65535; }

http {
  upstream centrifugo { server centrifugo1:8000; server centrifugo2:8000; }

  map $http_upgrade $connection_upgrade { default upgrade; '' close; }

  server {
    listen 80;
    proxy_set_header Host $http_host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    location /connection/ {
      proxy_pass http://centrifugo;
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection $connection_upgrade;
      proxy_buffering off;
      proxy_read_timeout 60s;   # выше client.ping_interval (25s)
      proxy_send_timeout 60s;
    }

    location / { proxy_pass http://centrifugo; }  # в т.ч. /emulation
  }
}
```

# Приложение E. Источники сверки
- Конфиг: https://centrifugal.dev/docs/server/configuration
- Каналы и namespaces: https://centrifugal.dev/docs/server/channels
- Права: https://centrifugal.dev/docs/server/channel_permissions
- JWT: https://centrifugal.dev/docs/server/authentication, https://centrifugal.dev/docs/server/channel_token_auth
- Server API: https://centrifugal.dev/docs/server/server_api
- Proxy: https://centrifugal.dev/docs/server/proxy
- History и recovery: https://centrifugal.dev/docs/server/history_and_recovery
- Presence: https://centrifugal.dev/docs/server/presence
- Server-side подписки: https://centrifugal.dev/docs/server/server_subs
- Engines: https://centrifugal.dev/docs/server/engines
- Consumers: https://centrifugal.dev/docs/server/consumers
- Балансировка: https://centrifugal.dev/docs/server/load_balancing
- Тюнинг ОС: https://centrifugal.dev/docs/server/infra_tuning
- Клиентский API и протокол: https://centrifugal.dev/docs/transports/client_api, https://centrifugal.dev/docs/transports/client_protocol
- `centrifuge-js`: README и `src/types.ts` (github.com/centrifugal/centrifuge-js)
- Пакет: README, `src/Centrifugo.php`, `src/CentrifugoBroadcaster.php` (github.com/denis660/laravel-centrifugo)
- k6 WebSockets: https://grafana.com/docs/k6/latest/using-k6/protocols/websockets/
