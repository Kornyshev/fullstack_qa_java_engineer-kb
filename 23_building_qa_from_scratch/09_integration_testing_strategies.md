# Стратегии интеграционного тестирования

## Обзор

Интеграционное тестирование проверяет, что отдельные модули системы корректно работают **вместе**. Unit-тесты доказывают,
что каждый кирпич прочный. Интеграционные тесты доказывают, что стена из этих кирпичей стоит ровно и не разваливается.

Выбор стратегии интеграционного тестирования — это не академическое упражнение. Это практическое решение, которое
определяет: как быстро вы найдёте баги на стыках модулей, сколько будет стоить поддержка тестов и насколько безопасно
команда будет деплоить в пятницу вечером (спойлер: не деплойте в пятницу).

В этом разделе мы разберём четыре классических стратегии интеграции, сравним их, а затем перейдём к реальности
микросервисной архитектуры, где всё немного сложнее.

---

## Стратегия 1: Top-Down Integration

### Суть подхода

Начинаем с верхнего уровня системы (UI, API Gateway, Controller) и движемся вниз. Нижние компоненты, которые ещё не
интегрированы, заменяем **заглушками (stubs)**.

### Как это работает

```
Шаг 1: Тестируем Controller со stub-ами Service и Repository
┌──────────────┐
│  Controller   │  ← тестируем
├──────────────┤
│  Service STUB │  ← заглушка
├──────────────┤
│  Repo STUB    │  ← заглушка
└──────────────┘

Шаг 2: Заменяем Service stub на реальный Service
┌──────────────┐
│  Controller   │  ← тестируем
├──────────────┤
│  Service REAL │  ← реальный
├──────────────┤
│  Repo STUB    │  ← заглушка
└──────────────┘

Шаг 3: Заменяем Repository stub на реальный Repository
┌──────────────┐
│  Controller   │  ← тестируем
├──────────────┤
│  Service REAL │  ← реальный
├──────────────┤
│  Repo REAL    │  ← реальный (с тестовой БД)
└──────────────┘
```

### Пример кода

```java
// Шаг 1: Controller + stub Service
@WebMvcTest(OrderController.class)
class OrderControllerTopDownTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean // Заглушка — вместо реального сервиса
    private OrderService orderService;

    @Test
    void createOrder_returnsCreated() throws Exception {
        // Настраиваем заглушку: при вызове createOrder вернуть заготовленный ответ
        when(orderService.createOrder(any()))
            .thenReturn(new OrderDto(1L, "CREATED", BigDecimal.valueOf(1500)));

        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"productId\": 1, \"quantity\": 2}"))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.status").value("CREATED"));
    }
}

// Шаг 2: Controller + реальный Service + stub Repository
@SpringBootTest
@AutoConfigureMockMvc
class OrderControllerServiceIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean // Теперь только Repository — заглушка
    private OrderRepository orderRepository;

    @Test
    void createOrder_withRealService_calculatesTotal() throws Exception {
        // Настраиваем заглушку Repository
        when(orderRepository.save(any()))
            .thenAnswer(inv -> {
                Order order = inv.getArgument(0);
                order.setId(1L);
                return order;
            });

        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"productId\": 1, \"quantity\": 2}"))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.total").value(1500)); // Бизнес-логика Service проверена
    }
}
```

### Плюсы

- Баги в верхних слоях находятся рано
- Можно демонстрировать работающий UI/API до готовности всех компонентов
- Естественный порядок: сначала то, что видит пользователь

### Минусы

- Много stub-ов на нижних уровнях — их нужно поддерживать
- Нижние модули тестируются в последнюю очередь — баги в БД, файловой системе, внешних сервисах находятся поздно
- Stub-ы могут не отражать реальное поведение (stub возвращает данные, а реальный сервис бросает исключение)

### Когда использовать

- UI-первый подход к разработке
- Прототипирование: нужно быстро показать работающий интерфейс
- Когда нижние слои разрабатываются другой командой и ещё не готовы

---

## Стратегия 2: Bottom-Up Integration

