# Управление тестовыми данными

## Обзор

Управление тестовыми данными (Test Data Management) — одна из ключевых задач QA-инженера. Качество тестовых данных напрямую влияет на надёжность и воспроизводимость тестов. В этом разделе рассматриваются подходы к генерации, организации и изоляции тестовых данных в Java-проектах: от простых библиотек генерации до архитектурных паттернов.

**Основные зависимости (Maven):**

```xml
<!-- Datafaker — генерация реалистичных данных -->
<dependency>
    <groupId>net.datafaker</groupId>
    <artifactId>datafaker</artifactId>
    <version>2.1.0</version>
    <scope>test</scope>
</dependency>

<!-- Lombok — @Builder для тестовых объектов -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.30</version>
    <scope>provided</scope>
</dependency>

<!-- Jackson — работа с JSON тестовыми данными -->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.17.0</version>
    <scope>test</scope>
</dependency>
```

---

## JavaFaker / Datafaker — генерация тестовых данных

JavaFaker — классическая библиотека, Datafaker — её активно развивающийся форк с поддержкой Java 17+ и расширенными провайдерами.

### Базовое использование Datafaker

```java
import net.datafaker.Faker;

class DataFakerExamples {

    private static final Faker faker = new Faker();

    @Test
    void generateBasicData() {
        // Персональные данные
        String firstName = faker.name().firstName();        // "Иван"
        String lastName = faker.name().lastName();           // "Петров"
        String fullName = faker.name().fullName();           // "Иван Петров"

        // Контактная информация
        String email = faker.internet().emailAddress();      // "ivan.petrov@gmail.com"
        String phone = faker.phoneNumber().cellPhone();      // "+7-999-123-4567"

        // Адрес
        String city = faker.address().city();                // "Москва"
        String street = faker.address().streetAddress();     // "ул. Ленина, 42"
        String zipCode = faker.address().zipCode();          // "101000"

        // Текст
        String paragraph = faker.lorem().paragraph();
        String sentence = faker.lorem().sentence(10);
    }

    @Test
    void generateBusinessData() {
        // Финансы
        String cardNumber = faker.finance().creditCard();    // "4111-1111-1111-1111"
        String iban = faker.finance().iban();

        // Компания
        String company = faker.company().name();             // "ООО Рога и Копыта"
        String industry = faker.company().industry();

        // Продукты
        String productName = faker.commerce().productName(); // "Ergonomic Steel Chair"
        String price = faker.commerce().price();             // "29.99"
    }
}
```

### Генерация уникальных данных для тестов

```java
class UniqueTestDataGenerator {

    private static final Faker faker = new Faker();

    /**
     * Генерирует уникальный email с временной меткой —
     * гарантия отсутствия дубликатов при параллельном запуске
     */
    public static String uniqueEmail() {
        return String.format("test.%s.%d@example.com",
                faker.internet().username(),
                System.nanoTime());
    }

    /**
     * Генерирует уникальное имя пользователя
     */
    public static String uniqueUsername() {
        return faker.internet().username() + "_" + System.currentTimeMillis();
    }

    /**
     * Генерирует реалистичный пароль, соответствующий требованиям безопасности
     */
    public static String securePassword() {
        return faker.internet().password(
                10,     // минимальная длина
                20,     // максимальная длина
                true,   // верхний регистр
                true,   // спецсимволы
                true    // цифры
        );
    }

    @Test
    void shouldRegisterWithUniqueData() {
        String email = uniqueEmail();
        String username = uniqueUsername();
        String password = securePassword();

        RegistrationRequest request = new RegistrationRequest(username, email, password);
        RegistrationResult result = authService.register(request);

        assertThat(result.isSuccess()).isTrue();
    }
}
```

### Локализация

```java
// Русская локализация
Faker ruFaker = new Faker(new Locale("ru"));
String russianName = ruFaker.name().fullName();    // "Иванов Пётр Сергеевич"
String russianCity = ruFaker.address().city();      // "Санкт-Петербург"

// Английская локализация
Faker enFaker = new Faker(new Locale("en"));
String englishName = enFaker.name().fullName();     // "John Smith"
```

---

## Builder-паттерн для тестовых объектов

### С Lombok @Builder

