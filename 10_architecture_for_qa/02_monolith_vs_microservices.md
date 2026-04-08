# Монолит vs Микросервисы

## Обзор

Выбор архитектуры приложения — монолит или микросервисы — напрямую определяет тестовую стратегию QA-инженера.
В монолите всё находится в одном развёртываемом артефакте: один репозиторий, одна база данных, один деплой.
В микросервисной архитектуре приложение разбито на десятки (иногда сотни) независимых сервисов, каждый из
которых отвечает за свою область и деплоится отдельно. Для QA это означает принципиально разные подходы к
тестированию, другие инструменты и совершенно иные типы багов.

---

## Монолитная архитектура

### Структура

```
┌──────────────────────────────────────┐
│           Monolith Application       │
│                                      │
│  ┌──────────┐  ┌──────────────────┐  │
│  │   UI     │  │  Business Logic  │  │
│  │  Layer   │  │     Layer        │  │
│  └──────────┘  └──────────────────┘  │
│  ┌──────────┐  ┌──────────────────┐  │
│  │   API    │  │  Data Access     │  │
│  │  Layer   │  │     Layer        │  │
│  └──────────┘  └──────────────────┘  │
│                                      │
└──────────────┬───────────────────────┘
               │
        ┌──────┴──────┐
        │  Database   │
        │ (единая БД) │
        └─────────────┘
```

### Характеристики монолита

- **Единая кодовая база** — все модули в одном репозитории
- **Единый процесс** — приложение запускается как один процесс (один JAR/WAR)
- **Общая база данных** — все модули работают с одной БД
- **Единый деплой** — любое изменение требует деплоя всего приложения
- **Внутренние вызовы** — модули вызывают друг друга через обычные вызовы методов (in-process)

### Преимущества монолита

1. **Простота разработки** — один проект, одна IDE, лёгкий старт
2. **Простота тестирования** — одно приложение, легко поднять локально
3. **Простота деплоя** — один артефакт, один сервер
4. **Транзакции** — ACID-транзакции в рамках одной БД
5. **Отладка** — stack trace показывает полный путь через все слои

### Недостатки монолита

1. **Масштабируемость** — нельзя масштабировать отдельный модуль
2. **Зависимость команд** — изменения одной команды могут сломать код другой
3. **Технологический lock-in** — один стек для всего приложения
4. **Время сборки** — растёт с размером кодовой базы
5. **Рискованный деплой** — баг в одном модуле ломает всё приложение

---

## Микросервисная архитектура

### Структура

```
                    ┌──────────────┐
                    │  API Gateway │
                    └──────┬───────┘
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │  User    │    │  Order   │    │ Payment  │
    │ Service  │    │ Service  │    │ Service  │
    └────┬─────┘    └────┬─────┘    └────┬─────┘
         │               │               │
    ┌────┴─────┐    ┌────┴─────┐    ┌────┴─────┐
    │ User DB  │    │ Order DB │    │Payment DB│
    └──────────┘    └──────────┘    └──────────┘
```

### Характеристики микросервисов

- **Отдельные репозитории** (или монорепо с разделением)
- **Независимые процессы** — каждый сервис работает в своём контейнере
- **Собственные базы данных** — каждый сервис владеет своими данными (Database per Service)
- **Независимый деплой** — каждый сервис деплоится отдельно
- **Коммуникация** — через HTTP/REST, gRPC или message queue (Kafka, RabbitMQ)

### Преимущества микросервисов

1. **Независимое масштабирование** — нагруженные сервисы масштабируются отдельно
2. **Технологическая свобода** — каждый сервис может использовать свой стек
3. **Независимый деплой** — команды деплоят свои сервисы без координации
4. **Изоляция сбоев** — падение одного сервиса не роняет всё приложение
5. **Маленькие команды** — каждая команда владеет 1-3 сервисами

### Недостатки микросервисов

1. **Сложность системы** — десятки сервисов, каждый со своей конфигурацией
2. **Сетевые проблемы** — latency, timeouts, partial failures
3. **Распределённые транзакции** — нет простого ACID, нужен Saga pattern
4. **Сложность тестирования** — E2E тесты требуют поднятия многих сервисов
5. **Observability** — нужны distributed tracing, centralized logging