### Суть подхода

Начинаем с нижнего уровня (Repository, внешние сервисы, утилиты) и движемся вверх. Верхние компоненты, которые ещё не
интегрированы, заменяем **драйверами (test drivers)** — специальными вызывающими модулями.

### Как это работает

```
Шаг 1: Тестируем Repository с реальной БД
┌──────────────┐
│  Driver       │  ← тестовый драйвер (вызывает Repository напрямую)
├──────────────┤
│  Repo REAL    │  ← тестируем с реальной БД
└──────────────┘

Шаг 2: Добавляем реальный Service
┌──────────────┐
│  Driver       │  ← тестовый драйвер (вызывает Service напрямую)
├──────────────┤
│  Service REAL │  ← реальный
├──────────────┤
│  Repo REAL    │  ← реальный
└──────────────┘

Шаг 3: Добавляем Controller
┌──────────────┐
│  Controller   │  ← реальный, полная интеграция
├──────────────┤
│  Service REAL │  ← реальный
├──────────────┤
│  Repo REAL    │  ← реальный
└──────────────┘
```

### Пример кода

```java
// Шаг 1: Repository с реальной тестовой БД (TestContainers)
@DataJpaTest
@Testcontainers
class OrderRepositoryBottomUpTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void save_persistsOrderToDatabase() {
        // Тестируем Repository напрямую — это наш «нижний» уровень
        Order order = new Order();
        order.setProductId(1L);
        order.setQuantity(2);
        order.setStatus("CREATED");

        Order saved = orderRepository.save(order);

        assertThat(saved.getId()).isNotNull();
        assertThat(orderRepository.findById(saved.getId())).isPresent();
    }
}

// Шаг 2: Service + Repository (тестовый драйвер вызывает Service напрямую)
@SpringBootTest
@Testcontainers
class OrderServiceBottomUpTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @Autowired
    private OrderService orderService; // Реальный сервис

    @Test
    void createOrder_calculatesTotalAndPersists() {
        // Driver: вызываем Service напрямую (без HTTP, без Controller)
        OrderDto result = orderService.createOrder(new CreateOrderRequest(1L, 2));

        assertThat(result.getStatus()).isEqualTo("CREATED");
        assertThat(result.getTotal()).isEqualTo(BigDecimal.valueOf(1500));
    }
}
```

### Плюсы

- Нижние модули проверены тщательно — фундамент надёжный
- Не нужны stub-ы (используем реальные компоненты)
- Баги в работе с БД, файлами, внешними сервисами находятся рано

### Минусы

- Пользовательский интерфейс / API тестируется в последнюю очередь
- Нельзя показать работающий продукт до интеграции верхних слоёв
- Нужны test drivers для вызова нижних модулей

### Когда использовать

- Data-intensive приложения (основная логика — работа с данными)
- Когда нижние модули критичны (финансовые расчёты, работа с БД)
- Когда верхние слои часто меняются, а нижние стабильны

---

## Стратегия 3: Sandwich (Hybrid) Integration

### Суть подхода

Комбинируем top-down и bottom-up: тестируем верхние и нижние слои параллельно, потом встречаемся в середине.

### Как это работает

```
Параллельно:

Поток 1 (top-down):           Поток 2 (bottom-up):
┌──────────────┐               ┌──────────────┐
│  Controller   │               │  Driver       │
├──────────────┤               ├──────────────┤
│  Service STUB │               │  Repo REAL    │
└──────────────┘               └──────────────┘

Затем — встречаемся в середине:
┌──────────────┐
│  Controller   │
├──────────────┤
│  Service REAL │  ← точка встречи
├──────────────┤
│  Repo REAL    │
└──────────────┘
```

### Плюсы

- Баги находятся быстро на всех уровнях
- Параллельная работа двух команд (фронтенд-команда идёт сверху, бэкенд — снизу)
- Гибкость: можно адаптировать к конкретному проекту

### Минусы

- Сложнее в организации: нужна координация между командами
- Больше тестовых артефактов (и stub-ы, и drivers)
- Момент «встречи» в середине может выявить несовместимости

