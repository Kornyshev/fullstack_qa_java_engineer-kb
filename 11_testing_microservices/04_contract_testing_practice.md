# Практика контрактного тестирования

## Обзор

Контрактное тестирование — подход, при котором взаимодействие между сервисами описывается
в виде формального контракта, и каждая сторона проверяется на соответствие этому контракту
**независимо**. Это позволяет обнаружить проблемы интеграции без необходимости запускать
оба сервиса одновременно.

В микросервисной архитектуре контрактное тестирование решает ключевую проблему:
**как убедиться, что изменение API одного сервиса не сломает другие сервисы?**
Интеграционные тесты требуют запуска всех зависимых сервисов и дорого стоят.
Контрактные тесты запускаются быстро, локально и дают ранний фидбек.

В этом разделе пошагово разбирается практика контрактного тестирования с использованием
**Pact** (consumer-driven) и **Spring Cloud Contract** (provider-driven) на реальном примере
взаимодействия `order-service` и `payment-service`.

---

## Реальный пример: order-service → payment-service

### Архитектура взаимодействия

```
┌─────────────────┐        POST /api/payments         ┌──────────────────┐
│  order-service   │  ──────────────────────────────►  │  payment-service  │
│  (Consumer)      │                                    │  (Provider)       │
│                  │  ◄──────────────────────────────  │                   │
│                  │     {paymentId, status, amount}    │                   │
└─────────────────┘                                    └──────────────────┘
```

### API-контракт (ожидания)

**Запрос** (order-service отправляет):
```json
{
  "orderId": "order-123",
  "customerId": "customer-456",
  "amount": 5000.00,
  "currency": "RUB"
}
```

**Ответ** (payment-service возвращает):
```json
{
  "paymentId": "pay-789",
  "orderId": "order-123",
  "status": "APPROVED",
  "amount": 5000.00
}
```

---

## Часть 1: Pact (Consumer-Driven Contract Testing)

### Шаг 1: Consumer-тест (order-service)

Consumer описывает свои ожидания от Provider'а. Этот тест генерирует Pact-файл (контракт).

#### Зависимости (order-service/pom.xml)

```xml
<dependency>
    <groupId>au.com.dius.pact.consumer</groupId>
    <artifactId>junit5</artifactId>
    <version>4.6.5</version>
    <scope>test</scope>
</dependency>
```

#### Consumer-тест

```java
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "payment-service", port = "8888")
class PaymentClientPactTest {

    // Клиент, который делает HTTP-вызовы к payment-service
    private final PaymentClient paymentClient = new PaymentClient("http://localhost:8888");

    // Шаг 1: Описываем ожидания от Provider'а
    @Pact(consumer = "order-service", provider = "payment-service")
    V4Pact createPaymentPact(PactDslWithProvider builder) {
        return builder
                .given("платёжная система доступна") // Состояние провайдера
                .uponReceiving("запрос на создание платежа")
                    .method("POST")
                    .path("/api/payments")
                    .headers(Map.of(
                            "Content-Type", "application/json",
                            "X-Request-Id", Matchers.uuid() // Любой UUID
                    ))
                    .body(new PactDslJsonBody()
                            .stringType("orderId", "order-123")
                            .stringType("customerId", "customer-456")
                            .decimalType("amount", 5000.00)
                            .stringValue("currency", "RUB")
                    )
                .willRespondWith()
                    .status(200)
                    .headers(Map.of("Content-Type", "application/json"))
                    .body(new PactDslJsonBody()
                            .stringType("paymentId")       // Любая строка
                            .stringValue("orderId", "order-123") // Точное значение
                            .stringMatcher("status", "APPROVED|PENDING", "APPROVED")
                            .decimalType("amount", 5000.00)
                    )
                .toPact(V4Pact.class);
    }

    // Шаг 2: Consumer-тест с моком Provider'а
    @Test
    @PactTestFor(pactMethod = "createPaymentPact")
    void shouldCreatePayment() {
        // Pact запускает мок-сервер, имитирующий payment-service
        var request = new PaymentRequest("order-123", "customer-456",
                BigDecimal.valueOf(5000), "RUB");

        var response = paymentClient.createPayment(request);

        // Проверяем, что наш клиент корректно обрабатывает ответ
        assertThat(response.getPaymentId()).isNotBlank();
        assertThat(response.getOrderId()).isEqualTo("order-123");
        assertThat(response.getStatus()).isIn("APPROVED", "PENDING");
        assertThat(response.getAmount()).isEqualByComparingTo(BigDecimal.valueOf(5000));
    }

    // Дополнительный контракт: ошибка при невалидных данных
    @Pact(consumer = "order-service", provider = "payment-service")
    V4Pact paymentRejectedPact(PactDslWithProvider builder) {
        return builder
                .given("платёжная система доступна")
                .uponReceiving("запрос на создание платежа с нулевой суммой")
                    .method("POST")
                    .path("/api/payments")
                    .body(new PactDslJsonBody()
                            .stringType("orderId", "order-bad")
                            .stringType("customerId", "customer-1")
                            .decimalType("amount", 0.00)
                            .stringValue("currency", "RUB")
                    )
                .willRespondWith()
                    .status(400)
                    .body(new PactDslJsonBody()
                            .stringType("error", "Сумма должна быть положительной")
                    )
                .toPact(V4Pact.class);
    }

    @Test
    @PactTestFor(pactMethod = "paymentRejectedPact")
    void shouldHandlePaymentRejection() {
        var request = new PaymentRequest("order-bad", "customer-1",
                BigDecimal.ZERO, "RUB");

        assertThatThrownBy(() -> paymentClient.createPayment(request))
                .isInstanceOf(PaymentRejectedException.class)
                .hasMessageContaining("Сумма должна быть положительной");
    }
}
```