---

## Сравнительная таблица

| Аспект | Монолит | Микросервисы |
|--------|---------|--------------|
| Кодовая база | Единая | Распределённая |
| Деплой | Один артефакт | Независимые деплои |
| Масштабирование | Вертикальное | Горизонтальное, гранулярное |
| Коммуникация | In-process вызовы | HTTP/gRPC/Message Queue |
| База данных | Одна общая | По одной на сервис |
| Транзакции | ACID | Eventual consistency, Saga |
| Тестирование | Простое локально | Сложное, нужна инфраструктура |
| Отладка | Stack trace | Distributed tracing |
| Время старта команды | Быстрый | Долгий (инфраструктура) |

---

## Компоненты микросервисной архитектуры

### API Gateway

API Gateway — единая точка входа для всех клиентских запросов. Он маршрутизирует запросы к нужным
сервисам, выполняет аутентификацию, rate limiting, агрегацию ответов.

```
Клиент → API Gateway → User Service
                      → Order Service
                      → Payment Service
```

**Что тестировать в API Gateway:**
- Маршрутизация запросов к правильным сервисам
- Аутентификация и авторизация на уровне gateway
- Rate limiting (превышение лимитов)
- Трансформация запросов/ответов
- Обработка ошибок (когда downstream-сервис недоступен)

### Service Mesh

Service Mesh (Istio, Linkerd) — инфраструктурный слой, который управляет коммуникацией между
сервисами. Он обеспечивает mTLS, circuit breaking, retries, load balancing.

**Что тестировать:**
- mTLS между сервисами (невозможность обойти)
- Circuit breaker — поведение при каскадных сбоях
- Retry policy — корректность повторных попыток
- Traffic routing — canary deploys, A/B testing

### Service Discovery

Сервисы динамически регистрируются и находят друг друга через Service Discovery (Consul, Eureka,
Kubernetes DNS).

**Что тестировать:**
- Регистрация нового инстанса сервиса
- Удаление упавшего инстанса из реестра
- Балансировка нагрузки между инстансами

---

## Влияние архитектуры на тестовую стратегию

### Тестовая пирамида: Монолит

```
        /  E2E Tests  \          ← Мало (10-15%)
       / Integration   \        ← Среднее количество (25-30%)
      /   Unit Tests    \       ← Много (55-65%)
     ──────────────────────
```

В монолите интеграционные тесты проще — всё в одном процессе. Можно использовать `@SpringBootTest`
для поднятия полного контекста и тестирования взаимодействия модулей.

### Тестовая пирамида: Микросервисы (Testing Honeycomb)

```
        /  E2E Tests  \          ← Минимум (5-10%)
       / Integration   \        ← Много (40-50%) — ключевой уровень!
      / Contract Tests  \       ← Новый уровень!
     /   Unit Tests      \      ← Достаточно (30-40%)
    ────────────────────────
```

В микросервисах появляется **contract testing** — проверка, что сервисы корректно взаимодействуют
друг с другом на уровне контрактов (Pact, Spring Cloud Contract).

### Типы тестов в микросервисах

**1. Unit Tests** — тестирование бизнес-логики отдельного сервиса:
```java
// Юнит-тест для OrderService — проверяет логику расчёта без внешних зависимостей
@Test
void shouldCalculateOrderTotal() {
    Order order = new Order();
    order.addItem(new Item("Laptop", 1000.0));
    order.addItem(new Item("Mouse", 50.0));
    assertEquals(1050.0, order.getTotal());
}
```

**2. Integration Tests** — проверка взаимодействия сервиса с его зависимостями (БД, кэш):
```java
// Интеграционный тест — проверяет взаимодействие с реальной БД через Testcontainers
@SpringBootTest
@Testcontainers
class OrderRepositoryTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @Test
    void shouldSaveAndRetrieveOrder() {
        Order saved = repository.save(new Order("ORD-001", 1050.0));
        Order found = repository.findById(saved.getId()).orElseThrow();
        assertEquals("ORD-001", found.getOrderNumber());
    }
}
```