### Когда использовать

- Большие проекты с несколькими командами
- Когда и UI, и data layer критичны одновременно
- Когда есть чёткое разделение на фронтенд и бэкенд команды

---

## Стратегия 4: Big Bang Integration

### Суть подхода

Все модули разрабатываются отдельно, затем интегрируются **все сразу** и тестируются как единое целое.

### Как это работает

```
Разработка (отдельно):
[Controller]  [Service]  [Repository]  [EmailService]  [PaymentClient]

Интеграция (всё сразу):
┌──────────────────────────────────────────────────┐
│  Controller → Service → Repository               │
│                ↓                                  │
│           EmailService                            │
│                ↓                                  │
│          PaymentClient                            │
└──────────────────────────────────────────────────┘
← тестируем всё вместе
```

### Плюсы

- Минимум подготовительной работы: не нужны stub-ы и drivers
- Просто организовать: «собрали → запустили → посмотрели»
- Тестирует реальное взаимодействие всех компонентов

### Минусы

- **Локализация бага — кошмар.** Если тест упал — непонятно, какой из 10 модулей виноват
- Все модули должны быть готовы одновременно — нельзя начать тестирование раньше
- Высокий риск: все баги на стыках обнаруживаются в последний момент
- Очень дорогой debugging: ошибка в одном модуле маскируется ошибками в другом

### Когда использовать

- Маленькие проекты (3-5 модулей, один разработчик)
- Прототипы и MVP, где скорость важнее качества
- Legacy-системы, где нет возможности тестировать по частям
- **НИКОГДА** для больших или критичных систем

### Реальная история

Команда из 4 разработчиков два месяца писала микросервисную систему из 5 сервисов. Каждый разработчик тестировал свой
сервис unit-тестами. В день интеграции собрали всё вместе — ничего не работало. Сервис A отправлял `userId` как число,
сервис B ожидал строку. Сервис C отправлял даты в формате `dd-MM-yyyy`, сервис D ожидал ISO 8601. На исправление
интеграционных багов ушло 3 недели — почти столько же, сколько на разработку.

**Мораль:** интегрируйте рано и часто.

---

## Интеграционное тестирование в микросервисах

В микросервисной архитектуре интеграционное тестирование приобретает новые уровни:

### Service-Level Integration (внутри одного сервиса)

Проверяем, что все слои одного микросервиса работают вместе: Controller → Service → Repository → Database.

```java
// Полный интеграционный тест одного сервиса с TestContainers
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class OrderServiceFullIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void fullOrderLifecycle() {
        // Создаём заказ через HTTP (реальный Controller, Service, Repository, DB)
        var createResponse = restTemplate.postForEntity(
            "/api/orders",
            new CreateOrderRequest(1L, 2),
            OrderDto.class
        );
        assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);

        Long orderId = createResponse.getBody().getId();

        // Получаем заказ
        var getResponse = restTemplate.getForEntity(
            "/api/orders/" + orderId,
            OrderDto.class
        );
        assertThat(getResponse.getBody().getStatus()).isEqualTo("CREATED");

        // Отменяем заказ
        restTemplate.put("/api/orders/" + orderId + "/cancel", null);

        // Проверяем статус
        var cancelledResponse = restTemplate.getForEntity(
            "/api/orders/" + orderId,
            OrderDto.class
        );
        assertThat(cancelledResponse.getBody().getStatus()).isEqualTo("CANCELLED");
    }
}
```

### Cross-Service Integration (между сервисами)

Проверяем, что два или более сервисов корректно общаются друг с другом.