```java
// Доменный объект
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class User {
    private Long id;
    private String firstName;
    private String lastName;
    private String email;
    private int age;
    private String role;
    private boolean active;
    private LocalDateTime createdAt;
}

// Использование в тестах — чистый и лаконичный код
@Test
void shouldDeactivateUser() {
    User user = User.builder()
        .id(1L)
        .firstName("Иван")
        .lastName("Петров")
        .email("ivan@mail.com")
        .age(30)
        .role("USER")
        .active(true)
        .createdAt(LocalDateTime.now())
        .build();

    userService.deactivate(user.getId());

    assertThat(user.isActive()).isFalse();
}
```

### Кастомный Test Builder (без Lombok)

```java
// Для случаев, когда нужна дополнительная логика или нет Lombok
public class TestUserBuilder {

    private Long id = 1L;
    private String firstName = "Тест";
    private String lastName = "Пользователь";
    private String email = "test@example.com";
    private int age = 25;
    private String role = "USER";
    private boolean active = true;
    private LocalDateTime createdAt = LocalDateTime.now();

    // Дефолтные значения уже заданы — можно создать объект без настроек
    public static TestUserBuilder aUser() {
        return new TestUserBuilder();
    }

    public TestUserBuilder withId(Long id) {
        this.id = id;
        return this;
    }

    public TestUserBuilder withName(String firstName, String lastName) {
        this.firstName = firstName;
        this.lastName = lastName;
        return this;
    }

    public TestUserBuilder withEmail(String email) {
        this.email = email;
        return this;
    }

    public TestUserBuilder withAge(int age) {
        this.age = age;
        return this;
    }

    public TestUserBuilder withRole(String role) {
        this.role = role;
        return this;
    }

    public TestUserBuilder inactive() {
        this.active = false;
        return this;
    }

    public User build() {
        User user = new User();
        user.setId(id);
        user.setFirstName(firstName);
        user.setLastName(lastName);
        user.setEmail(email);
        user.setAge(age);
        user.setRole(role);
        user.setActive(active);
        user.setCreatedAt(createdAt);
        return user;
    }
}

// Использование: минимум кода для создания тестового объекта
@Test
void shouldAllowAdminToDeleteUser() {
    User admin = TestUserBuilder.aUser()
        .withRole("ADMIN")
        .build();

    User target = TestUserBuilder.aUser()
        .withId(2L)
        .withEmail("target@mail.com")
        .build();

    boolean result = userService.delete(target.getId(), admin);
    assertThat(result).isTrue();
}
```

---

## Object Mother — паттерн создания тестовых объектов

Object Mother — фабричный класс, предоставляющий готовые тестовые объекты с говорящими именами.

```java
public class UserMother {

    private static final Faker faker = new Faker();

    /**
     * Стандартный активный пользователь с ролью USER
     */
    public static User typicalUser() {
        return User.builder()
            .id(1L)
            .firstName("Иван")
            .lastName("Петров")
            .email("ivan@mail.com")
            .age(30)
            .role("USER")
            .active(true)
            .createdAt(LocalDateTime.now())
            .build();
    }

    /**
     * Администратор
     */
    public static User admin() {
        return User.builder()
            .id(100L)
            .firstName("Админ")
            .lastName("Системный")
            .email("admin@company.com")
            .age(40)
            .role("ADMIN")
            .active(true)
            .createdAt(LocalDateTime.now().minusYears(2))
            .build();
    }

    /**
     * Заблокированный пользователь
     */
    public static User blockedUser() {
        return User.builder()
            .id(50L)
            .firstName("Blocked")
            .lastName("User")
            .email("blocked@mail.com")
            .age(25)
            .role("USER")
            .active(false)
            .createdAt(LocalDateTime.now().minusMonths(6))
            .build();
    }

    /**
     * Несовершеннолетний пользователь
     */
    public static User minorUser() {
        return User.builder()
            .id(200L)
            .firstName("Юный")
            .lastName("Тестер")
            .email("minor@mail.com")
            .age(16)
            .role("USER")
            .active(true)
            .createdAt(LocalDateTime.now())
            .build();
    }

    /**
     * Пользователь со случайными данными
     */
    public static User random() {
        return User.builder()
            .id(faker.number().randomNumber())
            .firstName(faker.name().firstName())
            .lastName(faker.name().lastName())
            .email(faker.internet().emailAddress())
            .age(faker.number().numberBetween(18, 65))
            .role("USER")
            .active(true)
            .createdAt(LocalDateTime.now())
            .build();
    }
}

// Использование в тестах — максимально читаемый код
@Test
void adminShouldBeAbleToDeleteUser() {
    User admin = UserMother.admin();
    User target = UserMother.typicalUser();

    boolean result = userService.delete(target.getId(), admin);
    assertThat(result).isTrue();
}

@Test
void blockedUserShouldNotLogin() {
    User blocked = UserMother.blockedUser();

    assertThatThrownBy(() -> authService.login(blocked.getEmail(), "password"))
        .isInstanceOf(AccountBlockedException.class);
}
```