### Шаг 2: Генерация Pact-файла

После выполнения теста Pact генерирует JSON-файл контракта:

```
target/pacts/order-service-payment-service.json
```

```json
{
  "consumer": {"name": "order-service"},
  "provider": {"name": "payment-service"},
  "interactions": [
    {
      "description": "запрос на создание платежа",
      "providerState": "платёжная система доступна",
      "request": {
        "method": "POST",
        "path": "/api/payments",
        "headers": {"Content-Type": "application/json"},
        "body": {
          "orderId": "order-123",
          "customerId": "customer-456",
          "amount": 5000.00,
          "currency": "RUB"
        },
        "matchingRules": {
          "body": {
            "$.orderId": {"matchers": [{"match": "type"}]},
            "$.customerId": {"matchers": [{"match": "type"}]},
            "$.amount": {"matchers": [{"match": "decimal"}]}
          }
        }
      },
      "response": {
        "status": 200,
        "headers": {"Content-Type": "application/json"},
        "body": {
          "paymentId": "string",
          "orderId": "order-123",
          "status": "APPROVED",
          "amount": 5000.00
        }
      }
    }
  ]
}
```

### Шаг 3: Provider-верификация (payment-service)

Provider запускает верификацию: Pact поднимает реальный payment-service и воспроизводит
запросы из контракта, проверяя, что ответы соответствуют ожиданиям consumer'а.

#### Зависимости (payment-service/pom.xml)

```xml
<dependency>
    <groupId>au.com.dius.pact.provider</groupId>
    <artifactId>junit5spring</artifactId>
    <version>4.6.5</version>
    <scope>test</scope>
</dependency>
```

#### Provider-тест

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Provider("payment-service")
@PactBroker(url = "${pact.broker.url}") // Или @PactFolder("pacts") для локальной разработки
class PaymentProviderPactVerificationTest {

    @LocalServerPort
    int port;

    @Autowired
    PaymentRepository paymentRepository;

    @BeforeEach
    void setUp(PactVerificationContext context) {
        context.setTarget(new HttpTestTarget("localhost", port));
    }

    // Настраиваем состояние провайдера (provider state)
    @State("платёжная система доступна")
    void paymentSystemAvailable() {
        // Подготавливаем данные или конфигурацию,
        // чтобы payment-service мог обработать запрос
        // Например, убеждаемся, что БД доступна и содержит нужные данные
    }

    @State("заказ order-123 существует")
    void orderExists() {
        // Создаём тестовые данные для этого состояния
        paymentRepository.save(new PaymentEntity("order-123", BigDecimal.valueOf(5000)));
    }