```java
// Тест взаимодействия Order Service и Payment Service
@SpringBootTest
@Testcontainers
class OrderPaymentIntegrationTest {

    @Container
    static DockerComposeContainer<?> environment =
        new DockerComposeContainer<>(new File("docker-compose-test.yml"))
            .withExposedService("order-service", 8080)
            .withExposedService("payment-service", 8081)
            .withExposedService("postgres", 5432);

    @Test
    void orderCreation_triggersPayment() {
        // Создаём заказ в Order Service
        var orderResponse = given()
            .baseUri("http://localhost:" + environment.getServicePort("order-service", 8080))
            .contentType(ContentType.JSON)
            .body(new CreateOrderRequest(1L, 2))
            .post("/api/orders")
            .then()
            .statusCode(201)
            .extract().as(OrderDto.class);

        // Ждём обработки и проверяем, что Payment Service создал платёж
        await().atMost(Duration.ofSeconds(10)).untilAsserted(() -> {
            var paymentResponse = given()
                .baseUri("http://localhost:" + environment.getServicePort("payment-service", 8081))
                .get("/api/payments?orderId=" + orderResponse.getId())
                .then()
                .statusCode(200)
                .extract().as(PaymentDto.class);

            assertThat(paymentResponse.getStatus()).isEqualTo("PENDING");
        });
    }
}
```

### Contract Testing (контрактное тестирование)

Вместо тяжёлых cross-service тестов проверяем **контракт** между сервисами.

```java
// Consumer side (Order Service ожидает определённый ответ от Payment Service)
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "payment-service")
class PaymentClientContractTest {

    @Pact(consumer = "order-service")
    V4Pact createPaymentPact(PactDslWithProvider builder) {
        return builder
            .given("пользователь существует")
            .uponReceiving("запрос на создание платежа")
            .method("POST")
            .path("/api/payments")
            .body(new PactDslJsonBody()
                .numberType("orderId", 1L)
                .decimalType("amount", 1500.00))
            .willRespondWith()
            .status(201)
            .body(new PactDslJsonBody()
                .numberType("paymentId")
                .stringType("status", "PENDING"))
            .toPact(V4Pact.class);
    }

    @Test
    @PactTestFor(pactMethod = "createPaymentPact")
    void createPayment_returnsExpectedResponse(MockServer mockServer) {
        // Настраиваем клиент на mock-сервер
        PaymentClient client = new PaymentClient(mockServer.getUrl());
        PaymentResponse response = client.createPayment(1L, BigDecimal.valueOf(1500));

        assertThat(response.getStatus()).isEqualTo("PENDING");
    }
}
```

### TestContainers — основной инструмент

TestContainers позволяет поднимать реальные зависимости (БД, брокеры, кэши) в Docker-контейнерах прямо в тестах.

```java
// Пример: тест с PostgreSQL, Redis и Kafka
@SpringBootTest
@Testcontainers
class FullStackIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7")
        .withExposedPorts(6379);

    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.4.0")
    );

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        // Подключаем Spring Boot к тестовым контейнерам
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.redis.host", redis::getHost);
        registry.add("spring.redis.port", () -> redis.getMappedPort(6379));
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }

    @Test
    void orderCreation_publishesEventAndCachesResult() {
        // Полный тест: HTTP → Service → DB → Kafka → Redis
        // Все зависимости реальные, но изолированные
    }
}
```

---

## Сравнительная таблица стратегий

| Критерий | Top-Down | Bottom-Up | Sandwich | Big Bang |
|---|---|---|---|---|
| **Порядок интеграции** | Сверху вниз | Снизу вверх | Параллельно с обеих сторон | Всё сразу |
| **Заглушки** | Stubs для нижних слоёв | Drivers для верхних слоёв | И stubs, и drivers | Не нужны |
| **Ранний UI/API** | Да | Нет | Частично | Нет |
| **Ранняя проверка данных** | Нет | Да | Частично | Нет |
| **Локализация багов** | Хорошая | Хорошая | Хорошая | Плохая |
| **Подготовительная работа** | Средняя | Средняя | Высокая | Низкая |
| **Параллельная разработка** | Ограничена | Ограничена | Хорошо поддерживается | Нет |
| **Подходит для** | UI-first, API-first | Data-first | Большие команды | MVP, маленькие проекты |
| **Риск** | Поздние баги в data layer | Поздние баги в UI | Сложная координация | Поздние баги ВЕЗДЕ |

