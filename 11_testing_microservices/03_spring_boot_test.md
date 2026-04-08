# Spring Boot Test для QA

## Обзор

Spring Boot Test — фреймворк для тестирования Spring Boot приложений, который предоставляет
набор аннотаций, утилит и auto-конфигураций для написания тестов на разных уровнях:
от изолированного тестирования отдельного контроллера до полноценного поднятия всего приложения.

Для QA-инженера Spring Boot Test является основным инструментом при написании автотестов
для backend-сервисов. Понимание test slices (`@WebMvcTest`, `@DataJpaTest`), работы с моками
(`@MockBean`, `@SpyBean`), конфигурации (`@TestConfiguration`, test profiles) и HTTP-клиентов
(`TestRestTemplate`, `MockMvc`) позволяет эффективно тестировать сервисы на всех уровнях.

Этот раздел охватывает все ключевые возможности Spring Boot Test с примерами и рекомендациями
для QA-инженеров.

---

## @SpringBootTest — полный контекст приложения

### Когда использовать

`@SpringBootTest` поднимает полный Spring Application Context — все бины, конфигурации,
подключения к БД. Это самый «тяжёлый» вид теста, но и самый реалистичный.

### Режимы запуска

```java
// Без веб-сервера (по умолчанию) — для тестирования сервисного слоя
@SpringBootTest
class OrderServiceTest { }

// С реальным веб-сервером на случайном порту — для интеграционных тестов
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OrderControllerIntegrationTest { }

// С реальным веб-сервером на фиксированном порту
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
class OrderControllerFixedPortTest { }

// Без веб-окружения (не создаёт WebApplicationContext)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
class BatchJobTest { }
```

### Пример: интеграционный тест с реальным HTTP

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class OrderControllerIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @DynamicPropertySource
    static void configureDb(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    TestRestTemplate restTemplate;

    @Test
    void shouldCreateOrder() {
        var request = new CreateOrderRequest("item-1", 3, BigDecimal.valueOf(1500));

        var response = restTemplate.postForEntity("/api/orders", request, OrderResponse.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody()).isNotNull();
        assertThat(response.getBody().getOrderId()).isNotBlank();
        assertThat(response.getBody().getStatus()).isEqualTo("PENDING");
    }

    @Test
    void shouldReturn404ForNonExistentOrder() {
        var response = restTemplate.getForEntity("/api/orders/non-existent", OrderResponse.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND);
    }
}
```

---

## @WebMvcTest — тестирование контроллеров с MockMvc

### Что это такое

`@WebMvcTest` — test slice, который поднимает только веб-слой: контроллеры, фильтры,
`@ControllerAdvice`, конвертеры. Сервисы и репозитории не загружаются — их нужно мокировать.

### Преимущества

- **Скорость**: поднимается только часть контекста (секунды, а не десятки секунд).
- **Изоляция**: тестируем только логику контроллера, не зависим от БД или внешних сервисов.
- **Детальность**: MockMvc позволяет проверять HTTP-статусы, заголовки, тело ответа, валидацию.

### Пример: тестирование контроллера заказов

```java
// Контроллер
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderController {

    private final OrderService orderService;

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public OrderResponse createOrder(@Valid @RequestBody CreateOrderRequest request) {
        return orderService.createOrder(request);
    }

    @GetMapping("/{orderId}")
    public OrderResponse getOrder(@PathVariable String orderId) {
        return orderService.getOrder(orderId);
    }

    @GetMapping
    public List<OrderResponse> listOrders(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return orderService.listOrders(page, size);
    }
}
```

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired
    MockMvc mockMvc;

    @MockBean
    OrderService orderService;

    @Autowired
    ObjectMapper objectMapper;

    @Test
    void shouldCreateOrder() throws Exception {
        // Подготовка: мокируем ответ сервиса
        var expectedResponse = new OrderResponse("order-1", "PENDING", BigDecimal.valueOf(4500));
        when(orderService.createOrder(any())).thenReturn(expectedResponse);

        var request = new CreateOrderRequest("item-1", 3, BigDecimal.valueOf(1500));

        // Действие и проверка
        mockMvc.perform(post("/api/orders")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.orderId").value("order-1"))
                .andExpect(jsonPath("$.status").value("PENDING"))
                .andExpect(jsonPath("$.totalAmount").value(4500));

        // Проверяем, что сервис был вызван с правильными параметрами
        verify(orderService).createOrder(argThat(req ->
                req.getItemId().equals("item-1") && req.getQuantity() == 3));
    }

    @Test
    void shouldReturn400ForInvalidRequest() throws Exception {
        // Запрос без обязательного поля itemId
        var invalidRequest = """
                {"quantity": 3, "price": 1500}
                """;

        mockMvc.perform(post("/api/orders")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(invalidRequest))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.errors").isArray())
                .andExpect(jsonPath("$.errors[0].field").value("itemId"));
    }

    @Test
    void shouldReturn404WhenOrderNotFound() throws Exception {
        when(orderService.getOrder("non-existent"))
                .thenThrow(new OrderNotFoundException("non-existent"));

        mockMvc.perform(get("/api/orders/non-existent"))
                .andExpect(status().isNotFound())
                .andExpect(jsonPath("$.message").value("Заказ не найден: non-existent"));
    }

    @Test
    void shouldReturnPaginatedOrders() throws Exception {
        var orders = List.of(
                new OrderResponse("order-1", "PENDING", BigDecimal.valueOf(1000)),
                new OrderResponse("order-2", "CONFIRMED", BigDecimal.valueOf(2000))
        );
        when(orderService.listOrders(0, 10)).thenReturn(orders);

        mockMvc.perform(get("/api/orders")
                        .param("page", "0")
                        .param("size", "10"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$", hasSize(2)))
                .andExpect(jsonPath("$[0].orderId").value("order-1"))
                .andExpect(jsonPath("$[1].orderId").value("order-2"));
    }
}
```

