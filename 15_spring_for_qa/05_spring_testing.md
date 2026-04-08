# Тестирование Spring приложений

## Обзор

Spring предоставляет мощный инструментарий для тестирования на всех уровнях: от быстрых юнит-тестов
без контекста до полных интеграционных тестов с реальным HTTP-сервером. Для QA-инженера критически
важно понимать, когда какой подход применять, чем отличается `@SpringBootTest` от `@WebMvcTest`,
как использовать MockMvc, `@MockBean` и TestRestTemplate. Этот раздел — практическое руководство
по написанию тестов для Spring-приложений с точки зрения QA: что тестировать, какой инструмент
выбрать, и как не сделать тесты медленными и хрупкими.

---

## Обзор подходов к тестированию

### Пирамида тестов Spring-приложения

```
              ┌──────────────┐
              │  E2E-тесты   │  TestRestTemplate / WebClient
              │  (немного)   │  @SpringBootTest(RANDOM_PORT)
             ┌┴──────────────┴┐
             │  Slice-тесты   │  @WebMvcTest, @DataJpaTest
             │  (средне)      │  Один слой + моки
            ┌┴────────────────┴┐
            │  Юнит-тесты      │  Mockito, без Spring
            │  (много)         │  Максимально быстрые
            └──────────────────┘
```

### Сравнение подходов

| Подход | Аннотация | Что поднимает | Скорость | Когда использовать |
|--------|-----------|---------------|----------|-------------------|
| Юнит-тест | Без Spring | Ничего | Миллисекунды | Бизнес-логика в Service |
| Controller slice | `@WebMvcTest` | Web-слой | Секунды | Валидация, маршрутизация, HTTP-статусы |
| Repository slice | `@DataJpaTest` | JPA + H2 | Секунды | SQL-запросы, constraints |
| Full context | `@SpringBootTest(MOCK)` | Весь контекст | Десятки секунд | Интеграция между слоями |
| Full server | `@SpringBootTest(RANDOM_PORT)` | Контекст + HTTP-сервер | Десятки секунд | E2E через HTTP |

---

## @SpringBootTest — полный контекст

`@SpringBootTest` поднимает полный ApplicationContext — все bean, конфигурации, подключения.
Это самый «тяжёлый» вид теста, но он наиболее приближен к реальному поведению.

### Базовое использование

```java
// Поднимает весь контекст приложения
@SpringBootTest
class ApplicationContextTest {

    @Autowired
    private UserService userService;

    @Autowired
    private OrderService orderService;

    @Test
    void contextLoads() {
        // Если этот тест проходит — все bean успешно созданы,
        // все зависимости внедрены, конфигурация корректна
        assertNotNull(userService);
        assertNotNull(orderService);
    }
}
```

### WebEnvironment — режимы запуска

```java
// MOCK (по умолчанию) — без реального HTTP-сервера
// Используйте с MockMvc
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)
class ServiceIntegrationTest {

    @Autowired
    private MockMvc mockMvc; // Нужно добавить @AutoConfigureMockMvc

    @Test
    void shouldReturnUsers() throws Exception {
        mockMvc.perform(get("/api/users"))
                .andExpect(status().isOk());
    }
}

// RANDOM_PORT — реальный HTTP-сервер на случайном порту
// Используйте с TestRestTemplate
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ApiEndToEndTest {

    @LocalServerPort
    private int port;

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void shouldCreateAndRetrieveUser() {
        // Реальный HTTP-запрос к реальному серверу
        CreateUserRequest request = new CreateUserRequest("Иван", "ivan@mail.ru", "password");

        ResponseEntity<UserResponse> createResponse = restTemplate.postForEntity(
                "/api/users", request, UserResponse.class);

        assertEquals(HttpStatus.CREATED, createResponse.getStatusCode());
        assertNotNull(createResponse.getBody().id());

        // Получаем созданного пользователя
        Long userId = createResponse.getBody().id();
        ResponseEntity<UserResponse> getResponse = restTemplate.getForEntity(
                "/api/users/" + userId, UserResponse.class);

        assertEquals(HttpStatus.OK, getResponse.getStatusCode());
        assertEquals("Иван", getResponse.getBody().name());
    }
}
```

### Когда использовать @SpringBootTest

| Сценарий | Нужен @SpringBootTest? | Альтернатива |
|----------|----------------------|--------------|
| Проверить, что контекст поднимается | Да | — |
| Интеграция между слоями | Да | — |
| Тест одного Controller | Нет | `@WebMvcTest` |
| Тест SQL-запросов | Нет | `@DataJpaTest` |
| Тест бизнес-логики Service | Нет | Юнит-тест с Mockito |
| E2E через HTTP | Да (RANDOM_PORT) | — |

---