### Object Mother для заказов

```java
public class OrderMother {

    public static Order newOrder() {
        return Order.builder()
            .id(1L)
            .userId(1L)
            .status(OrderStatus.NEW)
            .items(List.of(
                new OrderItem("Laptop", 999.99, 1),
                new OrderItem("Mouse", 29.99, 2)
            ))
            .totalAmount(1059.97)
            .createdAt(LocalDateTime.now())
            .build();
    }

    public static Order paidOrder() {
        Order order = newOrder();
        order.setStatus(OrderStatus.PAID);
        order.setPaymentId("PAY-" + System.currentTimeMillis());
        return order;
    }

    public static Order shippedOrder() {
        Order order = paidOrder();
        order.setStatus(OrderStatus.SHIPPED);
        order.setTrackingNumber("TRACK-123456");
        return order;
    }

    public static Order cancelledOrder() {
        Order order = newOrder();
        order.setStatus(OrderStatus.CANCELLED);
        order.setCancellationReason("Отменено пользователем");
        return order;
    }
}
```

---

## Test Data Factory

Фабрика тестовых данных — более гибкий подход, комбинирующий Builder и Object Mother.

```java
public class TestDataFactory {

    private static final Faker faker = new Faker(new Locale("ru"));

    // ===== Пользователи =====

    public static User.UserBuilder userTemplate() {
        return User.builder()
            .id(faker.number().randomNumber())
            .firstName(faker.name().firstName())
            .lastName(faker.name().lastName())
            .email(uniqueEmail())
            .age(faker.number().numberBetween(18, 65))
            .role("USER")
            .active(true)
            .createdAt(LocalDateTime.now());
    }

    public static User randomUser() {
        return userTemplate().build();
    }

    public static User randomAdmin() {
        return userTemplate().role("ADMIN").build();
    }

    // ===== Заказы =====

    public static Order.OrderBuilder orderTemplate() {
        return Order.builder()
            .id(faker.number().randomNumber())
            .userId(faker.number().numberBetween(1L, 1000L))
            .status(OrderStatus.NEW)
            .items(randomOrderItems(faker.number().numberBetween(1, 5)))
            .createdAt(LocalDateTime.now());
    }

    public static Order randomOrder() {
        Order order = orderTemplate().build();
        order.setTotalAmount(order.getItems().stream()
            .mapToDouble(i -> i.getPrice() * i.getQuantity())
            .sum());
        return order;
    }

    // ===== Товары =====

    public static List<OrderItem> randomOrderItems(int count) {
        return IntStream.range(0, count)
            .mapToObj(i -> new OrderItem(
                faker.commerce().productName(),
                Double.parseDouble(faker.commerce().price(10, 500)),
                faker.number().numberBetween(1, 5)
            ))
            .toList();
    }

    // ===== Утилиты =====

    private static String uniqueEmail() {
        return String.format("%s.%d@test.com",
            faker.internet().username(),
            System.nanoTime());
    }

    public static String randomPhoneNumber() {
        return String.format("+7%s", faker.number().digits(10));
    }
}

// Использование
@Test
void shouldProcessOrderForNewUser() {
    // Быстрое создание тестовых данных с минимумом кода
    User user = TestDataFactory.randomUser();
    Order order = TestDataFactory.randomOrder();

    OrderResult result = orderService.process(user, order);
    assertThat(result.isSuccess()).isTrue();
}

// Кастомизация через template builder
@Test
void shouldApplyVipDiscountForHighValueOrder() {
    User vipUser = TestDataFactory.userTemplate()
        .role("VIP")
        .build();

    Order expensiveOrder = TestDataFactory.orderTemplate()
        .items(List.of(new OrderItem("Premium Laptop", 2500.0, 1)))
        .build();

    OrderResult result = orderService.process(vipUser, expensiveOrder);
    assertThat(result.getDiscount()).isGreaterThan(0);
}
```