### Тестирование валидации

```java
// DTO с валидационными аннотациями
public record CreateOrderRequest(
        @NotBlank(message = "itemId обязателен")
        String itemId,

        @Min(value = 1, message = "Количество должно быть >= 1")
        @Max(value = 100, message = "Количество не должно превышать 100")
        int quantity,

        @NotNull(message = "Цена обязательна")
        @Positive(message = "Цена должна быть положительной")
        BigDecimal price
) {}
```

```java
@Test
void shouldValidateMinQuantity() throws Exception {
    var request = """
            {"itemId": "item-1", "quantity": 0, "price": 100}
            """;

    mockMvc.perform(post("/api/orders")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(request))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors[?(@.field=='quantity')].message")
                    .value("Количество должно быть >= 1"));
}

@Test
void shouldValidateNegativePrice() throws Exception {
    var request = """
            {"itemId": "item-1", "quantity": 1, "price": -100}
            """;

    mockMvc.perform(post("/api/orders")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(request))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors[?(@.field=='price')].message")
                    .value("Цена должна быть положительной"));
}
```

---

## @DataJpaTest — тестирование репозиториев

### Что это такое

`@DataJpaTest` — test slice для тестирования JPA-слоя. Поднимает только JPA-компоненты:
репозитории, `EntityManager`, Flyway/Liquibase миграции. По умолчанию использует in-memory H2.

### Пример: тестирование репозитория заказов