**3. Contract Tests** — проверка контракта между producer и consumer:
```java
// Consumer-side contract test (Pact) — описывает ожидания от User Service
@Pact(consumer = "OrderService", provider = "UserService")
RequestResponsePact getUserPact(PactDslWithProvider builder) {
    return builder
        .given("user with id 1 exists")
        .uponReceiving("запрос на получение пользователя")
        .path("/api/users/1")
        .method("GET")
        .willRespondWith()
        .status(200)
        .body(newJsonBody(body -> {
            body.integerType("id", 1);
            body.stringType("name", "Иван");
        }).build())
        .toPact();
}
```

**4. Component Tests** — тестирование одного сервиса целиком с замоканными зависимостями:
```java
// Компонентный тест — поднимаем сервис, мокаем внешние зависимости через WireMock
@SpringBootTest(webEnvironment = RANDOM_PORT)
class OrderServiceComponentTest {
    @Autowired
    private TestRestTemplate restTemplate;

    // WireMock эмулирует ответ от UserService
    @RegisterExtension
    static WireMockExtension userService = WireMockExtension.newInstance()
        .options(wireMockConfig().port(8081))
        .build();

    @Test
    void shouldCreateOrder() {
        // Настраиваем мок для UserService
        userService.stubFor(get("/api/users/1")
            .willReturn(okJson("{\"id\": 1, \"name\": \"Иван\"}")));

        // Тестируем OrderService
        ResponseEntity<Order> response = restTemplate.postForEntity(
            "/api/orders", new OrderRequest(1L, List.of("item1")), Order.class);

        assertEquals(HttpStatus.CREATED, response.getStatusCode());
    }
}
```

**5. E2E Tests** — тестирование всего пользовательского сценария:
```java
// E2E-тест — проверяет полный сценарий через все сервисы
@Test
void shouldCompleteOrderFlow() {
    // 1. Регистрация пользователя (User Service)
    var user = createUser("Иван", "ivan@example.com");
    // 2. Создание заказа (Order Service)
    var order = createOrder(user.getId(), List.of("item1", "item2"));
    // 3. Оплата (Payment Service)
    var payment = processPayment(order.getId(), "4111111111111111");
    // 4. Проверка статуса заказа
    assertEquals("PAID", getOrderStatus(order.getId()));
}
```

---

## E2E Testing в микросервисах: вызовы

### Проблема 1: Среда для тестирования
Для запуска E2E тестов нужно поднять **все** сервисы. В монолите — один `docker-compose up`.
В микросервисах — десятки контейнеров.

**Решение:** docker-compose с минимальным набором сервисов для конкретного flow:
```yaml
# docker-compose.test.yml — минимальная среда для тестирования order flow
services:
  api-gateway:
    image: api-gateway:latest
    ports: ["8080:8080"]
  user-service:
    image: user-service:latest
  order-service:
    image: order-service:latest
  payment-service:
    image: payment-service:latest
  postgres:
    image: postgres:15
  kafka:
    image: confluentinc/cp-kafka:latest
```

### Проблема 2: Тестовые данные
Каждый сервис имеет свою БД. Создание согласованных тестовых данных — нетривиальная задача.

**Решение:** Использовать API для подготовки данных (не прямые вставки в БД разных сервисов).

### Проблема 3: Flaky tests
Асинхронная коммуникация между сервисами (через Kafka) приводит к race conditions.

**Решение:** Polling/await паттерн вместо жёстких `Thread.sleep`:
```java
// Ожидаем, пока заказ перейдёт в статус PAID (обработка через Kafka может занять время)
await().atMost(30, SECONDS)
       .pollInterval(1, SECONDS)
       .until(() -> getOrderStatus(orderId), equalTo("PAID"));
```

### Проблема 4: Debugging failures
Когда E2E тест падает, непонятно, в каком из 10 сервисов проблема.

**Решение:** Centralized logging (ELK Stack) + distributed tracing (Jaeger, Zipkin) +
correlation ID, который проходит через все сервисы.