---

## Data-Driven Testing

### Данные из CSV-файла

**Файл `src/test/resources/testdata/users.csv`:**

```csv
firstName,lastName,email,age,expectedValid
Иван,Петров,ivan@mail.com,25,true
Мария,Сидорова,maria@mail.com,30,true
,Empty,no-name@mail.com,20,false
Тест,Юзер,invalid-email,25,false
Анна,Иванова,anna@mail.com,-5,false
```

```java
class CsvDataDrivenTest {

    @ParameterizedTest
    @CsvFileSource(resources = "/testdata/users.csv", numLinesToSkip = 1)
    void shouldValidateUserFromCsv(String firstName, String lastName,
                                    String email, int age,
                                    boolean expectedValid) {
        User user = User.builder()
            .firstName(firstName)
            .lastName(lastName)
            .email(email)
            .age(age)
            .build();

        boolean isValid = userValidator.validate(user);
        assertThat(isValid)
            .as("Валидация пользователя: %s %s (%s, возраст %d)",
                firstName, lastName, email, age)
            .isEqualTo(expectedValid);
    }
}
```

### Данные из JSON-файла

**Файл `src/test/resources/testdata/orders.json`:**

```json
[
  {
    "productName": "Laptop",
    "quantity": 1,
    "unitPrice": 999.99,
    "expectedTotal": 999.99
  },
  {
    "productName": "Mouse",
    "quantity": 3,
    "unitPrice": 29.99,
    "expectedTotal": 89.97
  },
  {
    "productName": "Keyboard",
    "quantity": 0,
    "unitPrice": 49.99,
    "expectedTotal": 0.0
  }
]
```

```java
class JsonDataDrivenTest {

    @ParameterizedTest
    @MethodSource("loadOrderTestData")
    void shouldCalculateOrderTotal(String productName, int quantity,
                                    double unitPrice, double expectedTotal) {
        OrderItem item = new OrderItem(productName, unitPrice, quantity);
        double total = calculator.calculateItemTotal(item);

        assertThat(total)
            .as("Итого для %s x%d", productName, quantity)
            .isCloseTo(expectedTotal, within(0.01));
    }

    static Stream<Arguments> loadOrderTestData() throws IOException {
        ObjectMapper mapper = new ObjectMapper();
        // Чтение JSON-файла из ресурсов
        InputStream is = JsonDataDrivenTest.class
            .getResourceAsStream("/testdata/orders.json");
        List<Map<String, Object>> testData = mapper.readValue(is,
            new TypeReference<>() {});

        return testData.stream().map(data -> Arguments.of(
            data.get("productName"),
            ((Number) data.get("quantity")).intValue(),
            ((Number) data.get("unitPrice")).doubleValue(),
            ((Number) data.get("expectedTotal")).doubleValue()
        ));
    }
}
```

### @MethodSource для сложных сценариев

```java
class RegistrationDataDrivenTest {

    @ParameterizedTest(name = "Регистрация: {0}")
    @MethodSource("registrationScenarios")
    void shouldHandleRegistrationScenarios(String scenario,
                                            RegistrationRequest request,
                                            boolean shouldSucceed,
                                            String expectedError) {
        if (shouldSucceed) {
            RegistrationResult result = authService.register(request);
            assertThat(result.isSuccess()).isTrue();
        } else {
            assertThatThrownBy(() -> authService.register(request))
                .hasMessageContaining(expectedError);
        }
    }

    static Stream<Arguments> registrationScenarios() {
        Faker faker = new Faker();
        return Stream.of(
            Arguments.of(
                "Валидные данные",
                new RegistrationRequest("user1", faker.internet().emailAddress(), "Pass123!"),
                true, null
            ),
            Arguments.of(
                "Пустой email",
                new RegistrationRequest("user2", "", "Pass123!"),
                false, "Email обязателен"
            ),
            Arguments.of(
                "Слабый пароль",
                new RegistrationRequest("user3", faker.internet().emailAddress(), "123"),
                false, "Пароль слишком слабый"
            ),
            Arguments.of(
                "Пустое имя пользователя",
                new RegistrationRequest("", faker.internet().emailAddress(), "Pass123!"),
                false, "Имя пользователя обязательно"
            ),
            Arguments.of(
                "Слишком длинное имя",
                new RegistrationRequest("a".repeat(256), faker.internet().emailAddress(), "Pass123!"),
                false, "Имя слишком длинное"
            )
        );
    }
}
```

