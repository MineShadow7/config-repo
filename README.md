# Конфигурации для Spring Cloud и схема работы

Централизованное хранилище конфигураций для нашего микросервисного кластера на базе Spring Cloud.

Здесь лежат `.yml` файлы, которые `config-service` стягивает при старте и раздает остальным микросервисам (`user-service`, `notification-service` и т.д.). Любые изменения в настройках, паролях или хостах делаются тут.

---

## Схема взаимодействия сервисов

Если вкратце, архитектура выглядит так:

1. **Config Service (:8888)** смотрит в этот репозиторий (ветка `master`) и работает как сервер конфигураций.
2. **Discovery Service (:8761)** — Eureka. Все сервисы при старте стучатся и регистрируются, чтобы знать IP-адреса друг друга.
3. **Gateway API (:8080)** — единая точка входа.
4. **Бизнес-логика**:
* Входящий запрос летит в Gateway.
* Gateway роутит его в `user-service` (:8081).
* `user-service` пишет пользователя в свою Postgres и кидает асинхронное событие в **Kafka** (топик `user-events`).
* `notification-service` (:8082) вычитывает событие из Kafka и дергает реальный SMTP-сервер для отправки email.
* *Fallback:* Если Kafka или уведомления лежат, `user-service` не валится с 500-й ошибкой, а отрабатывает через **Resilience4j Circuit Breaker** (возвращает заглушку).



```text
[Клиент] ---> [Gateway API :8080]
                     |
            (HTTP маршрутизация)
                     |
                     v
             [User Service] ------> (Kafka: user-events) ------> [Notification Service]
                     |                                                   |
                 (user_db)                                       (notification_db)

```

---

## Тестирование и верификация

Запуск контейнеров через `docker compose up -d` используя compose из данного репозитория.

### 1. Проверка инфраструктуры

* **Eureka:** `http://localhost:8761`. GATEWAY-API, USER-SERVICE и NOTIFICATION-SERVICE должны быть запущены.
* **Конфиги:** Дергаем ручку `http://localhost:8888/notification-service/default`. В ответе должен быть JSON с настройками.

### 2. Сквозной тест (Создание юзера + Email)

Запрос напрямую в Gateway. Он проксирует запрос на `user-service`:

```bash
curl -X POST http://localhost:8080/api/users/create \
-H "Content-Type: application/json" \
-d '{
    "name": "Name Name",
    "age": "30",
    "email": "email@ya.ru"
}'
```

**Что должно произойти в логах:**

1. В `docker compose logs user-service` — успешное сохранение в БД и отправка ивента.
2. В `docker compose logs notification-service` — консьюмер кафки пишет `Received user notification event...` и пытается отправить письмо. Проверяйте почтовый ящик.


## Важные памятки для разработки

* **Настройка SMTP:** Здесь не должны быть реальные пароли от почты. В `notification-service.yml` пароль должен подтягиваться из переменных окружения (`${SPRING_MAIL_PASSWORD}`).