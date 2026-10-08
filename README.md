# Centrifugo Lab — realtime с нуля

Лаба про realtime на [Centrifugo](https://centrifugal.dev): WebSocket-соединения, каналы и namespaces, `centrifuge-js`, публикация из Laravel, JWT и subscription tokens, proxy, presence и history, recovery и reconnect, Outbox, тесты и безопасность, production (nginx, Redis engine) и нагрузка. Сквозной проект — **Realtime Workspace** (Laravel + Centrifugo + Vue; Centrifugo в Docker).

**Статус: ⚪ заготовка (репозиторий создан 08.10.2026). Методички нет, кода нет, ничего не проверялось запуском: есть только план [`centrifugo_lab_plan_v4.md`](centrifugo_lab_plan_v4.md).** План написан, но не вычитан; эксперименты E1–E8 из его приложения C ещё не прогонялись.

## Что по плану

- 21 сессия, ~62 часа, каждая не больше 3,5 ч (кроме финала); после каждой мини-задание на 15–30 минут.
- Сложность по плану — 3 из 5; на сайте карточка стоит как «Средняя–высокая» (решение владельца).
- Версии: Centrifugo ≥ 6.9.0, `centrifuge-js` 5.x, Laravel 13 / PHP 8.4, Redis ≥ 6.2, PostgreSQL; пакет `denis660/laravel-centrifugo` (в сессии 11 — проверка совместимости).
- Нужно знать до старта: HTTP/JSON, JavaScript (`async/await`), основы Laravel, Docker из практики, SQL (транзакции). Standalone: от других лаб жёстко не зависит.
- Вне лабы: PRO-функции Centrifugo, WebSocket-протокол изнутри, Kafka, Kubernetes.

## Что в репозитории

| Файл | Что |
|---|---|
| `centrifugo_lab_plan_v4.md` | итоговый план лабы (v4): сессии, шаги, эксперименты |
| `LICENSE`, `NOTICE` | лицензия и авторство |

Обложка лежит не здесь, а в `works/images/centrifugo.png` и в бакете сайта.

## Лицензия и авторство
Код — MIT, тексты — CC BY 4.0, см. `LICENSE` и `NOTICE`.
