# Стратегии тестирования микросервисов

## Обзор

Микросервисная архитектура кардинально меняет подход к тестированию по сравнению с монолитными приложениями.
Вместо одного развёртываемого артефакта мы имеем дело с десятками (а иногда сотнями) независимых сервисов,
каждый из которых имеет собственную базу данных, API, очередь сообщений и жизненный цикл деплоя.
QA-инженер в микросервисной среде должен понимать, на каком уровне какие дефекты ловить,
как минимизировать стоимость тестирования и при этом обеспечить надёжность всей системы.

В этом разделе рассматриваются ключевые стратегии: пирамида тестирования для микросервисов,
уровни тестирования (unit, component, integration, E2E), контрактное тестирование,
тестирование через API Gateway и service virtualization.

---

## Пирамида тестирования для микросервисов

### Классическая пирамида vs микросервисная

В классической пирамиде (по Майку Кону) основание составляют unit-тесты, середина — интеграционные,
вершина — UI/E2E. В микросервисах эта модель трансформируется:

```
        /  E2E  \            ← Минимум: критические сценарии бизнес-потоков
       /----------\
      / Integration \        ← Взаимодействие между сервисами (API, события)
     /----------------\
    / Contract Testing  \    ← Верификация контрактов между сервисами
   /---------------------\
  /   Component Testing    \ ← Тестирование сервиса в изоляции (с моками зависимостей)
 /---------------------------\
/      Unit Testing            \ ← Бизнес-логика внутри сервиса
---------------------------------
```

### Почему E2E-тестов должно быть мало

- **Хрупкость**: E2E-тесты зависят от доступности всех сервисов одновременно.
- **Скорость**: запуск десятков микросервисов для одного теста — это минуты (иногда десятки минут).
- **Флакающие тесты**: сетевые задержки, race conditions между сервисами, нестабильные окружения.
- **Сложность отладки**: при падении E2E-теста непонятно, какой именно сервис виноват.

### Рекомендуемое соотношение

| Уровень            | Доля тестов | Скорость выполнения | Стоимость поддержки |
|--------------------|-------------|---------------------|---------------------|
| Unit               | ~40-50%     | Миллисекунды        | Низкая              |
| Component          | ~20-30%     | Секунды             | Средняя             |
| Contract           | ~10-15%     | Секунды             | Средняя             |
| Integration        | ~10-15%     | Десятки секунд      | Высокая             |
| E2E                | ~5%         | Минуты              | Очень высокая       |

---

## Unit Testing в микросервисах

### Что тестируем

- Бизнес-логику доменных сервисов (domain services, value objects, entities).
- Маппинг данных (DTO → Entity, Entity → Response).
- Валидационные правила.
- Утилитарные классы.

### Пример: тестирование бизнес-логики расчёта скидки

```java
// Сервис расчёта скидки для заказа
public class DiscountCalculator {

    public BigDecimal calculate(Order order, CustomerType customerType) {
        if (order.getTotal().compareTo(BigDecimal.valueOf(10000)) > 0
                && customerType == CustomerType.VIP) {
            return order.getTotal().multiply(BigDecimal.valueOf(0.15)); // 15% скидка
        }
        if (order.getTotal().compareTo(BigDecimal.valueOf(5000)) > 0) {
            return order.getTotal().multiply(BigDecimal.valueOf(0.05)); // 5% скидка
        }
        return BigDecimal.ZERO; // Без скидки
    }
}
```

```java
@Test
void shouldApplyVipDiscountForLargeOrder() {
    // Подготовка: VIP-клиент с заказом на 15000
    var order = new Order(BigDecimal.valueOf(15000));
    var calculator = new DiscountCalculator();

    // Действие
    var discount = calculator.calculate(order, CustomerType.VIP);

    // Проверка: 15% от 15000 = 2250
    assertThat(discount).isEqualByComparingTo(BigDecimal.valueOf(2250));
}

@Test
void shouldApplyStandardDiscountForMediumOrder() {
    var order = new Order(BigDecimal.valueOf(7000));
    var calculator = new DiscountCalculator();

    var discount = calculator.calculate(order, CustomerType.REGULAR);

    // 5% от 7000 = 350
    assertThat(discount).isEqualByComparingTo(BigDecimal.valueOf(350));
}

@Test
void shouldReturnZeroDiscountForSmallOrder() {
    var order = new Order(BigDecimal.valueOf(3000));
    var calculator = new DiscountCalculator();

    var discount = calculator.calculate(order, CustomerType.VIP);

    assertThat(discount).isEqualByComparingTo(BigDecimal.ZERO);
}
```