## @WebMvcTest — тестирование контроллеров

`@WebMvcTest` поднимает только web-слой: контроллер, фильтры, обработчики ошибок.
Service и Repository заменяются моками.

### Базовое использование

```java
@WebMvcTest(UserController.class) // Только UserController
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc; // Инструмент для HTTP-вызовов без сервера

    @MockBean // Заменяет реальный UserService на мок в контексте
    private UserService userService;

    @Test
    void shouldReturnAllUsers() throws Exception {
        // Arrange — настраиваем мок
        List<UserResponse> users = List.of(
                new UserResponse(1L, "Иван", "ivan@mail.ru", "ACTIVE", LocalDateTime.now()),
                new UserResponse(2L, "Мария", "maria@mail.ru", "ACTIVE", LocalDateTime.now())
        );
        when(userService.getAllUsers()).thenReturn(users);

        // Act & Assert — выполняем запрос и проверяем
        mockMvc.perform(get("/api/users"))
                .andExpect(status().isOk())
                .andExpect(content().contentType(MediaType.APPLICATION_JSON))
                .andExpect(jsonPath("$").isArray())
                .andExpect(jsonPath("$.length()").value(2))
                .andExpect(jsonPath("$[0].name").value("Иван"))
                .andExpect(jsonPath("$[1].email").value("maria@mail.ru"));
    }

    @Test
    void shouldReturnUserById() throws Exception {
        UserResponse user = new UserResponse(1L, "Иван", "ivan@mail.ru", "ACTIVE", LocalDateTime.now());
        when(userService.getUserById(1L)).thenReturn(user);

        mockMvc.perform(get("/api/users/{id}", 1L))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.id").value(1))
                .andExpect(jsonPath("$.name").value("Иван"));
    }

    @Test
    void shouldReturn404WhenUserNotFound() throws Exception {
        when(userService.getUserById(999L))
                .thenThrow(new UserNotFoundException("Пользователь не найден"));

        mockMvc.perform(get("/api/users/{id}", 999L))
                .andExpect(status().isNotFound())
                .andExpect(jsonPath("$.message").value("Пользователь не найден"));
    }
}
```

### Тестирование POST с валидацией

```java
@WebMvcTest(UserController.class)
class UserControllerValidationTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    @Test
    void shouldCreateUserWithValidData() throws Exception {
        String validJson = """
                {
                    "name": "Иван Петров",
                    "email": "ivan@mail.ru",
                    "password": "securePassword123"
                }
                """;

        UserResponse response = new UserResponse(1L, "Иван Петров", "ivan@mail.ru",
                "ACTIVE", LocalDateTime.now());
        when(userService.createUser(any())).thenReturn(response);

        mockMvc.perform(post("/api/users")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(validJson))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.id").value(1))
                .andExpect(jsonPath("$.name").value("Иван Петров"));
    }

    @Test
    void shouldReturn400ForEmptyName() throws Exception {
        String invalidJson = """
                {
                    "name": "",
                    "email": "ivan@mail.ru",
                    "password": "securePassword123"
                }
                """;

        mockMvc.perform(post("/api/users")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(invalidJson))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.errors[?(@.field == 'name')]").exists());
    }

    @Test
    void shouldReturn400ForInvalidEmail() throws Exception {
        String invalidJson = """
                {
                    "name": "Иван",
                    "email": "not-an-email",
                    "password": "securePassword123"
                }
                """;

        mockMvc.perform(post("/api/users")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(invalidJson))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.errors[?(@.field == 'email')]").exists());
    }

    @Test
    void shouldReturn400ForShortPassword() throws Exception {
        String invalidJson = """
                {
                    "name": "Иван",
                    "email": "ivan@mail.ru",
                    "password": "123"
                }
                """;

        mockMvc.perform(post("/api/users")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(invalidJson))
                .andExpect(status().isBadRequest());
    }

    @Test
    void shouldReturn400ForEmptyBody() throws Exception {
        mockMvc.perform(post("/api/users")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{}"))
                .andExpect(status().isBadRequest());
    }

    @Test
    void shouldReturn400ForMalformedJson() throws Exception {
        mockMvc.perform(post("/api/users")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{invalid json}"))
                .andExpect(status().isBadRequest());
    }
}
```

### Продвинутые возможности MockMvc

```java
@Test
void shouldSupportPaginationParams() throws Exception {
    // Тестирование query-параметров
    mockMvc.perform(get("/api/users")
                    .param("page", "0")
                    .param("size", "10")
                    .param("sort", "name,asc"))
            .andExpect(status().isOk());
}

@Test
void shouldSupportAuthorizationHeader() throws Exception {
    // Тестирование заголовков
    mockMvc.perform(get("/api/users")
                    .header("Authorization", "Bearer test-token")
                    .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk());
}

@Test
void shouldReturnProblemDetailsOnError() throws Exception {
    // Подробная проверка тела ответа с выводом в консоль
    mockMvc.perform(get("/api/users/999"))
            .andDo(print()) // Выводит запрос и ответ в консоль — полезно при отладке
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.code").value("NOT_FOUND"))
            .andExpect(jsonPath("$.message").isString());
}
```