    // Pact автоматически проверит все interactions из контракта
    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void pactVerificationTestTemplate(PactVerificationContext context) {
        context.verifyInteraction();
    }
}
```

### Шаг 4: Pact Broker

Pact Broker — центральное хранилище контрактов и результатов верификации.

#### Запуск Pact Broker (Docker Compose)

```yaml
version: '3.8'
services:
  pact-broker:
    image: pactfoundation/pact-broker:latest
    ports:
      - "9292:9292"
    environment:
      PACT_BROKER_DATABASE_URL: sqlite:///tmp/pact_broker.sqlite
      PACT_BROKER_BASIC_AUTH_USERNAME: pact
      PACT_BROKER_BASIC_AUTH_PASSWORD: pact

  # Или с PostgreSQL для production
  # pact-broker-db:
  #   image: postgres:15
  #   environment:
  #     POSTGRES_DB: pact_broker
```

#### Публикация контракта в Pact Broker

```xml
<!-- pom.xml: плагин для публикации -->
<plugin>
    <groupId>au.com.dius.pact.provider</groupId>
    <artifactId>maven</artifactId>
    <version>4.6.5</version>
    <configuration>
        <pactBrokerUrl>http://localhost:9292</pactBrokerUrl>
        <pactBrokerUsername>pact</pactBrokerUsername>
        <pactBrokerPassword>pact</pactBrokerPassword>
        <tags>
            <tag>${git.branch}</tag>
        </tags>
    </configuration>
</plugin>
```

```bash
# Публикация контракта после запуска consumer-тестов
mvn pact:publish
```

### Шаг 5: CI/CD интеграция

#### can-i-deploy — безопасный деплой

Перед деплоем проверяем, что контракты верифицированы и совместимы:

```bash
# Проверяем, можно ли деплоить order-service
pact-broker can-i-deploy \
  --pacticipant order-service \
  --version $(git rev-parse --short HEAD) \
  --to-environment production
```

#### Pipeline для Consumer (order-service)

```yaml
# .github/workflows/order-service.yml
name: Order Service CI
on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Запуск unit и contract тестов
        run: mvn test

      - name: Публикация контрактов в Pact Broker
        run: mvn pact:publish -Dpact.broker.url=${{ secrets.PACT_BROKER_URL }}
        env:
          GIT_BRANCH: ${{ github.ref_name }}

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Проверка can-i-deploy
        run: |
          pact-broker can-i-deploy \
            --pacticipant order-service \
            --version ${{ github.sha }} \
            --to-environment production

      - name: Деплой
        if: success()
        run: ./deploy.sh
```

#### Pipeline для Provider (payment-service)

```yaml
# .github/workflows/payment-service.yml
name: Payment Service CI
on:
  push:
  # Webhook от Pact Broker: запускается, когда consumer публикует новый контракт
  repository_dispatch:
    types: [pact-changed]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Запуск unit-тестов
        run: mvn test -Dgroups=unit

      - name: Верификация контрактов от всех consumers
        run: mvn test -Dgroups=pact -Dpact.broker.url=${{ secrets.PACT_BROKER_URL }}

      - name: Публикация результатов верификации
        run: |
          pact-broker create-version-tag \
            --pacticipant payment-service \
            --version ${{ github.sha }} \
            --tag ${{ github.ref_name }}