```java
@DataJpaTest
@Testcontainers
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class OrderRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @DynamicPropertySource
    static void dbProps(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    OrderRepository orderRepository;

    @Autowired
    TestEntityManager entityManager;

    @Test
    void shouldSaveAndFindOrder() {
        // Сохраняем заказ
        var order = new OrderEntity();
        order.setCustomerId("customer-1");
        order.setStatus(OrderStatus.PENDING);
        order.setTotalAmount(BigDecimal.valueOf(5000));

        entityManager.persistAndFlush(order);
        entityManager.clear(); // Очищаем кэш первого уровня для честного чтения из БД

        // Ищем заказ
        var found = orderRepository.findById(order.getId());

        assertThat(found).isPresent();
        assertThat(found.get().getCustomerId()).isEqualTo("customer-1");
        assertThat(found.get().getStatus()).isEqualTo(OrderStatus.PENDING);
        assertThat(found.get().getTotalAmount()).isEqualByComparingTo(BigDecimal.valueOf(5000));
    }

    @Test
    void shouldFindOrdersByStatus() {
        // Создаём заказы с разными статусами
        createOrder("customer-1", OrderStatus.PENDING, BigDecimal.valueOf(1000));
        createOrder("customer-2", OrderStatus.CONFIRMED, BigDecimal.valueOf(2000));
        createOrder("customer-3", OrderStatus.PENDING, BigDecimal.valueOf(3000));

        entityManager.flush();
        entityManager.clear();

        // Ищем заказы со статусом PENDING
        var pendingOrders = orderRepository.findByStatus(OrderStatus.PENDING);

        assertThat(pendingOrders).hasSize(2);
        assertThat(pendingOrders).allSatisfy(order ->
                assertThat(order.getStatus()).isEqualTo(OrderStatus.PENDING));
    }

    @Test
    void shouldFindOrdersByCustomerIdAndDateRange() {
        var now = LocalDateTime.now();
        createOrderWithDate("customer-1", now.minusDays(5));
        createOrderWithDate("customer-1", now.minusDays(2));
        createOrderWithDate("customer-1", now.minusDays(1));
        createOrderWithDate("customer-2", now.minusDays(1)); // Другой клиент

        entityManager.flush();
        entityManager.clear();

        var orders = orderRepository.findByCustomerIdAndCreatedAtBetween(
                "customer-1",
                now.minusDays(3),
                now
        );

        // Только 2 заказа customer-1 за последние 3 дня
        assertThat(orders).hasSize(2);
    }

    @Test
    void shouldCalculateTotalRevenueByStatus() {
        createOrder("c-1", OrderStatus.CONFIRMED, BigDecimal.valueOf(1000));
        createOrder("c-2", OrderStatus.CONFIRMED, BigDecimal.valueOf(2500));
        createOrder("c-3", OrderStatus.CANCELLED, BigDecimal.valueOf(500));

        entityManager.flush();
        entityManager.clear();

        // Кастомный запрос: сумма подтверждённых заказов
        var revenue = orderRepository.calculateTotalRevenueByStatus(OrderStatus.CONFIRMED);

        assertThat(revenue).isEqualByComparingTo(BigDecimal.valueOf(3500));
    }

    private void createOrder(String customerId, OrderStatus status, BigDecimal amount) {
        var order = new OrderEntity();
        order.setCustomerId(customerId);
        order.setStatus(status);
        order.setTotalAmount(amount);
        entityManager.persist(order);
    }

    private void createOrderWithDate(String customerId, LocalDateTime createdAt) {
        var order = new OrderEntity();
        order.setCustomerId(customerId);
        order.setStatus(OrderStatus.PENDING);
        order.setTotalAmount(BigDecimal.valueOf(1000));
        order.setCreatedAt(createdAt);
        entityManager.persist(order);
    }
}
```

### @DataJpaTest: ключевые особенности

- Каждый тест выполняется в транзакции, которая откатывается в конце (`@Transactional`).
- По умолчанию используется H2 in-memory. Для реальной БД — `@AutoConfigureTestDatabase(replace = NONE)` + Testcontainers.
- Доступен `TestEntityManager` — обёртка над JPA `EntityManager` с удобными методами для тестов.

---

## Test Slices — обзор

### Все доступные test slices

| Аннотация              | Что поднимает                       | Типичное применение                 |
|------------------------|-------------------------------------|-------------------------------------|
| `@WebMvcTest`          | Controllers, filters, advice        | Тестирование REST API               |
| `@DataJpaTest`         | JPA repositories, EntityManager     | Тестирование запросов к БД          |
| `@DataMongoTest`       | MongoDB repositories                | Тестирование MongoDB-запросов       |
| `@DataRedisTest`       | Redis repositories                  | Тестирование Redis-операций         |
| `@RestClientTest`      | RestTemplate, RestClient            | Тестирование HTTP-клиентов          |
| `@JsonTest`            | Jackson ObjectMapper                | Тестирование сериализации           |
| `@WebFluxTest`         | WebFlux controllers (reactive)      | Тестирование реактивных контроллеров|