### Ключевой принцип

Unit-тесты не поднимают Spring Context, не обращаются к базе данных или внешним сервисам.
Они быстрые, детерминированные и изолированные.

---

## Component Testing (тестирование отдельного сервиса)

### Суть подхода

Component test проверяет один микросервис целиком — от HTTP-эндпоинта до базы данных,
но все внешние зависимости (другие сервисы, очереди, внешние API) заменяются моками или стабами.

### Технологии

- **Spring Boot Test** (`@SpringBootTest`) — поднимает полный контекст приложения.
- **Testcontainers** — реальная БД (PostgreSQL, MongoDB) в Docker-контейнере.
- **WireMock** — мокирование внешних HTTP-зависимостей.
- **Embedded Kafka / Testcontainers Kafka** — мокирование очередей.

### Пример: component test для Order Service

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class OrderServiceComponentTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
            .withDatabaseName("orders_test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    TestRestTemplate restTemplate;

    // WireMock заменяет payment-service
    @RegisterExtension
    static WireMockExtension paymentService = WireMockExtension.newInstance()
            .options(wireMockConfig().dynamicPort())
            .build();

    @Test
    void shouldCreateOrderAndCallPayment() {
        // Настраиваем мок payment-service
        paymentService.stubFor(post("/api/payments")
                .willReturn(okJson("{\"paymentId\": \"pay-123\", \"status\": \"APPROVED\"}")));

        // Отправляем запрос на создание заказа
        var request = new CreateOrderRequest("item-1", 2, BigDecimal.valueOf(5000));
        var response = restTemplate.postForEntity("/api/orders", request, OrderResponse.class);

        // Проверяем, что заказ создан
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody().getStatus()).isEqualTo("CONFIRMED");

        // Проверяем, что payment-service был вызван
        paymentService.verify(postRequestedFor(urlEqualTo("/api/payments")));
    }
}
```

### Что выявляют component-тесты

- Ошибки конфигурации Spring (неправильные бины, отсутствующие зависимости).
- Некорректные SQL-запросы и миграции Flyway/Liquibase.
- Ошибки сериализации/десериализации JSON.
- Неправильная обработка HTTP-ошибок.

---

## Integration Testing (тестирование взаимодействия сервисов)

### Отличие от component testing

Integration test проверяет реальное взаимодействие между двумя или более сервисами.
Здесь нет моков — сервисы общаются по сети (HTTP, gRPC, Kafka).

### Подходы к интеграционному тестированию

| Подход                      | Описание                                        | Плюсы                 | Минусы                    |
|-----------------------------|------------------------------------------------|-----------------------|---------------------------|
| Docker Compose              | Поднимаем все сервисы в Docker                  | Близко к проду        | Медленно, ресурсоёмко     |
| Testcontainers (несколько)  | Каждый сервис в контейнере                      | Программный контроль  | Сложная настройка         |
| Staging-окружение           | Тесты на реальном staging                       | Максимально реалистично| Общее окружение, конфликты|
| Contract testing            | Проверка контрактов без реальной связи          | Быстро, надёжно       | Не ловит runtime-ошибки   |

### Пример: Docker Compose для интеграционных тестов

```yaml
# docker-compose.test.yml
version: '3.8'
services:
  order-service:
    image: order-service:latest
    ports:
      - "8081:8080"
    environment:
      - PAYMENT_SERVICE_URL=http://payment-service:8080
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/orders
    depends_on:
      - postgres
      - payment-service

  payment-service:
    image: payment-service:latest
    ports:
      - "8082:8080"
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/payments
    depends_on:
      - postgres

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: orders
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
```

```java
// Интеграционный тест, выполняемый после docker-compose up
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class OrderPaymentIntegrationTest {

    static final String ORDER_SERVICE = "http://localhost:8081";
    static final RestClient client = RestClient.create();

    @Test
    @Order(1)
    void shouldCreateOrderAndProcessPayment() {
        // Создаём заказ — order-service вызывает payment-service
        var response = client.post()
                .uri(ORDER_SERVICE + "/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .body("""
                    {"itemId": "item-1", "quantity": 2, "amount": 5000}
                    """)
                .retrieve()
                .toEntity(String.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
    }
}
```

---

## E2E Testing

### Когда E2E необходимы

- **Критические бизнес-потоки**: оформление заказа, оплата, регистрация пользователя.
- **Smoke-тесты после деплоя**: проверка, что ключевые функции работают на production-like среде.
- **Регрессия перед релизом**: ограниченный набор сценариев для финальной проверки.

### Инструменты для E2E в микросервисной среде

- **REST Assured / HTTP-клиенты** — для проверки API-потоков без UI.
- **Selenium / Playwright** — если есть фронтенд.
- **Karate DSL** — DSL для API-тестирования со встроенной поддержкой цепочек вызовов.

### Паттерн: тестирование через API Gateway

```
Тест → API Gateway → Service A → Service B → Service C → БД
```

Преимущества тестирования через Gateway:
- Тест видит систему так же, как реальный клиент.
- Проверяется маршрутизация, авторизация, rate limiting.
- Единая точка входа упрощает написание тестов.

```java
// E2E-тест: полный цикл заказа через API Gateway
@Test
void fullOrderLifecycle() {
    // 1. Регистрация пользователя
    var token = given()
            .baseUri(GATEWAY_URL)
            .body(new RegisterRequest("user@test.com", "password123"))
            .post("/api/auth/register")
            .then().statusCode(201)
            .extract().jsonPath().getString("token");

    // 2. Создание заказа (Gateway → Order Service → Inventory Service)
    var orderId = given()
            .baseUri(GATEWAY_URL)
            .header("Authorization", "Bearer " + token)
            .body(new CreateOrderRequest("item-1", 2))
            .post("/api/orders")
            .then().statusCode(201)
            .extract().jsonPath().getString("orderId");

    // 3. Оплата заказа (Gateway → Payment Service)
    given()
            .baseUri(GATEWAY_URL)
            .header("Authorization", "Bearer " + token)
            .body(new PaymentRequest(orderId, "4111111111111111"))
            .post("/api/payments")
            .then().statusCode(200)
            .body("status", equalTo("PAID"));

    // 4. Проверка статуса заказа (должен обновиться асинхронно)
    await().atMost(Duration.ofSeconds(10)).untilAsserted(() -> {
        given()
                .baseUri(GATEWAY_URL)
                .header("Authorization", "Bearer " + token)
                .get("/api/orders/" + orderId)
                .then().statusCode(200)
                .body("status", equalTo("COMPLETED"));
    });
}
```

---

## Контрактное тестирование (обзор роли в стратегии)

### Зачем нужно контрактное тестирование

В микросервисной архитектуре основная причина поломок — **несогласованные изменения API**.
Один сервис меняет формат ответа, и все зависимые сервисы ломаются.

Контрактное тестирование решает эту проблему:
- **Consumer** (потребитель) описывает ожидания от API провайдера.
- **Provider** (провайдер) верифицирует, что его API соответствует ожиданиям.
- Тесты выполняются **независимо** — не нужно поднимать оба сервиса одновременно.

### Consumer-Driven Contract Testing (CDC)

```
1. Consumer пишет тест → генерирует Pact-файл (контракт)
2. Pact-файл публикуется в Pact Broker
3. Provider запускает верификацию контракта
4. Если верификация успешна → деплой разрешён (can-i-deploy)
```

Подробная практика описана в файле `04_contract_testing_practice.md`.

---

## Service Virtualization

### Что это такое

Service virtualization — создание виртуальных копий зависимых сервисов,
которые имитируют их поведение (ответы, задержки, ошибки) без необходимости
запускать реальные экземпляры.

### Отличие от моков

| Характеристика      | Mock (WireMock)                   | Service Virtualization              |
|----------------------|-----------------------------------|-------------------------------------|
| Область применения   | Один тест / тестовый класс       | Общее тестовое окружение            |
| Сложность поведения  | Простые стабы                    | Stateful-имитация, сценарии         |
| Данные               | Захардкожены в тесте             | Записаны с реального сервиса        |
| Инструменты          | WireMock, MockServer             | Hoverfly, Mountebank, WireMock Standalone |

### Пример: Hoverfly для service virtualization

```java
@ExtendWith(HoverflyExtension.class)
class OrderServiceWithHoverflyTest {

    // Hoverfly перехватывает вызовы к payment-service
    static {
        var simulation = SimulationSource.dsl(
                service("payment-service.internal")
                        .post("/api/payments")
                        .body(equalsToJson("{\"orderId\": \"order-1\", \"amount\": 5000}"))
                        .willReturn(success()
                                .body("{\"paymentId\": \"pay-1\", \"status\": \"APPROVED\"}")
                                .header("Content-Type", "application/json"))
                        // Имитация задержки — важно для тестирования timeout'ов
                        .post("/api/payments")
                        .body(equalsToJson("{\"orderId\": \"order-slow\", \"amount\": 99999}"))
                        .willReturn(success()
                                .body("{\"paymentId\": \"pay-2\", \"status\": \"APPROVED\"}")
                                .withDelay(5, TimeUnit.SECONDS))
        );
    }
}
```

### Запись и воспроизведение трафика

Service virtualization позволяет записать реальные ответы сервиса (capture mode),
а затем воспроизводить их в тестах (simulate mode). Это особенно полезно, когда:
- Внешний сервис платный (например, платёжный шлюз).
- Сервис недоступен в CI-окружении.
- Нужно тестировать обработку специфических ошибок.

---

## Тестирование отдельного сервиса vs цепочки сервисов

### Матрица решений: что тестировать на каком уровне

| Что проверяем                          | Уровень тестирования       | Инструмент                |
|----------------------------------------|----------------------------|---------------------------|
| Бизнес-логика (валидация, расчёты)     | Unit                       | JUnit 5 + Mockito         |
| REST API одного сервиса                | Component                  | @SpringBootTest + WireMock|
| Запросы к БД, миграции                 | Component                  | @DataJpaTest + Testcontainers |
| Формат сообщений между сервисами       | Contract                   | Pact / Spring Cloud Contract |
| Синхронный вызов между сервисами       | Integration                | Docker Compose / Staging  |
| Асинхронное взаимодействие через Kafka | Integration                | Testcontainers Kafka      |
| Полный бизнес-сценарий                 | E2E                        | REST Assured / Karate     |
| Нефункциональные требования            | Performance / Chaos        | Gatling, Chaos Monkey     |

### Принцип «сдвига влево» (shift left)

Чем раньше в пирамиде мы ловим баг, тем дешевле его исправление.
QA-инженер должен стремиться к тому, чтобы максимум дефектов выявлялось
на уровне unit и component тестов.

---

## Связь с тестированием

Стратегия тестирования микросервисов напрямую определяет:
- **Скорость CI/CD-пайплайна**: слишком много E2E-тестов замедляют деплой.
- **Стабильность тестов**: правильная стратегия минимизирует flaky-тесты.
- **Покрытие**: каждый уровень тестирования ловит свой класс дефектов.
- **Стоимость**: поддержка тестовой инфраструктуры в микросервисах — серьёзная статья расходов.

QA-инженер в микросервисной команде участвует в:
- Определении стратегии тестирования для каждого сервиса.
- Написании component и contract тестов.
- Настройке тестовой инфраструктуры (Testcontainers, WireMock, Docker Compose).
- Разработке E2E-тестов для критических бизнес-потоков.
- Мониторинге quality gates в CI/CD-пайплайне.

---

## Типичные ошибки

1. **Перебор с E2E-тестами** — попытка покрыть все сценарии E2E-тестами.
   Результат: медленный CI, нестабильные тесты, высокая стоимость поддержки.

2. **Игнорирование контрактного тестирования** — надежда на то, что интеграционные тесты
   поймают все проблемы с API. Контрактные тесты быстрее, надёжнее и дают раннюю обратную связь.

3. **Использование общего тестового окружения** — несколько команд тестируют на одном staging.
   Результат: конфликты данных, непредсказуемое состояние сервисов, flaky-тесты.

4. **Отсутствие изоляции тестовых данных** — тесты зависят от данных, созданных другими тестами.
   Решение: каждый тест создаёт и очищает свои данные.

5. **Мокирование всего подряд** — если замокировано слишком много, тест не проверяет реальное поведение.
   Мокируйте только внешние зависимости, а не внутренние компоненты сервиса.

6. **Отсутствие тестирования негативных сценариев** — тестируем только happy path.
   В микросервисах критически важно тестировать: таймауты, недоступность сервиса,
   некорректные ответы, retry-логику, circuit breaker.

7. **Тестирование внутренней реализации вместо поведения** — тесты ломаются при рефакторинге,
   хотя поведение не изменилось.

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Что такое пирамида тестирования? Как она выглядит для микросервисов?
2. В чём разница между unit-тестом и component-тестом?
3. Почему E2E-тестов должно быть меньше, чем unit-тестов?
4. Что такое WireMock и зачем он нужен?
5. Какие уровни тестирования вы знаете?

### 🟡 Средний уровень
6. Как организовать тестирование сервиса, зависящего от 5 других сервисов?
7. Что такое контрактное тестирование и зачем оно нужно?
8. Как тестировать асинхронное взаимодействие между микросервисами через Kafka?
9. Чем service virtualization отличается от мокирования?
10. Как тестировать микросервис, если зависимый сервис ещё не разработан?
11. Как обеспечить изоляцию тестовых данных в микросервисной среде?

### 🔴 Продвинутый уровень
12. Как построить стратегию тестирования для системы из 50+ микросервисов?
13. Как минимизировать flaky-тесты в микросервисной среде?
14. Объясните концепцию «can-i-deploy» в контрактном тестировании.
15. Как тестировать saga-паттерн (распределённые транзакции)?
16. Как организовать тестирование при canary deployment?
17. Какие метрики качества тестирования вы используете для микросервисов?

---

## Практические задания

### Задание 1: Определение стратегии тестирования
Дано: e-commerce система из 6 сервисов (User, Catalog, Order, Payment, Notification, Shipping).
Задача: определите для каждого сервиса набор тестов по уровням пирамиды.
Укажите, какие взаимодействия покрывать контрактными тестами, а какие — интеграционными.

### Задание 2: Component test с WireMock
Напишите component test для Order Service, который:
- Поднимает сервис с реальной PostgreSQL (Testcontainers).
- Мокирует Payment Service через WireMock.
- Проверяет создание заказа (happy path).
- Проверяет поведение при недоступности Payment Service (таймаут, 500-ошибка).

### Задание 3: Матрица тестирования
Составьте матрицу тестирования для взаимодействия Order Service и Payment Service:
- Перечислите все возможные сценарии (успех, ошибки, таймауты, невалидные данные).
- Для каждого сценария укажите уровень тестирования и используемый инструмент.

### Задание 4: Анализ flaky-тестов
Дано: E2E-тест, который проходит в 70% запусков. Тест проверяет цепочку:
`UI → Gateway → Order → Payment → Notification (email)`.
Задача: предложите план стабилизации. Какие части теста можно перевести на более низкий уровень?

---

## Дополнительные ресурсы

- **Книга**: Sam Newman — "Building Microservices" (глава о тестировании)
- **Книга**: Toby Clemson — "Testing Strategies in a Microservice Architecture" (статья на martinfowler.com)
- **Статья**: Martin Fowler — "The Practical Test Pyramid"
- **Инструмент**: [Testcontainers](https://testcontainers.com/) — официальная документация
- **Инструмент**: [WireMock](https://wiremock.org/) — мокирование HTTP-зависимостей
- **Инструмент**: [Pact](https://docs.pact.io/) — контрактное тестирование
- **Инструмент**: [Hoverfly](https://hoverfly.io/) — service virtualization
- **Видео**: Microservice Testing (Sam Newman, GOTO Conference)