---

## Стратегии изоляции тестовых данных

### Стратегия 1: Уникальные данные для каждого теста

```java
@ExtendWith(MockitoExtension.class)
class IsolatedDataTest {

    @BeforeEach
    void setUp() {
        // Каждый тест получает свои уникальные данные —
        // нет зависимостей между тестами
        testUser = TestDataFactory.randomUser();
        testOrder = TestDataFactory.randomOrder();
    }

    @Test
    void testOne() {
        // Работает с уникальными testUser и testOrder
    }

    @Test
    void testTwo() {
        // Работает с ДРУГИМИ уникальными testUser и testOrder
    }
}
```

### Стратегия 2: Транзакционная изоляция (интеграционные тесты)

```java
@SpringBootTest
@Transactional  // Каждый тест откатывается после выполнения
class UserRepositoryIntegrationTest {

    @Autowired
    private UserRepository userRepository;

    @Test
    void shouldSaveAndFindUser() {
        User user = TestDataFactory.randomUser();
        User saved = userRepository.save(user);

        Optional<User> found = userRepository.findById(saved.getId());
        assertThat(found).isPresent();
        // После теста транзакция откатится — данные не останутся в БД
    }
}
```

### Стратегия 3: Очистка данных в @AfterEach

```java
class CleanupStrategyTest {

    private final List<Long> createdUserIds = new ArrayList<>();

    @AfterEach
    void cleanUp() {
        // Удаляем все созданные в тесте данные
        createdUserIds.forEach(id -> userRepository.deleteById(id));
        createdUserIds.clear();
    }

    @Test
    void shouldCreateUser() {
        User user = userService.create(TestDataFactory.randomUser());
        createdUserIds.add(user.getId());  // Запоминаем для очистки

        assertThat(user.getId()).isNotNull();
    }
}
```

### Стратегия 4: Testcontainers для полной изоляции

```java
@Testcontainers
@SpringBootTest
class DatabaseIsolationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    // Каждый запуск тестов — чистая база данных в контейнере
}
```

---

## Database Seeding — наполнение базы тестовыми данными

```java
// Утилитный класс для предзаполнения базы данных
public class DatabaseSeeder {

    private final UserRepository userRepository;
    private final OrderRepository orderRepository;
    private final Faker faker = new Faker(new Locale("ru"));

    public DatabaseSeeder(UserRepository userRepository,
                          OrderRepository orderRepository) {
        this.userRepository = userRepository;
        this.orderRepository = orderRepository;
    }

    /**
     * Создаёт набор тестовых пользователей для интеграционных тестов
     */
    public List<User> seedUsers(int count) {
        List<User> users = IntStream.range(0, count)
            .mapToObj(i -> User.builder()
                .firstName(faker.name().firstName())
                .lastName(faker.name().lastName())
                .email(String.format("user%d_%d@test.com", i, System.nanoTime()))
                .age(faker.number().numberBetween(18, 65))
                .role(i == 0 ? "ADMIN" : "USER")
                .active(true)
                .build())
            .toList();

        return userRepository.saveAll(users);
    }

    /**
     * Создаёт набор заказов для указанных пользователей
     */
    public List<Order> seedOrders(List<User> users, int ordersPerUser) {
        List<Order> orders = new ArrayList<>();
        for (User user : users) {
            for (int i = 0; i < ordersPerUser; i++) {
                Order order = Order.builder()
                    .userId(user.getId())
                    .status(randomStatus())
                    .items(TestDataFactory.randomOrderItems(
                        faker.number().numberBetween(1, 5)))
                    .createdAt(LocalDateTime.now().minusDays(
                        faker.number().numberBetween(0, 30)))
                    .build();
                orders.add(order);
            }
        }
        return orderRepository.saveAll(orders);
    }

    private OrderStatus randomStatus() {
        OrderStatus[] statuses = OrderStatus.values();
        return statuses[faker.number().numberBetween(0, statuses.length)];
    }
}

// Использование в тестах
@SpringBootTest
class ReportServiceIntegrationTest {

    @Autowired private DatabaseSeeder seeder;

    @BeforeEach
    void setUp() {
        // Наполняем базу тестовыми данными перед каждым тестом
        List<User> users = seeder.seedUsers(10);
        seeder.seedOrders(users, 3);
    }

    @Test
    void shouldGenerateMonthlyReport() {
        Report report = reportService.generateMonthly(YearMonth.now());
        assertThat(report.getTotalOrders()).isGreaterThan(0);
        assertThat(report.getActiveUsers()).isGreaterThan(0);
    }
}
```