---

## @DataJpaTest — тестирование репозиториев

`@DataJpaTest` поднимает только JPA-слой: entity, repository, in-memory БД (H2 по умолчанию).
Каждый тест оборачивается в транзакцию и откатывается после выполнения.

### Базовое использование

```java
@DataJpaTest
class UserRepositoryTest {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private TestEntityManager entityManager; // Низкоуровневый доступ к JPA

    @Test
    void shouldSaveAndFindUser() {
        // Arrange — создаём entity
        User user = new User(null, "Иван", "ivan@mail.ru", UserStatus.ACTIVE, LocalDateTime.now());

        // Act — сохраняем и читаем
        User saved = userRepository.save(user);
        entityManager.flush(); // Принудительно отправляем SQL в БД
        entityManager.clear(); // Очищаем кэш первого уровня

        Optional<User> found = userRepository.findById(saved.getId());

        // Assert
        assertTrue(found.isPresent());
        assertEquals("Иван", found.get().getName());
        assertEquals("ivan@mail.ru", found.get().getEmail());
    }

    @Test
    void shouldFindByEmail() {
        // Arrange — используем TestEntityManager для подготовки данных
        User user = new User(null, "Мария", "maria@mail.ru", UserStatus.ACTIVE, LocalDateTime.now());
        entityManager.persistAndFlush(user);
        entityManager.clear();

        // Act
        Optional<User> found = userRepository.findByEmail("maria@mail.ru");

        // Assert
        assertTrue(found.isPresent());
        assertEquals("Мария", found.get().getName());
    }

    @Test
    void shouldReturnEmptyForNonExistentEmail() {
        Optional<User> found = userRepository.findByEmail("nobody@mail.ru");
        assertTrue(found.isEmpty());
    }

    @Test
    void shouldEnforceUniqueEmailConstraint() {
        User user1 = new User(null, "Иван", "same@mail.ru", UserStatus.ACTIVE, LocalDateTime.now());
        User user2 = new User(null, "Мария", "same@mail.ru", UserStatus.ACTIVE, LocalDateTime.now());

        userRepository.save(user1);
        entityManager.flush();

        // Дупликат email — нарушение unique-constraint
        assertThrows(DataIntegrityViolationException.class, () -> {
            userRepository.save(user2);
            entityManager.flush();
        });
    }
}
```

### Тестирование кастомных запросов

```java
@DataJpaTest
class ProductRepositoryTest {

    @Autowired
    private ProductRepository productRepository;

    @Autowired
    private TestEntityManager entityManager;

    @BeforeEach
    void setUp() {
        // Подготовка тестовых данных
        entityManager.persist(new Product(null, "Ноутбук", new BigDecimal("50000"), "electronics"));
        entityManager.persist(new Product(null, "Мышь", new BigDecimal("1500"), "electronics"));
        entityManager.persist(new Product(null, "Книга", new BigDecimal("800"), "books"));
        entityManager.flush();
        entityManager.clear();
    }

    @Test
    void shouldFindByCategory() {
        List<Product> electronics = productRepository.findByCategory("electronics");

        assertEquals(2, electronics.size());
        assertTrue(electronics.stream().allMatch(p -> "electronics".equals(p.getCategory())));
    }

    @Test
    void shouldFindByPriceRange() {
        List<Product> affordable = productRepository.findByPriceRange(
                new BigDecimal("500"), new BigDecimal("2000"));

        assertEquals(2, affordable.size()); // Мышь (1500) и Книга (800)
    }

    @Test
    void shouldReturnEmptyListForNonExistentCategory() {
        List<Product> toys = productRepository.findByCategory("toys");
        assertTrue(toys.isEmpty());
    }

    @Test
    void shouldSupportPagination() {
        Page<Product> firstPage = productRepository.findAll(PageRequest.of(0, 2));

        assertEquals(2, firstPage.getContent().size());
        assertEquals(3, firstPage.getTotalElements());
        assertEquals(2, firstPage.getTotalPages());
    }
}
```

### @DataJpaTest с Testcontainers

```java
// Тестирование с реальной БД вместо H2
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE) // Не заменять на H2
@Testcontainers
class UserRepositoryPostgresTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("testdb");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private UserRepository userRepository;

    @Test
    void shouldWorkWithRealPostgres() {
        User user = new User(null, "Иван", "ivan@mail.ru", UserStatus.ACTIVE, LocalDateTime.now());
        User saved = userRepository.save(user);

        assertNotNull(saved.getId());
    }
}
```

