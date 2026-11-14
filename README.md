# Centrifugo Lab — realtime с нуля

Лаба про realtime на [Centrifugo](https://centrifugal.dev): WebSocket-соединения, каналы и namespaces, `centrifuge-js`, публикация из Laravel, JWT и subscription tokens, proxy, presence и history, recovery и reconnect, Outbox, тесты и безопасность, production (nginx, Redis engine) и нагрузка. Сквозной проект — **Realtime Workspace** (Laravel + Centrifugo + Vue; Centrifugo в Docker).

**Статус: 🟡 методичка написана (11.10.2026); сессии 1–3 прогнаны в Docker 15.11.2026 (Centrifugo v6.10.3, 20 исправлений); сессии 4–21 код и команды не запускались, кода прохождения нет. Методичка [`centrifugo.html`](centrifugo.html): 21 сессия, 138 шагов.** Авторы исходного чата отмечают: код сессий 8B, 12B, 13B, 15 и пример k6 не запускались; имена опций consumers, channel_regex, лимитов соединений, emulated_headers, internal_port — по памяти, сверить через `centrifugo defaultenv`.

## Что по плану

- 21 сессия, ~62 часа, каждая не больше 3,5 ч (кроме финала); после каждой мини-задание на 15–30 минут.
- Сложность по плану — 3 из 5; на сайте карточка стоит как «Средняя–высокая» (решение владельца).
- Версии: Centrifugo ≥ 6.9.0, `centrifuge-js` 5.x, Laravel 13 / PHP 8.4, Redis ≥ 6.2, PostgreSQL; пакет `denis660/laravel-centrifugo` (в сессии 11 — проверка совместимости).
- Нужно знать до старта: HTTP/JSON, JavaScript (`async/await`), основы Laravel, Docker из практики, SQL (транзакции). Standalone: от других лаб жёстко не зависит.
- Вне лабы: PRO-функции Centrifugo, WebSocket-протокол изнутри, Kafka, Kubernetes.

## Что в репозитории

| Файл | Что |
|---|---|
| `centrifugo.html` | методичка (21 сессия) |
| `centrifugo_lab_plan_v4.md` | итоговый план лабы (v4): сессии, шаги, эксперименты |
| `LICENSE`, `NOTICE` | лицензия и авторство |

Обложка лежит не здесь, а в `works/images/centrifugo.png` и в бакете сайта.

## Лицензия и авторство
Код — MIT, тексты — CC BY 4.0, см. `LICENSE` и `NOTICE`.