### @JsonTest — тестирование сериализации/десериализации

```java
@JsonTest
class OrderResponseJsonTest {

    @Autowired
    JacksonTester<OrderResponse> json;

    @Test
    void shouldSerializeOrderResponse() throws Exception {
        var order = new OrderResponse("order-1", "PENDING", BigDecimal.valueOf(5000));

        var result = json.write(order);

        assertThat(result).hasJsonPathStringValue("$.orderId", "order-1");
        assertThat(result).hasJsonPathStringValue("$.status", "PENDING");
        assertThat(result).hasJsonPathNumberValue("$.totalAmount", 5000);
        // Проверяем, что нет лишних полей
        assertThat(result).doesNotHaveJsonPath("$.internalId");
    }

    @Test
    void shouldDeserializeOrderResponse() throws Exception {
        var content = """
                {"orderId": "order-1", "status": "PENDING", "totalAmount": 5000}
                """;

        var result = json.parse(content);

        assertThat(result.getObject().getOrderId()).isEqualTo("order-1");
        assertThat(result.getObject().getStatus()).isEqualTo("PENDING");
    }
}
```

---

## @MockBean vs @SpyBean

### @MockBean

Заменяет бин в Spring Context полным моком (все методы возвращают `null`/defaults).

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @MockBean
    OrderService orderService; // Полностью замокирован — нужно настраивать when/thenReturn

    @Test
    void shouldReturnOrder() throws Exception {
        // Без when — orderService.getOrder вернёт null
        when(orderService.getOrder("order-1"))
                .thenReturn(new OrderResponse("order-1", "PENDING", BigDecimal.TEN));

        mockMvc.perform(get("/api/orders/order-1"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.orderId").value("order-1"));
    }
}
```

### @SpyBean

Оборачивает реальный бин — по умолчанию вызывается реальная реализация,
но отдельные методы можно переопределить.

```java
@SpringBootTest
class NotificationServiceTest {

    @SpyBean
    NotificationService notificationService; // Реальная реализация с возможностью перехвата

    @Autowired
    OrderService orderService;

    @Test
    void shouldSendNotificationOnOrderCreation() {
        // NotificationService работает как реальный, но мы можем перехватить вызов email
        doNothing().when(notificationService).sendEmail(any()); // Не отправляем реальные email

        orderService.createOrder(new CreateOrderRequest("item-1", 1, BigDecimal.TEN));

        // Проверяем, что метод был вызван
        verify(notificationService).sendEmail(argThat(email ->
                email.getSubject().contains("Заказ создан")));
    }
}
```

### Когда что использовать

| Ситуация                                          | Рекомендация      |
|---------------------------------------------------|-------------------|
| Тестирование контроллера без сервисного слоя      | `@MockBean`       |
| Замена внешнего HTTP-клиента                       | `@MockBean`       |
| Перехват отправки email без отключения логики      | `@SpyBean`        |
| Проверка, что метод вызван с определёнными аргументами | `@SpyBean`   |
| Полная изоляция от зависимости                     | `@MockBean`       |

### Важное ограничение

`@MockBean` и `@SpyBean` инвалидируют кэш Spring Context. Если разные тесты мокают
разные бины, Spring создаёт отдельный контекст для каждого теста, что замедляет запуск.

**Рекомендация**: группируйте тесты с одинаковыми моками в одном классе или используйте
`@TestConfiguration` для общих моков.

---

## TestRestTemplate

### Когда использовать

`TestRestTemplate` доступен при `webEnvironment = RANDOM_PORT` или `DEFINED_PORT`.
Это реальный HTTP-клиент, который отправляет запросы на запущенный сервер.

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OrderApiTest {

    @Autowired
    TestRestTemplate restTemplate;

    @Test
    void shouldCreateAndRetrieveOrder() {
        // Создание заказа
        var createRequest = new CreateOrderRequest("item-1", 2, BigDecimal.valueOf(500));
        var createResponse = restTemplate.postForEntity("/api/orders", createRequest, OrderResponse.class);

        assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        var orderId = createResponse.getBody().getOrderId();

        // Получение заказа
        var getResponse = restTemplate.getForEntity("/api/orders/" + orderId, OrderResponse.class);

        assertThat(getResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(getResponse.getBody().getStatus()).isEqualTo("PENDING");
    }

    @Test
    void shouldHandleAuthentication() {
        // TestRestTemplate с credentials
        var authenticatedTemplate = restTemplate.withBasicAuth("admin", "password");

        var response = authenticatedTemplate.getForEntity("/api/admin/orders", String.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
    }
}
```

### TestRestTemplate vs MockMvc

| Характеристика         | MockMvc                            | TestRestTemplate                   |
|------------------------|------------------------------------|------------------------------------|
| Реальный HTTP          | Нет (in-process)                   | Да                                 |
| Скорость               | Быстрее                           | Медленнее                          |
| Фильтры/Interceptors  | Можно пропустить                   | Всегда включены                    |
| Error handling         | Получаем MvcResult                 | Получаем HTTP-ответ                |
| Для чего               | Unit-тесты контроллера             | Интеграционные тесты               |

---

## @TestConfiguration и тестовые профили

### @TestConfiguration — дополнительная конфигурация для тестов

```java
// Тестовая конфигурация, заменяющая реальный PaymentClient на мок
@TestConfiguration
public class TestPaymentConfig {

    @Bean
    @Primary // Заменяет реальный бин
    public PaymentClient testPaymentClient() {
        var mock = Mockito.mock(PaymentClient.class);
        // Настраиваем поведение по умолчанию
        when(mock.processPayment(any()))
                .thenReturn(new PaymentResponse("pay-1", "APPROVED"));
        return mock;
    }
}
```

```java
// Использование в тесте
@SpringBootTest
@Import(TestPaymentConfig.class)
class OrderServiceWithMockPaymentTest {

    @Autowired
    OrderService orderService;

    @Autowired
    PaymentClient paymentClient; // Это будет мок из TestPaymentConfig

    @Test
    void shouldCreateOrderWithMockedPayment() {
        var response = orderService.createOrder(
                new CreateOrderRequest("item-1", 1, BigDecimal.valueOf(100)));

        assertThat(response.getStatus()).isEqualTo("CONFIRMED");
    }
}
```

### Тестовые профили (application-test.yml)

```yaml
# src/test/resources/application-test.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true

# Отключаем внешние зависимости
payment-service:
  url: http://localhost:8888  # WireMock порт
  timeout: 2s

kafka:
  enabled: false  # Отключаем Kafka для быстрых тестов
```

```java
// Активация профиля в тесте
@SpringBootTest
@ActiveProfiles("test")
class OrderServiceTestProfileTest {
    // Использует конфигурацию из application-test.yml
}
```

### Несколько профилей

```java
// Комбинация профилей: test + kafka
@SpringBootTest
@ActiveProfiles({"test", "kafka"})
class OrderServiceWithKafkaTest {
    // application-test.yml + application-kafka.yml
}
```

---

## Когда QA пишет Spring Boot тесты vs когда разработчики

### Зоны ответственности

| Тип теста                               | Кто пишет         | Обоснование                              |
|-----------------------------------------|--------------------|------------------------------------------|
| Unit-тесты бизнес-логики               | Разработчик        | Знает детали реализации                  |
| @WebMvcTest (контроллеры)              | Разработчик / QA   | QA проверяет валидацию и error handling  |
| @DataJpaTest (репозитории)             | Разработчик / QA   | QA проверяет граничные случаи запросов   |
| Component тесты (@SpringBootTest)      | QA / Разработчик   | QA тестирует сервис как чёрный ящик      |
| Интеграционные тесты (несколько сервисов)| QA               | QA отвечает за межсервисное взаимодействие|
| E2E-тесты                              | QA                 | Бизнес-сценарии — зона ответственности QA|
| Contract тесты (consumer)              | Разработчик / QA   | Зависит от организации в команде         |

### Когда QA точно должен писать Spring Boot тесты

1. **Тестирование через API** — QA тестирует REST API как чёрный ящик, проверяя:
   - Контракты (формат запроса/ответа).
   - Валидацию (граничные значения, невалидные данные).
   - Обработку ошибок (4xx, 5xx).
   - Авторизацию (доступ по ролям).

2. **Интеграция с внешними системами** — QA проверяет корректность взаимодействия
   с другими сервисами, БД, очередями.

3. **Регрессионные тесты** — автоматизация ручных тестов, которые проверяют
   критические бизнес-сценарии.

---

## Продвинутые техники

### Тестирование с авторизацией (Spring Security)

```java
@WebMvcTest(AdminController.class)
class AdminControllerSecurityTest {

    @Autowired
    MockMvc mockMvc;

    @MockBean
    AdminService adminService;

    @Test
    @WithMockUser(roles = "ADMIN")
    void adminShouldAccessAdminEndpoint() throws Exception {
        mockMvc.perform(get("/api/admin/stats"))
                .andExpect(status().isOk());
    }

    @Test
    @WithMockUser(roles = "USER")
    void regularUserShouldBeDeniedAccess() throws Exception {
        mockMvc.perform(get("/api/admin/stats"))
                .andExpect(status().isForbidden());
    }

    @Test
    void anonymousUserShouldBeUnauthorized() throws Exception {
        mockMvc.perform(get("/api/admin/stats"))
                .andExpect(status().isUnauthorized());
    }
}
```

### @DynamicPropertySource — динамические свойства

```java
@SpringBootTest
@Testcontainers
class DynamicPropertyTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7")
            .withExposedPorts(6379);

    @DynamicPropertySource
    static void registerProperties(DynamicPropertyRegistry registry) {
        // Динамически подставляем URL, порты и credentials из контейнеров
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add("spring.data.redis.port", () -> redis.getMappedPort(6379));
    }
}
```

### Тестирование асинхронных операций

```java
@SpringBootTest
class AsyncOrderProcessingTest {

    @Autowired
    OrderService orderService;

    @Autowired
    OrderRepository orderRepository;

    @Test
    void shouldProcessOrderAsynchronously() {
        // Создаём заказ (обработка происходит асинхронно)
        var orderId = orderService.submitOrder(new CreateOrderRequest("item-1", 1, BigDecimal.TEN));

        // Ждём завершения асинхронной обработки с помощью Awaitility
        await()
                .atMost(Duration.ofSeconds(10))
                .pollInterval(Duration.ofMillis(500))
                .untilAsserted(() -> {
                    var order = orderRepository.findById(orderId).orElseThrow();
                    assertThat(order.getStatus()).isEqualTo(OrderStatus.PROCESSED);
                });
    }
}
```

---

## Связь с тестированием

Spring Boot Test — основной инструмент QA-автоматизатора в Java-проектах:

- **@WebMvcTest** — для тестирования REST API контроллеров (валидация, error handling, контракты).
- **@DataJpaTest** — для проверки SQL-запросов и работы с данными.
- **@SpringBootTest** — для полных интеграционных тестов сервиса.
- **Testcontainers** — для реалистичного окружения в тестах.
- **@MockBean/@SpyBean** — для изоляции от внешних зависимостей.

QA-инженер использует эти инструменты для автоматизации регрессионного тестирования,
проверки бизнес-требований и обеспечения качества API.

---

## Типичные ошибки

1. **Использование @SpringBootTest везде** — поднятие полного контекста для тестирования
   одного контроллера. Используйте test slices (`@WebMvcTest`, `@DataJpaTest`).

2. **Забывают про @AutoConfigureTestDatabase(replace = NONE)** — `@DataJpaTest` по умолчанию
   подменяет БД на H2, что может скрывать ошибки, специфичные для PostgreSQL/MySQL.

3. **Не очищают данные между тестами** — тесты зависят друг от друга.
   Используйте `@Transactional` (автоматический откат) или `@BeforeEach` с очисткой.

4. **Чрезмерное использование @MockBean** — мокируют слишком много, тест не проверяет
   реальное поведение. Каждый `@MockBean` инвалидирует кэш контекста.

5. **Тестирование фреймворка, а не логики** — проверяют, что Spring корректно
   инжектирует зависимости, вместо тестирования бизнес-сценариев.

6. **Отсутствие тестирования негативных сценариев** — проверяют только happy path,
   не тестируют 400, 404, 500, таймауты, невалидные данные.

7. **Hardcoded порты в тестах** — используют фиксированные порты вместо `RANDOM_PORT`,
   что вызывает конфликты при параллельном запуске тестов.

8. **Игнорирование test profiles** — используют production-конфигурацию в тестах,
   что приводит к подключению к реальным БД или отправке реальных email.

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Что делает аннотация `@SpringBootTest`?
2. Чем `@WebMvcTest` отличается от `@SpringBootTest`?
3. Что такое `MockMvc` и для чего он используется?
4. Что такое `@MockBean`?
5. Как запустить тест на случайном порту?

### 🟡 Средний уровень
6. Чем `@MockBean` отличается от `@SpyBean`? Когда использовать каждый?
7. Что такое test slices в Spring Boot? Перечислите 3-4 примера.
8. Как использовать `@DataJpaTest` с Testcontainers вместо H2?
9. Зачем нужен `@DynamicPropertySource`?
10. Как тестировать авторизацию в Spring Boot?
11. Чем `TestRestTemplate` отличается от `MockMvc`?

### 🔴 Продвинутый уровень
12. Как `@MockBean` влияет на кэширование Spring Context между тестами?
13. Как оптимизировать время запуска тестов в большом Spring Boot проекте?
14. Как тестировать асинхронные операции (`@Async`) в Spring Boot?
15. Как написать `@TestConfiguration`, которая заменяет бин только в определённых тестах?
16. Как организовать тесты для микросервиса с 10+ зависимостями?
17. Как тестировать Flyway-миграции с помощью `@DataJpaTest`?

---

## Практические задания

### Задание 1: @WebMvcTest для CRUD-контроллера
Напишите набор тестов для `ProductController` с эндпоинтами:
- `POST /api/products` — создание продукта (с валидацией).
- `GET /api/products/{id}` — получение по ID.
- `GET /api/products?category=ELECTRONICS` — фильтрация по категории.
- `PUT /api/products/{id}` — обновление.
- `DELETE /api/products/{id}` — удаление.
Покройте: happy path, валидацию, 404, авторизацию (ADMIN vs USER).

### Задание 2: @DataJpaTest с Testcontainers
Напишите тесты для `OrderRepository` с кастомными запросами:
- Поиск по статусу и дате.
- Агрегация (сумма заказов по клиенту).
- Пагинация.
Используйте PostgreSQL через Testcontainers.

### Задание 3: Полный component test
Напишите `@SpringBootTest` с `RANDOM_PORT` для Order Service:
- PostgreSQL (Testcontainers).
- WireMock для Payment Service.
- Проверьте создание заказа, оплату, обновление статуса.
- Проверьте поведение при недоступности Payment Service.

### Задание 4: Оптимизация тестов
Даны 20 тестовых классов, каждый из которых использует `@SpringBootTest` с разными `@MockBean`.
Время запуска: 5 минут. Задача: предложите стратегию оптимизации, чтобы сократить время до 1-2 минут.

---

## Дополнительные ресурсы

- **Документация**: [Spring Boot Testing](https://docs.spring.io/spring-boot/reference/testing/)
- **Документация**: [MockMvc](https://docs.spring.io/spring-framework/reference/testing/spring-mvc-test-framework.html)
- **Статья**: Baeldung — "Testing in Spring Boot"
- **Статья**: Baeldung — "Spring Boot @WebMvcTest"
- **Инструмент**: [Testcontainers](https://testcontainers.com/) — для реалистичных тестовых окружений
- **Библиотека**: [AssertJ](https://assertj.github.io/doc/) — fluent assertions для Java
- **Видео**: "Spring Boot Testing Best Practices" (Spring I/O conference)