> **Для QA**: H2 и PostgreSQL ведут себя по-разному (регистрозависимость, типы данных,
> функции). Если тест на H2 проходит, но на production PostgreSQL — нет, используйте
> Testcontainers для тестирования с реальной СУБД.

---

## MockMvc — детальное тестирование HTTP

MockMvc позволяет тестировать HTTP-взаимодействие без запуска реального сервера.

### Основные методы MockMvc

```java
@WebMvcTest(OrderController.class)
class OrderControllerMockMvcTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private OrderService orderService;

    // ============ Проверка HTTP-статусов ============

    @Test
    void shouldReturn200ForExistingOrder() throws Exception {
        when(orderService.getOrderById(1L)).thenReturn(new OrderResponse(1L, "COMPLETED", BigDecimal.TEN));

        mockMvc.perform(get("/api/orders/1"))
                .andExpect(status().isOk());
    }

    @Test
    void shouldReturn201ForCreatedOrder() throws Exception {
        when(orderService.createOrder(any())).thenReturn(new OrderResponse(1L, "NEW", BigDecimal.TEN));

        mockMvc.perform(post("/api/orders")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("""
                                {"productId": "prod-1", "quantity": 2}
                                """))
                .andExpect(status().isCreated());
    }

    @Test
    void shouldReturn204ForDeletedOrder() throws Exception {
        doNothing().when(orderService).deleteOrder(1L);

        mockMvc.perform(delete("/api/orders/1"))
                .andExpect(status().isNoContent());
    }

    // ============ Проверка JSON-ответа ============

    @Test
    void shouldReturnCorrectJsonStructure() throws Exception {
        OrderResponse order = new OrderResponse(42L, "PROCESSING", new BigDecimal("199.99"));
        when(orderService.getOrderById(42L)).thenReturn(order);

        mockMvc.perform(get("/api/orders/42"))
                .andExpect(status().isOk())
                .andExpect(content().contentType(MediaType.APPLICATION_JSON))
                .andExpect(jsonPath("$.id").value(42))
                .andExpect(jsonPath("$.status").value("PROCESSING"))
                .andExpect(jsonPath("$.total").value(199.99))
                // Проверяем отсутствие лишних полей
                .andExpect(jsonPath("$.internalNotes").doesNotExist())
                .andExpect(jsonPath("$.password").doesNotExist());
    }

    // ============ Проверка заголовков ============

    @Test
    void shouldReturnCorrectHeaders() throws Exception {
        when(orderService.getOrderById(1L)).thenReturn(new OrderResponse(1L, "NEW", BigDecimal.TEN));

        mockMvc.perform(get("/api/orders/1"))
                .andExpect(header().string("Content-Type", "application/json"))
                .andExpect(header().doesNotExist("X-Internal-Header"));
    }

    // ============ Тестирование списка ============

    @Test
    void shouldReturnListOfOrders() throws Exception {
        List<OrderResponse> orders = List.of(
                new OrderResponse(1L, "NEW", BigDecimal.TEN),
                new OrderResponse(2L, "COMPLETED", BigDecimal.valueOf(50))
        );
        when(orderService.getAllOrders()).thenReturn(orders);

        mockMvc.perform(get("/api/orders"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$").isArray())
                .andExpect(jsonPath("$.length()").value(2))
                .andExpect(jsonPath("$[0].id").value(1))
                .andExpect(jsonPath("$[1].status").value("COMPLETED"));
    }
}
```

### JSONPath — навигация по JSON

| Выражение | Описание | Пример |
|-----------|----------|--------|
| `$.field` | Поле объекта | `$.name` → `"Иван"` |
| `$[0]` | Элемент массива по индексу | `$[0].id` → `1` |
| `$.length()` | Длина массива | `$.length()` → `3` |
| `$[*].field` | Поле каждого элемента | `$[*].name` → все имена |
| `$[?(@.age > 18)]` | Фильтрация | Элементы с age > 18 |
| `$.field.nested` | Вложенное поле | `$.address.city` |

---

## TestRestTemplate — тестирование через HTTP

TestRestTemplate отправляет реальные HTTP-запросы к реальному серверу. Это наиболее
реалистичный вид тестирования.