```

---

## Часть 2: Spring Cloud Contract (альтернатива)

### Отличие от Pact

| Характеристика           | Pact                          | Spring Cloud Contract         |
|--------------------------|-------------------------------|-------------------------------|
| Подход                   | Consumer-driven               | Provider-driven (обычно)      |
| Кто пишет контракт       | Consumer                      | Provider                      |
| Формат контракта          | JSON (Pact-файл)             | Groovy DSL / YAML             |
| Генерация тестов          | Нет                          | Автоматическая (для provider) |
| Stub-сервер               | Pact Mock Server             | WireMock (Spring Cloud Contract Stub Runner) |
| Экосистема                | Мультиязычная                | Spring-ориентированная        |
| Pact Broker               | Да                           | Nexus/Artifactory (stubs JAR) |

### Шаг 1: Контракт на стороне Provider (Groovy DSL)

```groovy
// payment-service/src/test/resources/contracts/createPayment.groovy
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "Создание платежа для заказа"

    request {
        method POST()
        url "/api/payments"
        headers {
            contentType applicationJson()
        }
        body([
                orderId   : $(consumer(regex('[a-z]+-\\d+')), producer('order-123')),
                customerId: $(consumer(regex('[a-z]+-\\d+')), producer('customer-456')),
                amount    : $(consumer(regex('\\d+\\.?\\d*')), producer(5000.00)),
                currency  : "RUB"
        ])
    }

    response {
        status OK()
        headers {
            contentType applicationJson()
        }
        body([
                paymentId: $(producer(regex('[a-z]+-\\d+')), consumer('pay-789')),
                orderId  : fromRequest().body('$.orderId'),
                status   : "APPROVED",
                amount   : fromRequest().body('$.amount')
        ])
    }
}
```

### Контракт в YAML-формате (альтернатива)

```yaml
# payment-service/src/test/resources/contracts/createPayment.yml
description: Создание платежа для заказа
request:
  method: POST
  url: /api/payments
  headers:
    Content-Type: application/json
  body:
    orderId: "order-123"
    customerId: "customer-456"
    amount: 5000.00
    currency: "RUB"
  matchers:
    body:
      - path: $.orderId
        type: by_regex
        value: "[a-z]+-\\d+"
      - path: $.amount
        type: by_regex
        value: "\\d+\\.?\\d*"
response:
  status: 200
  headers:
    Content-Type: application/json
  body:
    paymentId: "pay-789"
    orderId: "order-123"
    status: "APPROVED"
    amount: 5000.00
```

### Шаг 2: Автогенерация Provider-теста

Spring Cloud Contract автоматически генерирует тест по контракту.
Нужна только базовая конфигурация:

```java
// Базовый класс для автогенерированных тестов
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)
@AutoConfigureMockMvc
public abstract class BaseContractTest {

    @Autowired
    MockMvc mockMvc;

    @MockBean
    PaymentService paymentService;

    @BeforeEach
    void setUp() {
        // Настраиваем моки для контрактных тестов
        when(paymentService.processPayment(any()))
                .thenReturn(new PaymentResponse("pay-789", "order-123", "APPROVED",
                        BigDecimal.valueOf(5000)));
    }

    // Конфигурация в pom.xml указывает на этот базовый класс
}
```

```xml
<!-- payment-service/pom.xml -->
<plugin>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-contract-maven-plugin</artifactId>
    <version>4.1.0</version>
    <extensions>true</extensions>
    <configuration>
        <baseClassForTests>
            com.example.payment.BaseContractTest
        </baseClassForTests>
    </configuration>
</plugin>
```

После `mvn generate-test-sources` генерируется тест:

```java
// Автогенерированный тест (не нужно писать вручную)
public class ContractVerifierTest extends BaseContractTest {

    @Test
    public void validate_createPayment() throws Exception {
        mockMvc.perform(post("/api/payments")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"orderId\":\"order-123\",\"customerId\":\"customer-456\"," +
                                "\"amount\":5000.00,\"currency\":\"RUB\"}"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.paymentId").value("pay-789"))
                .andExpect(jsonPath("$.status").value("APPROVED"));
    }
}
```

### Шаг 3: Consumer-тест с Stub Runner

На стороне consumer'а используется `StubRunner`, который скачивает stub и запускает WireMock:

```java
@SpringBootTest
@AutoConfigureStubRunner(
        ids = "com.example:payment-service:+:stubs:8888",
        stubsMode = StubRunnerProperties.StubsMode.LOCAL // Или REMOTE для Nexus
)
class OrderServiceContractTest {

    @Autowired
    PaymentClient paymentClient;