---

## Практические примеры

### Создание реалистичных тестовых пользователей

```java
public class RealisticUserFactory {

    private static final Faker faker = new Faker(new Locale("ru"));

    /**
     * Создаёт полностью заполненный профиль пользователя
     * для тестирования UI-форм и API-запросов
     */
    public static UserProfile createFullProfile() {
        String firstName = faker.name().firstName();
        String lastName = faker.name().lastName();
        LocalDate birthDate = faker.date().birthday(18, 65)
            .toInstant().atZone(ZoneId.systemDefault()).toLocalDate();

        return UserProfile.builder()
            .firstName(firstName)
            .lastName(lastName)
            .email(uniqueEmail(firstName, lastName))
            .phone("+7" + faker.number().digits(10))
            .birthDate(birthDate)
            .address(Address.builder()
                .city(faker.address().city())
                .street(faker.address().streetAddress())
                .zipCode(faker.address().zipCode())
                .country("Россия")
                .build())
            .occupation(faker.job().title())
            .company(faker.company().name())
            .bio(faker.lorem().sentence(15))
            .avatarUrl(faker.internet().avatar())
            .build();
    }

    private static String uniqueEmail(String firstName, String lastName) {
        String transliterated = Transliterator.transliterate(
            firstName.toLowerCase() + "." + lastName.toLowerCase());
        return String.format("%s.%d@test.com", transliterated, System.nanoTime());
    }
}
```

### Генерация тестовых заказов с реалистичными товарами

```java
public class OrderTestDataFactory {

    private static final Faker faker = new Faker();

    /**
     * Заказ с одним дорогим товаром
     */
    public static Order highValueOrder(Long userId) {
        return Order.builder()
            .userId(userId)
            .status(OrderStatus.NEW)
            .items(List.of(
                new OrderItem(
                    faker.commerce().productName(),
                    faker.number().randomDouble(2, 500, 5000),
                    1
                )
            ))
            .createdAt(LocalDateTime.now())
            .build();
    }

    /**
     * Заказ с несколькими дешёвыми товарами
     */
    public static Order bulkOrder(Long userId) {
        List<OrderItem> items = IntStream.range(0, faker.number().numberBetween(5, 20))
            .mapToObj(i -> new OrderItem(
                faker.commerce().productName(),
                faker.number().randomDouble(2, 1, 50),
                faker.number().numberBetween(1, 10)
            ))
            .toList();

        return Order.builder()
            .userId(userId)
            .status(OrderStatus.NEW)
            .items(items)
            .createdAt(LocalDateTime.now())
            .build();
    }

    /**
     * Заказ с уникальным email для уведомлений
     */
    public static Order orderWithNotification(Long userId) {
        return Order.builder()
            .userId(userId)
            .status(OrderStatus.NEW)
            .notificationEmail(uniqueEmail())
            .items(List.of(
                new OrderItem("Test Product", 99.99, 1)
            ))
            .createdAt(LocalDateTime.now())
            .build();
    }

    private static String uniqueEmail() {
        return String.format("order_%d@test.com", System.nanoTime());
    }
}
```