### Использование с RANDOM_PORT

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserApiEndToEndTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void fullUserLifecycle() {
        // 1. Создание пользователя
        CreateUserRequest createRequest = new CreateUserRequest("Иван", "ivan@mail.ru", "password");
        ResponseEntity<UserResponse> createResponse = restTemplate.postForEntity(
                "/api/users", createRequest, UserResponse.class);

        assertEquals(HttpStatus.CREATED, createResponse.getStatusCode());
        Long userId = createResponse.getBody().id();
        assertNotNull(userId);

        // 2. Получение пользователя
        ResponseEntity<UserResponse> getResponse = restTemplate.getForEntity(
                "/api/users/" + userId, UserResponse.class);

        assertEquals(HttpStatus.OK, getResponse.getStatusCode());
        assertEquals("Иван", getResponse.getBody().name());

        // 3. Обновление пользователя
        UpdateUserRequest updateRequest = new UpdateUserRequest("Иван Петров", null);
        restTemplate.put("/api/users/" + userId, updateRequest);

        ResponseEntity<UserResponse> updatedResponse = restTemplate.getForEntity(
                "/api/users/" + userId, UserResponse.class);
        assertEquals("Иван Петров", updatedResponse.getBody().name());

        // 4. Удаление пользователя
        restTemplate.delete("/api/users/" + userId);

        ResponseEntity<String> deletedResponse = restTemplate.getForEntity(
                "/api/users/" + userId, String.class);
        assertEquals(HttpStatus.NOT_FOUND, deletedResponse.getStatusCode());
    }

    @Test
    void shouldReturn400ForInvalidInput() {
        // Невалидный запрос
        CreateUserRequest invalidRequest = new CreateUserRequest("", "not-email", "123");

        ResponseEntity<String> response = restTemplate.postForEntity(
                "/api/users", invalidRequest, String.class);

        assertEquals(HttpStatus.BAD_REQUEST, response.getStatusCode());
    }
}
```

### MockMvc vs TestRestTemplate

| Аспект | MockMvc | TestRestTemplate |
|--------|---------|-----------------|
| HTTP-сервер | Не нужен (mock) | Нужен (реальный) |
| Скорость | Быстрее | Медленнее |
| Реалистичность | Средняя | Высокая |
| Фильтры, Security | Можно включить/отключить | Всегда работают |
| Сериализация | Проверяется частично | Полный цикл JSON ↔ Object |
| Использование | `@WebMvcTest` или `@SpringBootTest(MOCK)` | `@SpringBootTest(RANDOM_PORT)` |
| Для QA | Тесты контроллеров | E2E-тесты API |

---

## @MockBean — подмена bean в контексте

`@MockBean` заменяет реальный bean на Mockito-мок в ApplicationContext.

### Использование @MockBean

```java
@SpringBootTest
class OrderServiceIntegrationTest {

    // Реальный OrderService из контекста
    @Autowired
    private OrderService orderService;

    // PaymentService заменён на мок — не вызываем реальный платёжный API
    @MockBean
    private PaymentService paymentService;

    // EmailService заменён на мок — не отправляем реальные письма
    @MockBean
    private EmailService emailService;

    @Test
    void shouldCreateOrderWithMockedPayment() {
        // Настраиваем мок: платёж всегда успешен
        when(paymentService.charge(any(BigDecimal.class)))
                .thenReturn(new PaymentResult(true, "txn-123"));

        // Вызываем реальный OrderService, который использует мок PaymentService
        OrderResponse order = orderService.createOrder(
                new CreateOrderRequest("product-1", 2));

        assertNotNull(order.id());
        assertEquals("NEW", order.status());

        // Проверяем, что платёжный сервис был вызван
        verify(paymentService).charge(any(BigDecimal.class));
        // Проверяем, что email был отправлен
        verify(emailService).sendOrderConfirmation(any());
    }

    @Test
    void shouldHandlePaymentFailure() {
        // Настраиваем мок: платёж не прошёл
        when(paymentService.charge(any(BigDecimal.class)))
                .thenReturn(new PaymentResult(false, null));

        // Ожидаем исключение
        assertThrows(PaymentException.class, () ->
                orderService.createOrder(new CreateOrderRequest("product-1", 2)));

        // Email НЕ должен быть отправлен при ошибке оплаты
        verify(emailService, never()).sendOrderConfirmation(any());
    }
}
```

### @MockBean vs @Mock

| Аспект | @MockBean | @Mock (Mockito) |
|--------|-----------|-----------------|
| Контекст | Заменяет bean в ApplicationContext | Создаёт независимый мок |
| Spring | Требует Spring-контекст | Не требует Spring |
| Скорость | Медленнее (пересоздание контекста) | Быстрее |
| Использование | `@SpringBootTest`, `@WebMvcTest` | `@ExtendWith(MockitoExtension.class)` |
| Когда | Интеграционные тесты | Юнит-тесты |

> **Для QA**: каждый уникальный набор `@MockBean` создаёт новый контекст. Если у вас
> 10 тестовых классов с разными `@MockBean` — Spring создаст 10 контекстов. Это замедляет
> тесты. Старайтесь унифицировать набор моков.

---

## Тестовые properties и профили

### Переопределение настроек в тестах

```java
// Способ 1: @ActiveProfiles
@SpringBootTest
@ActiveProfiles("test") // Использует application-test.yml
class TestWithProfile {
}