---

## Практический пример: выбор стратегии

### Задача

Система из трёх сервисов:
```
API Gateway → Order Service → Payment Service → PostgreSQL
```

### Анализ

| Компонент | Критичность | Готовность | Зависимости |
|---|---|---|---|
| API Gateway | Средняя (маршрутизация) | Готов | Order Service |
| Order Service | Высокая (бизнес-логика) | В разработке | Payment Service, PostgreSQL |
| Payment Service | Критическая (деньги!) | Готов | PostgreSQL, внешний Payment Provider |
| PostgreSQL | Инфраструктура | Всегда готов | — |

### Решение: Sandwich-подход

```
Поток 1 (Bottom-Up): Payment Service
─────────────────────────────────────
Шаг 1: Payment Service + PostgreSQL (TestContainers)
       → Проверяем: создание платежа, списание, возврат
       → Все сценарии с реальной БД

Шаг 2: Payment Service + Mock External Provider
       → Проверяем: обработку ответов от платёжной системы
       → Timeout, отказ, дубликат, успех

Поток 2 (Top-Down): API Gateway + Order Service
────────────────────────────────────────────────
Шаг 1: API Gateway + stub Order Service
       → Проверяем: маршрутизацию, авторизацию, rate limiting

Шаг 2: API Gateway + реальный Order Service + stub Payment Service
       → Проверяем: создание заказа, бизнес-логику

Точка встречи: Full Integration
───────────────────────────────
Шаг 3: API Gateway + Order Service + Payment Service + PostgreSQL
       → Docker-compose с TestContainers
       → E2E-сценарий: создать заказ → оплатить → проверить статусы
       → Contract testing между Order Service и Payment Service
```

### Количество тестов

| Уровень | Тесты | Время |
|---|---|---|
| Payment Service + DB | 20 тестов | 30 сек |
| Order Service + DB | 25 тестов | 30 сек |
| API Gateway + stubs | 15 тестов | 10 сек |
| Cross-service (docker-compose) | 10 тестов | 2 мин |
| Contract tests | 8 тестов | 5 сек |
| **Итого** | **78 тестов** | **~3.5 мин** |

---

## Связь с тестированием

Интеграционное тестирование — это **средний слой** тестовой пирамиды. Он связывает unit-тесты (которые проверяют
отдельные функции) с E2E-тестами (которые проверяют систему целиком).

```
       /  E2E-тесты  \         ← Мало, медленные, проверяют критичные пути
      / Интеграционные \        ← Средне, проверяют стыки модулей  ← МЫ ЗДЕСЬ
     /   Unit-тесты     \      ← Много, быстрые, проверяют логику
    ─────────────────────
```

Без интеграционных тестов у вас есть два крайних случая:
1. Unit-тесты проходят, но модули не работают вместе (классическая картинка: «два юнита зелёные, но вместе не стыкуются»)
2. E2E-тесты падают, но непонятно, какой модуль виноват

Интеграционные тесты закрывают этот разрыв: они достаточно детальные, чтобы локализовать баг, и достаточно широкие, чтобы
поймать проблемы на стыках.

---

## Типичные ошибки

### 1. Называть unit-тесты интеграционными

```java
// Это НЕ интеграционный тест — здесь всё замокано
@Test
void orderService_createOrder() {
    when(orderRepository.save(any())).thenReturn(new Order(1L, "CREATED"));
    when(paymentClient.createPayment(any())).thenReturn(new Payment("SUCCESS"));

    OrderDto result = orderService.createOrder(new CreateOrderRequest(1L, 2));

    assertThat(result.getStatus()).isEqualTo("CREATED");
}
// Этот тест не проверяет интеграцию — он проверяет логику Service в изоляции.
// Реальная БД, реальный Payment Service — всё замокано.
```

### 2. Тестировать только happy path

Интеграционные тесты должны проверять и сценарии ошибок: что происходит, когда Payment Service недоступен? Когда БД
вернула timeout? Когда Kafka-сообщение не доставлено?

### 3. Не использовать TestContainers