---

## Связь с тестированием

| Тип тестирования | Монолит | Микросервисы |
|------------------|---------|--------------|
| Unit | Стандартно (JUnit, Mockito) | Стандартно (JUnit, Mockito) |
| Integration | @SpringBootTest | Testcontainers + WireMock |
| Contract | Не нужны | Обязательны (Pact, SCC) |
| Component | = Integration | Изолированный сервис + моки |
| E2E | Относительно просто | Сложно, дорого, flaky |
| Performance | Один сервис | Нагрузка на отдельные сервисы + всю систему |

---

## Типичные ошибки

1. **Отсутствие contract tests в микросервисах** — сервисы ломают друг другу API без предупреждения
2. **Слишком много E2E тестов** — медленные, flaky, дорогие в поддержке
3. **Тестирование микросервисов как монолита** — попытка поднять всё в одном тесте
4. **Игнорирование partial failure** — не тестируют сценарии, когда один из сервисов недоступен
5. **Жёсткие sleep вместо polling** — ведёт к flaky tests или слишком долгому ожиданию
6. **Отсутствие мониторинга в тестовых средах** — невозможно понять причину падения теста
7. **Прямые вызовы между сервисами в тестах** — вместо API Gateway, тесты знают о внутренней структуре

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. В чём разница между монолитом и микросервисами?
2. Назовите 3 преимущества и 3 недостатка микросервисов.
3. Что такое API Gateway?
4. Как микросервисы общаются друг с другом?

### 🟡 Средний уровень
5. Как отличается тестовая пирамида для монолита и микросервисов?
6. Что такое contract testing и зачем он нужен?
7. Как организовать E2E-тестирование в микросервисной среде?
8. Что такое eventual consistency и как это влияет на тестирование?
9. Как вы будете отлаживать падающий E2E-тест в среде с 15 микросервисами?
10. Что такое circuit breaker и как его тестировать?

### 🔴 Продвинутый уровень
11. Сравните подходы: consumer-driven contract testing vs provider-driven. Когда какой использовать?
12. Как тестировать Saga pattern (распределённые транзакции)?
13. Как организовать тестовые данные в микросервисной архитектуре с Database per Service?
14. Опишите стратегию тестирования при миграции с монолита на микросервисы.
15. Как тестировать canary deployment и blue-green deployment?

---

## Практические задания

### Задание 1: Сравнение стратегий
Для гипотетического интернет-магазина (каталог, корзина, оплата, доставка):
1. Нарисуйте архитектуру в варианте монолита и микросервисов
2. Для каждого варианта составьте тестовую стратегию: какие типы тестов, сколько, какие инструменты

### Задание 2: Contract Test
Напишите consumer-side contract test (используя Pact или Spring Cloud Contract) для взаимодействия
между OrderService и ProductService.

### Задание 3: Тестирование отказоустойчивости
Поднимите 3 сервиса в docker-compose. Напишите тест, который:
1. Выполняет запрос через все 3 сервиса
2. Останавливает один из сервисов (`docker stop`)
3. Проверяет, что система корректно обрабатывает ошибку (возвращает fallback-ответ или понятную ошибку)

### Задание 4: docker-compose для тестовой среды
Создайте `docker-compose.yml`, который поднимает:
- API Gateway (nginx)
- 2 backend-сервиса (любые простые приложения)
- PostgreSQL
- Kafka
Напишите скрипт, который проверяет готовность всех сервисов.

---

## Дополнительные ресурсы

- [Martin Fowler: Microservices](https://martinfowler.com/articles/microservices.html)
- [Sam Newman: Building Microservices (книга)](https://samnewman.io/books/building_microservices_2nd_edition/)
- [Testing Strategies in a Microservice Architecture](https://martinfowler.com/articles/microservice-testing/)
- [Pact: Contract Testing](https://docs.pact.io/)
- [Spring Cloud Contract](https://spring.io/projects/spring-cloud-contract)
- [Microservice Testing Honeycomb](https://engineering.atspotify.com/2018/01/testing-of-microservices/)