// Способ 2: @TestPropertySource
@SpringBootTest
@TestPropertySource(properties = {
        "notification.email.enabled=false",
        "payment.api.url=http://localhost:8089/mock-payment"
})
class TestWithProperties {
}

// Способ 3: properties в @SpringBootTest
@SpringBootTest(properties = {
        "spring.datasource.url=jdbc:h2:mem:testdb",
        "spring.jpa.hibernate.ddl-auto=create-drop"
})
class TestWithInlineProperties {
}

// Способ 4: @DynamicPropertySource (для Testcontainers)
@SpringBootTest
@Testcontainers
class TestWithDynamicProperties {

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
            .withExposedPorts(6379);

    @DynamicPropertySource
    static void configureRedis(DynamicPropertyRegistry registry) {
        registry.add("spring.redis.host", redis::getHost);
        registry.add("spring.redis.port", () -> redis.getMappedPort(6379));
    }
}
```

### Структура тестовых ресурсов

```
src/test/resources/
├── application-test.yml           # Тестовый профиль
├── application-integration.yml    # Интеграционные тесты
├── data.sql                       # Тестовые данные для @DataJpaTest
└── schema.sql                     # Схема для H2
```

---

## Когда QA пишет Spring-тесты

### Уровни тестирования и ответственность

```
┌───────────────────────────────────────────────────┐
│ Уровень         │ Кто пишет    │ Инструменты      │
├───────────────────────────────────────────────────┤
│ Юнит-тесты      │ Разработчик  │ Mockito, JUnit   │
│ Slice-тесты     │ Разработчик  │ @WebMvcTest и т.д│
│ API-тесты       │ QA / Dev     │ MockMvc, REST    │
│ Интеграционные  │ QA           │ @SpringBootTest  │
│ E2E-тесты       │ QA           │ TestRestTemplate │
│ Внешние E2E     │ QA           │ RestAssured      │
└───────────────────────────────────────────────────┘
```

### Типичные задачи QA в Spring-тестах

1. **Проверка API-контракта** — ответы соответствуют документации
2. **Негативное тестирование** — невалидные данные, граничные значения
3. **Интеграционные сценарии** — полный цикл create → read → update → delete
4. **Тесты с тестовыми данными** — проверка на конкретных наборах данных
5. **Тесты ошибок** — поведение при недоступности зависимостей

---

## Связь с тестированием

### Полный пример: QA-тесты для REST API

```java
// Комплексный набор тестов для API пользователей
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class UserApiQATest {

    @Autowired
    private TestRestTemplate restTemplate;

    @MockBean
    private EmailService emailService; // Не отправляем реальные письма

    private static Long createdUserId;

    @Test
    @Order(1)
    void shouldCreateUser() {
        CreateUserRequest request = new CreateUserRequest("Тестовый Юзер", "test@mail.ru", "password123");
        ResponseEntity<UserResponse> response = restTemplate.postForEntity(
                "/api/users", request, UserResponse.class);

        assertAll("Создание пользователя",
                () -> assertEquals(HttpStatus.CREATED, response.getStatusCode()),
                () -> assertNotNull(response.getBody()),
                () -> assertNotNull(response.getBody().id()),
                () -> assertEquals("Тестовый Юзер", response.getBody().name()),
                () -> assertEquals("test@mail.ru", response.getBody().email()),
                () -> assertEquals("ACTIVE", response.getBody().status())
        );

        createdUserId = response.getBody().id();
    }

    @Test
    @Order(2)
    void shouldGetCreatedUser() {
        ResponseEntity<UserResponse> response = restTemplate.getForEntity(
                "/api/users/" + createdUserId, UserResponse.class);

        assertEquals(HttpStatus.OK, response.getStatusCode());
        assertEquals("Тестовый Юзер", response.getBody().name());
    }

    @Test
    @Order(3)
    void shouldNotCreateDuplicateEmail() {
        CreateUserRequest duplicate = new CreateUserRequest("Другой", "test@mail.ru", "password");
        ResponseEntity<String> response = restTemplate.postForEntity(
                "/api/users", duplicate, String.class);

        assertEquals(HttpStatus.CONFLICT, response.getStatusCode());
    }

    @Test
    @Order(4)
    void shouldRejectInvalidInput() {
        // Пустое имя
        CreateUserRequest noName = new CreateUserRequest("", "valid@mail.ru", "password123");
        assertEquals(HttpStatus.BAD_REQUEST,
                restTemplate.postForEntity("/api/users", noName, String.class).getStatusCode());

        // Невалидный email
        CreateUserRequest badEmail = new CreateUserRequest("Name", "not-email", "password123");
        assertEquals(HttpStatus.BAD_REQUEST,
                restTemplate.postForEntity("/api/users", badEmail, String.class).getStatusCode());

        // Короткий пароль
        CreateUserRequest shortPwd = new CreateUserRequest("Name", "a@b.ru", "123");
        assertEquals(HttpStatus.BAD_REQUEST,
                restTemplate.postForEntity("/api/users", shortPwd, String.class).getStatusCode());
    }

    @Test
    @Order(5)
    void shouldReturn404ForNonExistentUser() {
        ResponseEntity<String> response = restTemplate.getForEntity(
                "/api/users/999999", String.class);
        assertEquals(HttpStatus.NOT_FOUND, response.getStatusCode());
    }
}
```

---

## Типичные ошибки

### 1. @SpringBootTest для всех тестов

```java
// ОШИБКА: полный контекст для простого юнит-теста — медленно
@SpringBootTest
class CalculatorServiceTest {
    @Autowired
    private CalculatorService service;