```java
// Плохо: тесты зависят от локальной БД
@Test
void test() {
    // Подключаемся к localhost:5432 — работает только на моей машине
    // У коллеги PostgreSQL на порте 5433
    // На CI PostgreSQL не установлен вообще
}

// Хорошо: TestContainers поднимает БД в Docker
@Container
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
// Работает одинаково на всех машинах и на CI
```

### 4. Слишком большая область интеграции

Не нужно в каждом интеграционном тесте поднимать все 10 сервисов, Kafka, Redis и три базы данных. Тестируйте
**минимально необходимый набор** компонентов для проверки конкретного стыка.

### 5. Игнорировать порядок запуска

В интеграционных тестах порядок запуска контейнеров важен: БД должна быть готова до того, как стартует приложение.
Используйте `@DependsOn`, `wait-for-it.sh` или `withStartupCheckStrategy()` в TestContainers.

### 6. Не чистить данные между тестами

```java
// Плохо: тесты загрязняют БД друг для друга
@Test
void test1() {
    orderRepository.save(new Order("test-order")); // Остаётся в БД
}

@Test
void test2() {
    List<Order> orders = orderRepository.findAll();
    assertThat(orders).hasSize(1); // Зависит от test1!
}

// Хорошо: каждый тест начинает с чистого состояния
@BeforeEach
void cleanup() {
    orderRepository.deleteAll();
}
```

---

## Вопросы на интервью

### 🟢 Junior

1. **Что такое интеграционное тестирование и чем оно отличается от unit-тестирования?**
   Unit-тестирование проверяет один модуль в изоляции (с mock-ами). Интеграционное — проверяет взаимодействие нескольких
   модулей вместе. Пример: unit-тест проверяет метод `calculateTotal()`, интеграционный — что Controller вызывает Service,
   тот обращается к Repository, а Repository корректно пишет в реальную БД.

2. **Зачем нужны интеграционные тесты, если есть unit-тесты?**
   Unit-тесты проверяют логику внутри модуля, но не проверяют стыки. Два модуля могут быть идеальными по отдельности,
   но не стыковаться: разный формат данных, неправильный URL, несовместимые версии.

3. **Что такое TestContainers?**
   Библиотека для Java, которая позволяет поднимать Docker-контейнеры (PostgreSQL, Redis, Kafka и т.д.) прямо в тестах.
   Каждый тестовый прогон получает чистый экземпляр зависимости — не нужно настраивать БД вручную.

### 🟡 Middle

4. **Опишите top-down и bottom-up стратегии. Когда используете какую?**
   Top-down: начинаем с Controller, stub-им Service и Repository. Хорошо для API-first разработки, когда важно рано
   показать работающий API. Bottom-up: начинаем с Repository + реальная БД, потом добавляем Service. Хорошо для
   data-intensive приложений, когда критична корректность работы с данными.

5. **Как организовать интеграционные тесты в микросервисном проекте?**
   Три уровня: 1) service-level (внутри сервиса, Controller → Service → DB через TestContainers), 2) contract tests
   (Pact между сервисами), 3) cross-service (docker-compose для критичных E2E-сценариев, минимум тестов). Большинство
   интеграционных тестов — на уровне 1.

6. **Как изолировать тестовые данные в интеграционных тестах?**
   `@BeforeEach` с очисткой таблиц. Уникальные данные (UUID в названиях). `@Transactional` с rollback (для Spring).
   Отдельная тестовая БД в TestContainers (каждый прогон — чистый контейнер).

7. **Почему big bang — плохая стратегия для больших систем?**
   Невозможно локализовать баг: если упал тест с 10 интегрированными модулями, непонятно, где проблема. Все модули
   должны быть готовы одновременно — нельзя начать тестирование раньше. Все баги обнаруживаются в конце, когда
   исправлять дорого.

### 🔴 Senior