    @Test
    void shouldCreatePaymentAccordingToContract() {
        // WireMock на порту 8888 автоматически настроен по контракту
        var request = new PaymentRequest("order-123", "customer-456",
                BigDecimal.valueOf(5000), "RUB");

        var response = paymentClient.createPayment(request);

        assertThat(response.getPaymentId()).isEqualTo("pay-789");
        assertThat(response.getStatus()).isEqualTo("APPROVED");
    }
}
```

---

## Сравнение подходов: когда что выбрать

### Pact — лучше подходит, когда:

- Микросервисы написаны на разных языках (Java, Python, Node.js).
- Контракт определяется потребностями consumer'а.
- Нужен Pact Broker для централизованного управления.
- Команды работают независимо (разные репозитории, разные CI).

### Spring Cloud Contract — лучше подходит, когда:

- Все сервисы на Spring Boot.
- Контракт определяет provider (owner API).
- Stubs публикуются в Maven-репозиторий (Nexus/Artifactory).
- Нужна автогенерация тестов.

### Общая рекомендация

Для большинства Java-проектов **оба подхода** эффективны.
Выбор часто зависит от предпочтений команды и существующей инфраструктуры.

---

## Продвинутые сценарии

### Тестирование нескольких consumers

Один provider может иметь контракты с несколькими consumers:

```
order-service   ─── контракт ──►  payment-service
shipping-service ─── контракт ──►  payment-service
refund-service  ─── контракт ──►  payment-service
```

Provider верифицирует все контракты одновременно. Если изменение в API ломает хотя бы
один контракт — деплой блокируется.

### Версионирование контрактов

```bash
# Публикация контракта с тегом ветки
pact-broker create-version-tag \
  --pacticipant order-service \
  --version abc123 \
  --tag feature/new-payment

# Provider верифицирует контракты с определённого тега
mvn test -Dpact.provider.tag=main
```

### Тестирование асинхронных взаимодействий (Kafka) через Pact

```java
@Pact(consumer = "order-service", provider = "payment-service")
MessagePact paymentApprovedEvent(MessagePactBuilder builder) {
    return builder
            .given("платёж одобрен для заказа order-123")
            .expectsToReceive("событие PAYMENT_APPROVED")
            .withContent(new PactDslJsonBody()
                    .stringType("eventId")
                    .stringValue("orderId", "order-123")
                    .stringValue("status", "PAYMENT_APPROVED")
                    .decimalType("amount", 5000.00)
                    .stringType("timestamp")
            )
            .toPact();
}

@Test
@PactTestFor(pactMethod = "paymentApprovedEvent", providerType = ProviderType.ASYNCH)
void shouldProcessPaymentApprovedEvent(List<Message> messages) {
    // Pact предоставляет сообщение, соответствующее контракту
    var messageBody = messages.get(0).contentsAsString();
    var event = objectMapper.readValue(messageBody, PaymentEvent.class);

    // Обрабатываем событие
    paymentEventHandler.handle(event);

    // Проверяем результат обработки
    var order = orderRepository.findById("order-123").orElseThrow();
    assertThat(order.getStatus()).isEqualTo(OrderStatus.CONFIRMED);
}
```

---

## Связь с тестированием

Контрактное тестирование является **критическим уровнем** в пирамиде тестирования микросервисов:

- **Заменяет часть интеграционных тестов** — быстрее, надёжнее, не требует запуска всех сервисов.
- **Ранний фидбек** — ломающие изменения обнаруживаются до деплоя.
- **Документация API** — контракт служит живой документацией взаимодействия.
- **Безопасный деплой** — `can-i-deploy` гарантирует, что деплой не сломает зависимые сервисы.

Для QA-инженера контрактное тестирование:
- Снижает количество flaky интеграционных тестов.
- Позволяет тестировать взаимодействие без ожидания готовности зависимого сервиса.
- Обеспечивает автоматическую проверку обратной совместимости API.

---

## Типичные ошибки

1. **Слишком строгие контракты** — проверяют точные значения вместо типов и паттернов.
   Результат: контракт ломается при любом изменении тестовых данных.
   Используйте `stringType()`, `decimalType()`, `regex()` вместо точных значений.

2. **Не настроены Provider States** — Provider-тест падает, потому что нет тестовых данных.
   Каждый `given()` в контракте должен иметь соответствующий `@State` в provider-тесте.

3. **Контракты не публикуются в CI** — контракт существует только локально у разработчика.
   Настройте автоматическую публикацию в Pact Broker / Nexus.

4. **Игнорирование `can-i-deploy`** — деплой без проверки совместимости.
   Обязательно добавьте `can-i-deploy` как quality gate в CI/CD pipeline.

5. **Контракт тестирует реализацию, а не поведение** — проверяют внутренние поля,
   которые consumer не использует. Контракт должен содержать только то, что consumer реально читает.

6. **Нет контрактов для негативных сценариев** — тестируют только happy path.
   Добавьте контракты для 400 (валидация), 404 (не найдено), 500 (ошибка сервера).

7. **Consumer и Provider используют разные версии библиотеки** — несовместимость
   форматов Pact-файлов. Согласуйте версии между командами.

8. **Не тестируют эволюцию контракта** — добавление нового поля в ответ
   должно быть backward-compatible. Старые consumers не должны ломаться.

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Что такое контрактное тестирование и зачем оно нужно?
2. Кто такие Consumer и Provider в контрактном тестировании?
3. Что такое Pact-файл?
4. Чем контрактное тестирование отличается от интеграционного?
5. Что такое consumer-driven contract testing?

### 🟡 Средний уровень
6. Как написать consumer-тест с помощью Pact?
7. Что такое Provider State и зачем он нужен?
8. Как устроена верификация контракта на стороне provider'а?
9. Что такое Pact Broker и какова его роль?
10. Чем Pact отличается от Spring Cloud Contract?
11. Как контрактное тестирование вписывается в CI/CD pipeline?
12. Как тестировать асинхронные взаимодействия (Kafka) через контракты?

### 🔴 Продвинутый уровень
13. Что такое `can-i-deploy` и как это работает?
14. Как организовать контрактное тестирование для 20+ микросервисов?
15. Как обеспечить backward-совместимость при эволюции API?
16. Как тестировать контракты при использовании API Gateway?
17. Как совмещать контрактное тестирование с OpenAPI/Swagger?
18. В каких случаях контрактное тестирование не заменяет интеграционные тесты?

---

## Практические задания

### Задание 1: Consumer-тест с Pact
Напишите consumer-тест для `order-service`, который взаимодействует с `inventory-service`:
- `GET /api/inventory/{itemId}` — проверка наличия товара.
- Ответ: `{ "itemId": "...", "available": true, "quantity": 50 }`.
- Добавьте контракт для случая, когда товар не найден (404).

### Задание 2: Provider-верификация
На стороне `inventory-service` напишите provider-тест:
- Настройте `@State` для двух состояний: "товар item-1 есть на складе" и "товар не найден".
- Верифицируйте контракт из задания 1.

### Задание 3: Spring Cloud Contract
Перепишите контракт из задания 1 с использованием Spring Cloud Contract:
- Напишите контракт в Groovy DSL.
- Настройте автогенерацию тестов.
- Настройте StubRunner на стороне consumer'а.

### Задание 4: CI/CD pipeline
Спроектируйте CI/CD pipeline для двух сервисов (`order-service` и `payment-service`):
- Consumer-тесты запускаются при каждом push.
- Контракт публикуется в Pact Broker.
- Provider-верификация запускается при публикации нового контракта (webhook).
- `can-i-deploy` проверяется перед деплоем.
Нарисуйте диаграмму и опишите каждый шаг.

### Задание 5: Анализ поломки контракта
Дано: `payment-service` добавил обязательное поле `paymentMethod` в запрос.
`order-service` не обновлён. Опишите:
- Как контрактное тестирование обнаружит проблему.
- На каком этапе pipeline произойдёт блокировка.
- Какие шаги нужны для корректной эволюции API.

---

## Дополнительные ресурсы

- **Документация**: [Pact](https://docs.pact.io/) — официальная документация
- **Документация**: [Spring Cloud Contract](https://spring.io/projects/spring-cloud-contract)
- **Инструмент**: [Pact Broker](https://github.com/pact-foundation/pact_broker) — Docker-образ
- **Статья**: Martin Fowler — "Consumer-Driven Contracts: A Service Evolution Pattern"
- **Видео**: "Contract Testing in Practice" (Pact official, YouTube)
- **Книга**: Sam Newman — "Building Microservices" (глава о контрактном тестировании)
- **Пример**: [Pact JVM Examples](https://github.com/pact-foundation/pact-jvm/tree/master/samples)
- **Курс**: "Contract Testing with Pact" (PactFlow, бесплатный)