    @Test
    void shouldAdd() {
        assertEquals(4, service.add(2, 2)); // Для этого не нужен Spring!
    }
}

// ПРАВИЛЬНО: простой юнит-тест
class CalculatorServiceTest {
    private final CalculatorService service = new CalculatorService();

    @Test
    void shouldAdd() {
        assertEquals(4, service.add(2, 2));
    }
}
```

### 2. Разные @MockBean в каждом тесте

```java
// ОШИБКА: каждый класс создаёт новый контекст (3 разных контекста!)
@SpringBootTest
class Test1 {
    @MockBean private ServiceA serviceA; // Контекст 1
}

@SpringBootTest
class Test2 {
    @MockBean private ServiceB serviceB; // Контекст 2
}

@SpringBootTest
class Test3 {
    @MockBean private ServiceA serviceA;
    @MockBean private ServiceB serviceB; // Контекст 3
}

// ПРАВИЛЬНО: базовый класс с общими моками
@SpringBootTest
abstract class BaseIntegrationTest {
    @MockBean protected ServiceA serviceA;
    @MockBean protected ServiceB serviceB;
}

class Test1 extends BaseIntegrationTest { }
class Test2 extends BaseIntegrationTest { }
class Test3 extends BaseIntegrationTest { }
// Один общий контекст для всех!
```

### 3. Забыли flush/clear в @DataJpaTest

```java
@DataJpaTest
class UserRepositoryTest {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private TestEntityManager entityManager;

    @Test
    void shouldFindByEmail() {
        User user = new User(null, "Иван", "ivan@mail.ru", UserStatus.ACTIVE, LocalDateTime.now());
        userRepository.save(user);
        // ОШИБКА: без flush и clear Hibernate может вернуть из кэша,
        // не проверяя реальный SQL-запрос
        // entityManager.flush();
        // entityManager.clear();

        Optional<User> found = userRepository.findByEmail("ivan@mail.ru");
        assertTrue(found.isPresent()); // Может пройти из кэша, скрыв баг в SQL
    }
}
```

### 4. Жёсткая зависимость от порядка тестов

```java
// ОШИБКА: тесты зависят от порядка выполнения
@SpringBootTest
class OrderedTests {

    static Long userId; // Состояние между тестами

    @Test
    void step1_createUser() {
        // Создаёт пользователя, сохраняет ID
        userId = createUser();
    }

    @Test
    void step2_updateUser() {
        // Падает, если step1 не выполнился первым!
        updateUser(userId);
    }
}

// ПРАВИЛЬНО: каждый тест самодостаточен
@SpringBootTest
class IndependentTests {

    @Test
    void shouldUpdateUser() {
        // Создаём данные внутри теста
        Long userId = createUser();
        updateUser(userId);
        // Проверяем результат
    }
}
```

### 5. Тестирование через @SpringBootTest(MOCK) с ожиданием реального HTTP

```java
// ОШИБКА: MOCK не запускает реальный сервер — TestRestTemplate не работает
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)
class ApiTest {

    @Autowired
    private TestRestTemplate restTemplate; // Не будет работать без реального сервера!

    @Test
    void test() {
        restTemplate.getForEntity("/api/users", String.class); // Ошибка!
    }
}