8. **Как бы вы спроектировали стратегию интеграционного тестирования для системы из 8 микросервисов?**
   Определил бы критичные цепочки взаимодействий (обычно 3-5). Для каждого сервиса — service-level тесты с
   TestContainers. Между сервисами — contract tests (Pact Broker). Для 3-5 критичных E2E-сценариев — docker-compose
   с поднятием необходимых сервисов. Итого: ~80% тестов service-level, ~15% contract, ~5% cross-service.

9. **Contract testing vs integration testing — когда что использовать?**
   Contract testing: быстрый, не требует поднимать другие сервисы, проверяет совместимость API. Лучше для CI (каждый
   PR). Integration testing: медленнее, поднимает реальные сервисы, проверяет реальное поведение. Лучше для nightly runs
   или перед релизом. Комбинирую: contract в CI, integration — ночью.

10. **Как тестировать асинхронные взаимодействия между сервисами (через Kafka/RabbitMQ)?**
    TestContainers для Kafka. Отправляю сообщение в топик, через `Awaitility.await()` жду появления результата в БД или
    другом топике. Обязательно тестирую: happy path, ошибку обработки (DLQ), дубликат сообщения (идемпотентность),
    сообщение в неправильном формате (десериализация). Timeout на ожидание — не больше 30 секунд.

---

## Практические задания

### Задание 1: Классификация тестов
Даны 5 тестов (псевдокод). Определите для каждого: это unit-тест, интеграционный тест или E2E-тест? Обоснуйте.

```
a) Тест вызывает OrderService.calculateTotal() с mock OrderRepository → проверяет возвращаемое значение
b) Тест отправляет POST /api/orders на реально запущенное приложение с TestContainers PostgreSQL
c) Тест поднимает 3 сервиса в docker-compose и проходит сценарий «заказ → оплата → доставка»
d) Тест вызывает OrderRepository.save() на реальной PostgreSQL через TestContainers
e) Тест отправляет HTTP-запрос через MockMvc с @MockBean на все зависимости
```

### Задание 2: Выбор стратегии
Дана система: монолитное Spring Boot приложение с 4 слоями (Controller → Service → Repository → PostgreSQL) и
2 внешними интеграциями (Payment Gateway, Email Service). Выберите стратегию интеграционного тестирования и обоснуйте.
Нарисуйте схему пошагового подключения модулей.

### Задание 3: TestContainers
Напишите интеграционный тест для `UserRepository` с TestContainers PostgreSQL. Тест должен проверять: создание
пользователя, поиск по email, обновление имени, удаление. Используйте `@BeforeEach` для очистки данных.

### Задание 4: Contract Test
Опишите (текстом или псевдокодом) consumer-driven contract test между Order Service (consumer) и Inventory Service
(provider). Order Service вызывает `GET /api/inventory/{productId}` и ожидает: `{ "productId": 1, "available": 50 }`.

### Задание 5: Проектирование стратегии для микросервисов
Дана система: API Gateway → [User Service, Product Service, Order Service, Notification Service]. Order Service
зависит от User Service и Product Service. Notification Service слушает события от Order Service через Kafka.
Спроектируйте стратегию интеграционного тестирования: какие тесты, на каком уровне, какие инструменты, сколько тестов
на каждом уровне.

---

## Дополнительные ресурсы

- **Библиотека:** [TestContainers](https://testcontainers.com/) — Docker-контейнеры в тестах
- **Инструмент:** [Pact](https://docs.pact.io/) — consumer-driven contract testing
- **Инструмент:** [Spring Cloud Contract](https://spring.io/projects/spring-cloud-contract) — contract testing для Spring
- **Книга:** "Growing Object-Oriented Software, Guided by Tests" — Steve Freeman, Nat Pryce
- **Статья:** [Testing Strategies in a Microservice Architecture](https://martinfowler.com/articles/microservice-testing/) — Toby Clemson (Martin Fowler's blog)
- **Библиотека:** [Awaitility](https://github.com/awaitility/awaitility) — ожидание асинхронных операций в тестах
- **Видео:** "Integration Testing with TestContainers" — YouTube (множество докладов с конференций)
- **Статья:** [Practical Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html) — Ham Vocke