---

## Связь с тестированием

Управление тестовыми данными напрямую влияет на качество автоматизации:

- **Стабильность тестов**: уникальные данные исключают конфликты при параллельном запуске
- **Читаемость тестов**: Object Mother и Builder делают тесты самодокументирующимися
- **Скорость разработки**: фабрики ускоряют создание новых тестов
- **Data-Driven Testing**: покрытие граничных значений и негативных сценариев через внешние файлы
- **Изоляция**: каждый тест работает с чистыми данными, независимо от других
- **Реалистичность**: Datafaker создаёт данные, близкие к продакшену, — выявляет баги, которые пропускают простые «test123»

---

## Типичные ошибки

1. **Хардкод тестовых данных** — «test@test.com» в каждом тесте → конфликты при параллельном запуске
2. **Отсутствие уникальности** — два теста создают пользователя с одним email → flaky-тесты
3. **Общие данные между тестами** — изменение в одном тесте ломает другой
4. **Слишком сложные фабрики** — фабрика данных не должна содержать бизнес-логику
5. **Забыть очистку** — тестовые данные копятся в базе, замедляя тесты
6. **Не использовать `@Transactional`** — данные остаются после интеграционного теста
7. **Хранение чувствительных данных** — реальные пароли/ключи в тестовых fixtures
8. **Игнорирование локализации** — тесты проходят с ASCII, но падают с кириллицей

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Зачем нужна генерация тестовых данных? Почему нельзя везде использовать «test123»?
2. Что такое Datafaker / JavaFaker? Как сгенерировать случайный email?
3. Что такое паттерн Builder и как он помогает в тестах?
4. Как обеспечить уникальность тестовых данных?
5. Что такое data-driven testing?

### 🟡 Средний уровень
6. Чем Object Mother отличается от Test Data Factory?
7. Как организовать data-driven тест с данными из CSV-файла?
8. Какие стратегии изоляции тестовых данных вы знаете?
9. Как работает `@CsvFileSource` в JUnit 5? Какие ограничения?
10. Как обеспечить потокобезопасность генерации данных при параллельном запуске?

### 🔴 Продвинутый уровень
11. Как спроектировать систему управления тестовыми данными для проекта с 500+ тестами?
12. Как использовать Testcontainers для полной изоляции данных?
13. Когда database seeding лучше, чем генерация данных на лету?
14. Как реализовать кросс-сервисную подготовку данных в микросервисной архитектуре?
15. Как управлять тестовыми данными в CI/CD-пайплайне с параллельными джобами?

---

## Практические задания

### Задание 1: Datafaker
Создайте утилитный класс `TestDataGenerator` с методами генерации: уникальный email, российский номер телефона, валидный пароль, реалистичный адрес. Все методы должны быть потокобезопасными.

### Задание 2: Object Mother
Реализуйте `ProductMother` для интернет-магазина с методами: `inStockProduct()`, `outOfStockProduct()`, `discountedProduct()`, `premiumProduct()`, `randomProduct()`.

### Задание 3: Data-Driven Test
Создайте параметризованный тест для валидатора паролей. Данные в CSV-файле: пароль, ожидаемый результат, описание кейса. Минимум 15 тестовых случаев.

### Задание 4: Test Data Factory
Спроектируйте `TestDataFactory` для системы бронирования: пользователь, отель, номер, бронирование. Связи между объектами должны быть корректными.

### Задание 5: Изоляция данных
Напишите интеграционный тест с использованием `@Transactional` и database seeding. Убедитесь, что тесты проходят при параллельном запуске без конфликтов.

---

## Дополнительные ресурсы

- [Datafaker GitHub](https://github.com/datafaker-net/datafaker)
- [Datafaker Documentation](https://www.datafaker.net/documentation/getting-started/)
- [Object Mother Pattern](https://martinfowler.com/bliki/ObjectMother.html)
- [Test Data Builder Pattern](https://blog.ploeh.dk/2017/08/15/test-data-builders-in-c/)
- [Testcontainers](https://testcontainers.com/)
- [Baeldung — JavaFaker](https://www.baeldung.com/java-faker)
- [Baeldung — Test Data](https://www.baeldung.com/java-test-data-generation)