// ПРАВИЛЬНО: RANDOM_PORT для TestRestTemplate
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ApiTest {

    @Autowired
    private TestRestTemplate restTemplate; // Работает — есть реальный сервер

    @Test
    void test() {
        restTemplate.getForEntity("/api/users", String.class); // OK
    }
}
```

---

## Вопросы на интервью

### 🟢 Базовый уровень

1. **Что делает @SpringBootTest?**
   Поднимает полный ApplicationContext Spring Boot для интеграционных тестов. Создаёт все
   bean, конфигурации и подключения.

2. **В чём разница между @WebMvcTest и @SpringBootTest?**
   `@WebMvcTest` поднимает только web-слой (Controller), все Service и Repository нужно
   мокать. `@SpringBootTest` поднимает весь контекст.

3. **Что такое MockMvc?**
   Инструмент для тестирования HTTP-запросов без реального сервера. Позволяет проверять
   статусы, JSON-ответы, заголовки.

4. **Что делает @MockBean?**
   Заменяет реальный bean в ApplicationContext на Mockito-мок. Используется для изоляции
   тестируемого компонента от зависимостей.

### 🟡 Средний уровень

5. **Когда использовать MockMvc, а когда TestRestTemplate?**
   MockMvc — для тестов контроллера без HTTP-сервера (быстрее, проще). TestRestTemplate —
   для E2E-тестов через реальный HTTP (реалистичнее, включает фильтры и Security).

6. **Что делает @DataJpaTest?**
   Поднимает только JPA-слой: Entity, Repository, in-memory БД. Каждый тест
   оборачивается в транзакцию и откатывается. Не создаёт Controller и Service.

7. **Почему разные @MockBean замедляют тесты?**
   Каждый уникальный набор `@MockBean` создаёт новый ApplicationContext. Spring не может
   переиспользовать контекст, если набор моков отличается.

8. **Как подготовить тестовые данные для @DataJpaTest?**
   Через `TestEntityManager`, файлы `data.sql`, или `@BeforeEach` с вызовами Repository.

### 🔴 Продвинутый уровень

9. **Как работает кэширование контекста в Spring Test?**
   Spring кэширует ApplicationContext по ключу (конфигурация + профили + properties + MockBean).
   Если ключ совпадает — контекст переиспользуется. Разные @MockBean или @ActiveProfiles
   создают разные ключи и, соответственно, разные контексты.

10. **Как тестировать транзакционность с @DataJpaTest?**
    `@DataJpaTest` оборачивает каждый тест в транзакцию и откатывает после. Для проверки
    реального commit нужно `@Commit` или `@Rollback(false)`. Для проверки отката —
    создать сценарий с исключением посередине.

11. **Как организовать тестовую инфраструктуру для команды?**
    Базовый класс с общими `@MockBean`, shared Testcontainers (reusable = true),
    унифицированные профили, фабрики тестовых данных (Object Mother / Test Data Builder).

---

## Практические задания

### Задание 1: Выберите подход

Для каждого сценария выберите оптимальный подход тестирования:
1. Проверить валидацию поля email в CreateUserRequest
2. Проверить, что метод `findByStatus()` возвращает правильных пользователей
3. Проверить бизнес-логику расчёта скидки в OrderService
4. Проверить полный цикл: создание заказа → оплата → подтверждение по email
5. Проверить, что при ошибке оплаты заказ не сохраняется в БД

### Задание 2: Напишите @WebMvcTest

Для контроллера ниже напишите тесты на все эндпоинты с позитивными и негативными сценариями:

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {
    @GetMapping
    public List<ProductResponse> getAll(@RequestParam(required = false) String category) { ... }

    @GetMapping("/{id}")
    public ProductResponse getById(@PathVariable Long id) { ... }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ProductResponse create(@Valid @RequestBody CreateProductRequest request) { ... }
}
```

### Задание 3: Напишите @DataJpaTest

Для Repository ниже напишите тесты, включая граничные случаи:

```java
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByUserId(Long userId);
    List<Order> findByStatus(OrderStatus status);
    Optional<Order> findByOrderNumber(String orderNumber);

    @Query("SELECT o FROM Order o WHERE o.createdAt >= :since AND o.status = :status")
    List<Order> findRecentByStatus(@Param("since") LocalDateTime since,
                                    @Param("status") OrderStatus status);
}
```

### Задание 4: Оптимизируйте тесты

Предложите, как ускорить набор из 50 интеграционных тестов, каждый из которых использует
`@SpringBootTest` с разными комбинациями `@MockBean`.

---

## Дополнительные ресурсы

- [Spring Boot Testing Reference](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.testing) — официальная документация
- [Baeldung: Testing in Spring Boot](https://www.baeldung.com/spring-boot-testing) — практическое руководство
- [MockMvc Documentation](https://docs.spring.io/spring-framework/reference/testing/spring-mvc-test-framework.html) — MockMvc в деталях
- [Testcontainers Spring Boot](https://testcontainers.com/guides/testing-spring-boot-rest-api-using-testcontainers/) — Testcontainers + Spring
- [Baeldung: @MockBean vs @Mock](https://www.baeldung.com/java-spring-mockbean-vs-mock) — разница между подходами
