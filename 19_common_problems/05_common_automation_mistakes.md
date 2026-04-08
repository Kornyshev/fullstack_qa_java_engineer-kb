# Типичные ошибки автоматизации тестирования

## Обзор

Автоматизация тестирования — это не просто «написать код, который кликает по кнопкам».
Это инженерная дисциплина со своими паттернами, антипаттернами и best practices. Начинающие
(и не только) QA-инженеры регулярно допускают одни и те же ошибки, которые приводят к хрупким,
медленным и дорогим в поддержке тестам.

В этом разделе собраны 9 самых распространённых ошибок автоматизации с конкретными примерами
«плохого» и «хорошего» кода. Каждая ошибка проиллюстрирована реальным сценарием и сопровождается
объяснением, почему это плохо и как сделать правильно.

На собеседованиях часто просят: «Назовите типичные ошибки автоматизации» или дают код
и просят найти проблемы — этот материал покрывает оба формата вопросов.

---

## Ошибка 1: Hardcoded Values (захардкоженные значения)

### Проблема

URL-ы, учётные данные, тестовые данные и другие параметры прописаны прямо в коде теста.
Любое изменение окружения или данных требует правки множества файлов.

### Плохой пример

```java
// ПЛОХО — всё захардкожено прямо в тесте
class LoginTest {

    @Test
    void testSuccessfulLogin() {
        WebDriver driver = new ChromeDriver();
        driver.get("http://192.168.1.105:8080/login"); // IP-адрес тестового стенда

        driver.findElement(By.id("username")).sendKeys("admin@company.com");
        driver.findElement(By.id("password")).sendKeys("P@ssw0rd123!");

        driver.findElement(By.id("login-btn")).click();

        // Хардкод ожидаемого текста
        String welcome = driver.findElement(By.id("welcome")).getText();
        assertEquals("Добро пожаловать, Администратор!", welcome);

        driver.quit();
    }
}
```

**Что здесь плохо:**
- IP-адрес стенда изменится — тест сломается
- Пароль попадёт в git-историю (security risk)
- Нельзя запустить тест на другом окружении без правки кода
- При смене тестового пользователя — правки в каждом тесте

### Хороший пример

```java
// ХОРОШО — все параметры вынесены в конфигурацию
class LoginTest extends BaseUiTest {

    @Test
    void testSuccessfulLogin() {
        // URL берётся из конфигурации окружения
        open(Config.getBaseUrl() + "/login");

        LoginPage loginPage = new LoginPage(driver);

        // Учётные данные из безопасного хранилища
        TestUser admin = TestUsers.getUser("admin");

        DashboardPage dashboard = loginPage
            .enterUsername(admin.getUsername())
            .enterPassword(admin.getPassword())
            .clickLogin();

        // Проверка через Page Object, без хардкода текста
        assertTrue(dashboard.isWelcomeMessageDisplayed());
        assertThat(dashboard.getWelcomeMessage())
            .contains(admin.getDisplayName());
    }
}

// Конфигурация — вынесена из кода
public class Config {
    private static final Properties props = loadProperties();

    public static String getBaseUrl() {
        return props.getProperty("base.url");
    }

    private static Properties loadProperties() {
        String env = System.getProperty("test.env", "local");
        // Загрузка из config/local.properties или config/test.properties
        Properties p = new Properties();
        try (var is = Config.class.getResourceAsStream(
                "/config/" + env + ".properties")) {
            p.load(is);
        } catch (IOException e) {
            throw new RuntimeException("Не найден конфиг для окружения: " + env);
        }
        return p;
    }
}

// Тестовые пользователи — отдельный файл, не в git для секретов
public class TestUsers {
    private static final Map<String, TestUser> USERS = Map.of(
        "admin", new TestUser(
            System.getenv("TEST_ADMIN_EMAIL"),
            System.getenv("TEST_ADMIN_PASSWORD"),
            "Администратор"
        )
    );

    public static TestUser getUser(String role) {
        return USERS.get(role);
    }
}
```

---

## Ошибка 2: Thread.sleep вместо Proper Waits

### Проблема

Использование `Thread.sleep()` для ожидания — это гадание: «Надеюсь, 3 секунд хватит».
Иногда хватает, иногда нет. Результат — нестабильные тесты.

### Плохой пример

```java
// ПЛОХО — россыпь Thread.sleep по всему тесту
@Test
void testCheckout() throws InterruptedException {
    driver.findElement(By.id("add-to-cart")).click();
    Thread.sleep(2000); // Ждём обновления корзины

    driver.findElement(By.id("cart-icon")).click();
    Thread.sleep(3000); // Ждём загрузки страницы корзины

    driver.findElement(By.id("promo-code")).sendKeys("DISCOUNT10");
    driver.findElement(By.id("apply-promo")).click();
    Thread.sleep(5000); // Ждём применения промокода (запрос на сервер)

    String total = driver.findElement(By.id("total-price")).getText();
    assertEquals("900 ₽", total);

    driver.findElement(By.id("checkout-btn")).click();
    Thread.sleep(10000); // Ждём обработки заказа

    String status = driver.findElement(By.id("order-status")).getText();
    assertEquals("Заказ оформлен", status);
}
// Общее время ожидания: 20 секунд (даже если всё загрузилось за 2 секунды)
```

**Что здесь плохо:**
- Тест занимает минимум 20 секунд вместо 2-5
- Если сервер медленнее обычного — тест упадёт
- Если сервер быстрее — время тратится впустую
- `Thread.sleep` прерывает поток, но не проверяет состояние

### Хороший пример

```java
// ХОРОШО — явные ожидания конкретных условий
@Test
void testCheckout() {
    WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(15));

    // Добавить в корзину и дождаться обновления счётчика
    driver.findElement(By.id("add-to-cart")).click();
    wait.until(ExpectedConditions.textToBePresentInElementLocated(
        By.id("cart-count"), "1"
    ));

    // Перейти в корзину и дождаться загрузки
    driver.findElement(By.id("cart-icon")).click();
    wait.until(ExpectedConditions.visibilityOfElementLocated(By.id("cart-items")));

    // Применить промокод и дождаться пересчёта
    driver.findElement(By.id("promo-code")).sendKeys("DISCOUNT10");
    driver.findElement(By.id("apply-promo")).click();
    wait.until(ExpectedConditions.textToBePresentInElementLocated(
        By.id("total-price"), "900 ₽"
    ));

    // Оформить заказ и дождаться подтверждения
    driver.findElement(By.id("checkout-btn")).click();
    wait.until(ExpectedConditions.textToBePresentInElementLocated(
        By.id("order-status"), "Заказ оформлен"
    ));
    // Каждый wait прекращается, как только условие выполнено
    // Тест занимает ровно столько, сколько нужно приложению
}
```

---

## Ошибка 3: Зависимость между тестами (Execution Order)

### Проблема

Тесты зависят от порядка выполнения: Test B ожидает, что Test A уже создал данные.
При параллельном запуске или изменении порядка — всё ломается.

### Плохой пример

```java
// ПЛОХО — тесты зависят от порядка выполнения
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class UserFlowTest {

    private static String userId; // Общее состояние между тестами!

    @Test
    @Order(1)
    void step1_createUser() {
        // Создаём пользователя
        userId = api.createUser("John", "Doe").getId();
        assertNotNull(userId);
    }

    @Test
    @Order(2)
    void step2_updateUser() {
        // Обновляем пользователя, созданного в step1
        // Если step1 не выполнился — NullPointerException
        api.updateUser(userId, "Jane", "Doe");
        User user = api.getUser(userId);
        assertEquals("Jane", user.getFirstName());
    }

    @Test
    @Order(3)
    void step3_deleteUser() {
        // Удаляем пользователя
        api.deleteUser(userId);
        assertThrows(NotFoundException.class, () -> api.getUser(userId));
    }
}
```

**Что здесь плохо:**
- Если step1 упал — step2 и step3 тоже упадут (каскадное падение)
- Невозможно запустить step2 отдельно для отладки
- При параллельном выполнении — race condition
- Один тест проверяет слишком много (create + update + delete)

### Хороший пример

```java
// ХОРОШО — каждый тест самодостаточен и независим
class UserApiTest {

    @Test
    void testCreateUser() {
        // Arrange — ничего не нужно предварительно
        CreateUserRequest request = RequestFactory.validCreateUserRequest();

        // Act
        UserResponse response = api.createUser(request);

        // Assert
        assertNotNull(response.getId());
        assertEquals(request.getFirstName(), response.getFirstName());
    }

    @Test
    void testUpdateUser() {
        // Arrange — сам создаёт нужные данные
        UserResponse created = api.createUser(
            RequestFactory.validCreateUserRequest()
        );

        // Act
        UpdateUserRequest update = new UpdateUserRequest("Jane", "Doe");
        UserResponse updated = api.updateUser(created.getId(), update);

        // Assert
        assertEquals("Jane", updated.getFirstName());
    }

    @Test
    void testDeleteUser() {
        // Arrange — сам создаёт данные для удаления
        UserResponse created = api.createUser(
            RequestFactory.validCreateUserRequest()
        );

        // Act
        api.deleteUser(created.getId());

        // Assert
        assertThrows(NotFoundException.class,
            () -> api.getUser(created.getId()));
    }
}
```

---

## Ошибка 4: No Data Isolation (отсутствие изоляции данных)

### Проблема

Тесты используют общие данные (один и тот же пользователь, один и тот же заказ).
Результат одного теста влияет на другой.

### Плохой пример

```java
// ПЛОХО — все тесты работают с одним и тем же пользователем
class CartTest {

    // Один пользователь на все тесты!
    private static final String USER_EMAIL = "test@example.com";

    @Test
    void testAddItemToCart() {
        api.addToCart(USER_EMAIL, "product-1");
        Cart cart = api.getCart(USER_EMAIL);
        assertEquals(1, cart.getItems().size()); // Может быть > 1!
    }

    @Test
    void testRemoveItemFromCart() {
        api.removeFromCart(USER_EMAIL, "product-1");
        Cart cart = api.getCart(USER_EMAIL);
        assertTrue(cart.isEmpty()); // А если testAddItemToCart не запускался?
    }

    @Test
    void testCartTotal() {
        api.addToCart(USER_EMAIL, "product-2");
        Cart cart = api.getCart(USER_EMAIL);
        // Итого зависит от того, что в корзине — а там мусор от других тестов
        assertEquals(new BigDecimal("500.00"), cart.getTotal());
    }
}
```

### Хороший пример

```java
// ХОРОШО — каждый тест использует уникальные данные
class CartTest {

    private String testUserEmail;
    private ApiClient api;

    @BeforeEach
    void setUp() {
        // Уникальный пользователь для каждого теста
        testUserEmail = "test_" + UUID.randomUUID() + "@example.com";
        api.createUser(testUserEmail, "Test User");
    }

    @AfterEach
    void tearDown() {
        // Очистка данных после каждого теста
        try {
            api.deleteUser(testUserEmail);
        } catch (Exception e) {
            // Логируем, но не прерываем очистку
            log.warn("Не удалось удалить пользователя {}: {}", testUserEmail, e.getMessage());
        }
    }

    @Test
    void testAddItemToCart() {
        api.addToCart(testUserEmail, "product-1");
        Cart cart = api.getCart(testUserEmail);
        assertEquals(1, cart.getItems().size()); // Всегда 1 — чистая корзина
    }

    @Test
    void testRemoveItemFromCart() {
        // Сам добавляет, сам удаляет
        api.addToCart(testUserEmail, "product-1");
        api.removeFromCart(testUserEmail, "product-1");
        Cart cart = api.getCart(testUserEmail);
        assertTrue(cart.isEmpty());
    }

    @Test
    void testCartTotal() {
        api.addToCart(testUserEmail, "product-2"); // Цена: 500.00
        Cart cart = api.getCart(testUserEmail);
        assertEquals(new BigDecimal("500.00"), cart.getTotal());
    }
}
```

---

## Ошибка 5: Тестирование всего через UI (когда API быстрее)

### Проблема

Использование UI-тестов для проверки бизнес-логики, которую можно протестировать через API
в 10 раз быстрее и стабильнее. Нарушение Test Pyramid.

### Плохой пример

```java
// ПЛОХО — бизнес-логику проверяем через UI (медленно, хрупко)
@Test
void testDiscountCalculation() {
    // 1. Логин через UI (5 секунд)
    driver.get("http://app.test.com/login");
    driver.findElement(By.id("username")).sendKeys("admin");
    driver.findElement(By.id("password")).sendKeys("password");
    driver.findElement(By.id("login-btn")).click();
    Thread.sleep(3000);

    // 2. Переход в каталог (3 секунды)
    driver.findElement(By.linkText("Каталог")).click();
    Thread.sleep(2000);

    // 3. Добавление товара (3 секунды)
    driver.findElement(By.cssSelector("[data-product='laptop']")).click();
    driver.findElement(By.id("add-to-cart")).click();
    Thread.sleep(2000);

    // 4. Открытие корзины (2 секунды)
    driver.findElement(By.id("cart")).click();
    Thread.sleep(2000);

    // 5. Ввод промокода (3 секунды)
    driver.findElement(By.id("promo")).sendKeys("SALE20");
    driver.findElement(By.id("apply")).click();
    Thread.sleep(3000);

    // 6. Проверка — ради ЭТОГО всё затевалось (0.1 секунды)
    String total = driver.findElement(By.id("total")).getText();
    assertEquals("80 000 ₽", total); // 100 000 - 20% = 80 000
}
// Итого: ~23 секунды на проверку одного вычисления скидки
```

### Хороший пример

```java
// ХОРОШО — бизнес-логика через API (быстро, стабильно)
@Test
void testDiscountCalculation() {
    // Arrange — создаём данные через API (0.5 секунды)
    String token = authApi.login("admin", "password");

    CartResponse cart = cartApi.addItem(token, "laptop", 1);

    // Act — применяем скидку через API (0.3 секунды)
    CartResponse discounted = cartApi.applyPromoCode(token, "SALE20");

    // Assert — проверяем бизнес-логику (мгновенно)
    assertEquals(new BigDecimal("80000.00"), discounted.getTotal());
    assertEquals(new BigDecimal("20000.00"), discounted.getDiscount());
    assertEquals("SALE20", discounted.getAppliedPromoCode());
}
// Итого: < 1 секунды

// UI-тест оставляем ТОЛЬКО для проверки визуального отображения
@Test
void testDiscountDisplayedCorrectlyInUI() {
    // Подготовка данных через API (быстро)
    String token = authApi.login("admin", "password");
    cartApi.addItem(token, "laptop", 1);
    cartApi.applyPromoCode(token, "SALE20");

    // Проверяем ТОЛЬКО отображение через UI
    open(Config.getBaseUrl() + "/cart");
    loginPage.loginAs("admin", "password");

    CartPage cartPage = new CartPage(driver);
    assertThat(cartPage.getTotalText()).contains("80 000");
    assertTrue(cartPage.isDiscountBadgeVisible());
    assertEquals("Скидка 20%", cartPage.getDiscountBadgeText());
}
```

### Test Pyramid

```
         /\
        /  \        E2E/UI тесты (10%)
       /    \       — Только визуальное отображение
      /------\      — Критические пользовательские сценарии
     /        \
    / API тесты\    API/Integration тесты (30%)
   / (30%)      \   — Бизнес-логика
  /--------------\  — Интеграции между сервисами
 /                \
/ Unit-тесты (60%) \ Unit тесты (60%)
/                    \ — Вычисления, валидации
/____________________\ — Маппинг, трансформации
```

---

## Ошибка 6: No Test Data Cleanup (отсутствие очистки данных)

### Проблема

Тесты создают данные, но не удаляют их. Со временем тестовая база «засоряется»,
и тесты начинают падать из-за дублей, нарушения constraints или переполнения.

### Плохой пример

```java
// ПЛОХО — тест создаёт данные, но не убирает за собой
@Test
void testCreateProduct() {
    Product product = new Product("SKU-001", "Ноутбук", 100000);
    Response response = api.createProduct(product);
    assertEquals(201, response.statusCode());
    // Продукт остался в базе навсегда!
    // При повторном запуске: "SKU-001 already exists"
}
```

### Хороший пример

```java
// ХОРОШО — несколько стратегий очистки

// Стратегия 1: Уникальные данные + очистка в @AfterEach
class ProductTest {

    private final List<String> createdProductIds = new ArrayList<>();

    @AfterEach
    void cleanup() {
        // Удаляем все созданные тестом продукты
        for (String id : createdProductIds) {
            try {
                api.deleteProduct(id);
            } catch (Exception e) {
                log.warn("Не удалось удалить продукт {}: {}",
                    id, e.getMessage());
            }
        }
        createdProductIds.clear();
    }

    @Test
    void testCreateProduct() {
        // Уникальный SKU при каждом запуске
        String sku = "SKU-" + UUID.randomUUID().toString().substring(0, 8);
        Product product = new Product(sku, "Ноутбук", 100000);

        Response response = api.createProduct(product);
        assertEquals(201, response.statusCode());

        // Запоминаем для очистки
        createdProductIds.add(response.extractPath("id"));
    }
}

// Стратегия 2: Транзакционные тесты (для интеграционных тестов с БД)
@Transactional // Spring откатит все изменения после теста
class ProductRepositoryTest {

    @Autowired
    private ProductRepository repository;

    @Test
    void testSaveProduct() {
        Product product = new Product("SKU-001", "Ноутбук", 100000);
        Product saved = repository.save(product);
        assertNotNull(saved.getId());
        // Транзакция будет откачена автоматически
    }
}

// Стратегия 3: Dedicated cleanup endpoint (для E2E-тестов)
@AfterAll
static void globalCleanup() {
    // Вызываем API для удаления всех тестовых данных
    api.post("/test/cleanup", Map.of(
        "prefix", "test_",        // Удалить все данные с префиксом test_
        "olderThan", "1h"         // Старше 1 часа
    ));
}
```

---

## Ошибка 7: Over-engineering Framework (переусложнение фреймворка)

### Проблема

Фреймворк стал сложнее, чем тестируемое приложение. Несколько уровней абстракции,
кастомные DSL, generic-обёртки на всё — новый человек не может разобраться.

### Плохой пример

```java
// ПЛОХО — абстракция ради абстракции
public abstract class AbstractBaseGenericPageObject<T extends AbstractBaseGenericPageObject<T>> {
    protected abstract T self();

    @SuppressWarnings("unchecked")
    public <R extends AbstractBaseGenericPageObject<R>> R navigateTo(Class<R> pageClass) {
        return PageObjectFactory.getInstance().createPage(pageClass, getDriver());
    }

    public <E> FluentWaitWrapper<E> fluentWait() {
        return new FluentWaitWrapper<>(getDriver());
    }
}

// 5 уровней наследования, чтобы кликнуть по кнопке
public class CheckoutPage
    extends AbstractTransactionalPage<CheckoutPage>
    extends AbstractAuthenticatedPage<CheckoutPage>
    extends AbstractBasePage<CheckoutPage>
    extends AbstractBaseGenericPageObject<CheckoutPage> {

    public CheckoutPage clickPay() {
        // Чтобы понять, что делает этот метод, нужно пройти 5 классов
        return performAction(Actions.CLICK, Locators.PAY_BUTTON, self());
    }
}
```

### Хороший пример

```java
// ХОРОШО — простая иерархия, понятная новому человеку за 5 минут
public class BasePage {

    protected final WebDriver driver;
    protected final WebDriverWait wait;

    public BasePage(WebDriver driver) {
        this.driver = driver;
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(10));
    }

    // Только действительно общие методы
    protected WebElement waitAndFind(By locator) {
        return wait.until(ExpectedConditions.visibilityOfElementLocated(locator));
    }

    protected void waitAndClick(By locator) {
        wait.until(ExpectedConditions.elementToBeClickable(locator)).click();
    }
}

// Конкретный Page Object — 1 уровень наследования
public class CheckoutPage extends BasePage {

    private static final By PAY_BUTTON = By.cssSelector("[data-testid='pay-btn']");
    private static final By TOTAL = By.cssSelector("[data-testid='total']");

    public CheckoutPage(WebDriver driver) {
        super(driver);
    }

    @Step("Нажать кнопку 'Оплатить'")
    public PaymentResultPage clickPay() {
        waitAndClick(PAY_BUTTON);
        return new PaymentResultPage(driver);
    }

    @Step("Получить сумму заказа")
    public String getTotal() {
        return waitAndFind(TOTAL).getText();
    }
}
// Любой новый человек поймёт этот код за 2 минуты
```

**Правило:** Если новый член команды не может понять фреймворк за один день — он переусложнён.

---

## Ошибка 8: Copy-Paste Tests (копипаста тестов)

### Проблема

Вместо параметризации или вынесения общей логики — копируются целые тестовые методы
с минимальными изменениями. Поддержка 50 одинаковых тестов вместо одного параметризованного.

### Плохой пример

```java
// ПЛОХО — 6 одинаковых тестов, различающихся только данными
@Test
void testValidEmail_simple() {
    assertTrue(validator.isValid("user@mail.com"));
}

@Test
void testValidEmail_withSubdomain() {
    assertTrue(validator.isValid("user@sub.mail.com"));
}

@Test
void testValidEmail_withPlus() {
    assertTrue(validator.isValid("user+tag@mail.com"));
}

@Test
void testInvalidEmail_noAt() {
    assertFalse(validator.isValid("usermail.com"));
}

@Test
void testInvalidEmail_noDomain() {
    assertFalse(validator.isValid("user@"));
}

@Test
void testInvalidEmail_empty() {
    assertFalse(validator.isValid(""));
}
// А если нужно добавить проверку ещё 20 email-адресов?
// Или если нужно изменить формат вызова validator?
```

### Хороший пример

```java
// ХОРОШО — один параметризованный тест заменяет множество копий

@ParameterizedTest(name = "Email \"{0}\" → ожидается: {1}")
@MethodSource("emailTestData")
void testEmailValidation(String email, boolean expected, String description) {
    assertEquals(expected, validator.isValid(email),
        () -> "Ошибка при проверке: " + description);
}

static Stream<Arguments> emailTestData() {
    return Stream.of(
        // Валидные email
        Arguments.of("user@mail.com", true, "Простой email"),
        Arguments.of("user@sub.mail.com", true, "С поддоменом"),
        Arguments.of("user+tag@mail.com", true, "С тегом"),
        Arguments.of("user.name@mail.com", true, "С точкой в имени"),
        Arguments.of("USER@MAIL.COM", true, "В верхнем регистре"),

        // Невалидные email
        Arguments.of("usermail.com", false, "Без @"),
        Arguments.of("user@", false, "Без домена"),
        Arguments.of("@mail.com", false, "Без имени"),
        Arguments.of("", false, "Пустая строка"),
        Arguments.of("user @mail.com", false, "С пробелом"),
        Arguments.of("user@@mail.com", false, "Двойной @")
    );
}
// Добавить новый кейс = добавить одну строку
// Изменить логику вызова = поправить один метод
```

---

## Ошибка 9: Игнорирование качества тестового кода

### Проблема

К тестовому коду не применяются те же стандарты качества, что и к продуктовому:
нет code review, нет рефакторинга, нет code style, нет документации.

### Плохой пример

```java
// ПЛОХО — «тестовый код не обязан быть красивым»
@Test
void test1() {
    var r = given().body("{\"name\":\"a\",\"email\":\"b@c.d\"}").post("/users");
    assertEquals(201, r.statusCode());
    var id = r.path("id");
    var r2 = given().get("/users/" + id);
    assertEquals("a", r2.path("name"));
    // Что тестирует test1? Без чтения кода — непонятно
    // Что такое "a", "b@c.d"? Волшебные значения
    // Нет cleanup — данные останутся в базе
}

@Test
void test2() { /* ещё 50 строк такого же кода */ }

@Test
void test_3_DISABLED() { /* закомментированный тест, висит уже год */ }
```

### Хороший пример

```java
// ХОРОШО — тестовый код как продуктовый: чистый, читаемый, поддерживаемый

class UserCreationApiTest extends BaseApiTest {

    @Test
    @DisplayName("Создание пользователя с валидными данными возвращает 201 и данные пользователя")
    @Story("Управление пользователями")
    @Severity(SeverityLevel.CRITICAL)
    void shouldCreateUserWithValidData() {
        // Arrange — подготовка данных через фабрику
        CreateUserRequest request = RequestFactory.validCreateUserRequest();

        // Act — выполнение действия
        UserResponse response = userApi.createUser(request);

        // Assert — проверка результата с понятными assertions
        assertAll("Проверка созданного пользователя",
            () -> assertNotNull(response.getId(), "ID пользователя не должен быть null"),
            () -> assertEquals(request.getFirstName(), response.getFirstName(),
                "Имя должно совпадать с запросом"),
            () -> assertEquals(request.getEmail(), response.getEmail(),
                "Email должен совпадать с запросом"),
            () -> assertNotNull(response.getCreatedAt(),
                "Дата создания должна быть заполнена")
        );
    }

    @Test
    @DisplayName("Создание пользователя с дублирующим email возвращает 409")
    @Story("Управление пользователями")
    @Severity(SeverityLevel.NORMAL)
    void shouldReturn409WhenEmailAlreadyExists() {
        // Arrange — создаём пользователя
        CreateUserRequest request = RequestFactory.validCreateUserRequest();
        userApi.createUser(request);

        // Act — пытаемся создать второго с тем же email
        CreateUserRequest duplicate = RequestFactory.createUserRequest(
            b -> b.email(request.getEmail())
        );

        // Assert — ожидаем 409 Conflict
        ApiException exception = assertThrows(
            ApiException.class,
            () -> userApi.createUser(duplicate)
        );
        assertEquals(409, exception.getStatusCode());
        assertThat(exception.getMessage()).contains("already exists");
    }
}
```

**Чек-лист качества тестового кода:**

| Практика | Описание |
|----------|----------|
| **Naming** | Имя теста описывает сценарий: `shouldReturnErrorWhenInvalidInput` |
| **AAA Pattern** | Arrange-Act-Assert — чёткая структура каждого теста |
| **No Magic Values** | Все значения имеют смысл или объявлены как константы |
| **Single Responsibility** | Один тест проверяет одну вещь |
| **Cleanup** | Тест убирает за собой данные |
| **Code Review** | Тесты проходят такой же review, как продуктовый код |
| **No Dead Code** | Закомментированные тесты удалены |
| **Documentation** | Allure-аннотации (@Story, @Severity) для трассируемости |

---

## Связь с тестированием

Все перечисленные ошибки влияют на эффективность тестирования:

- **Hardcoded values** → невозможность мультисредового тестирования
- **Thread.sleep** → медленные и нестабильные тесты
- **Test dependencies** → каскадные падения, сложная отладка
- **No data isolation** → ложные срабатывания, загрязнение базы
- **Everything through UI** → медленный feedback loop, хрупкие тесты
- **No cleanup** → деградация окружения со временем
- **Over-engineering** → высокий порог входа, медленная разработка тестов
- **Copy-paste** → высокая стоимость поддержки
- **Low code quality** → нечитаемые тесты, трудная диагностика

---

## Типичные ошибки

1. **«Это же тесты, не продуктовый код»** — тестовый код живёт дольше и меняется чаще
2. **«Thread.sleep работает, зачем менять»** — работает до первого CI с медленным сервером
3. **«Мне так быстрее — скопирую тест и поменяю данные»** — через месяц придётся менять 50 копий
4. **«Всё проверим через UI»** — тесты будут идти часами и падать из-за любого CSS-изменения
5. **«Тесты чистить необязательно»** — через неделю тестовая база будет непригодна
6. **«Фреймворк должен быть универсальным»** — универсальность = сложность = медленная разработка
7. **«Хардкод — это быстрее»** — быстрее написать, но медленнее поддерживать

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Почему нельзя использовать `Thread.sleep` в тестах? Какая альтернатива?
2. Что такое Test Data Isolation? Зачем она нужна?
3. Почему тесты не должны зависеть от порядка выполнения?
4. Что плохого в hardcoded URL-ах и паролях в тестах?

### 🟡 Средний уровень
5. Объясните Test Pyramid. Как определить, что проверять через UI, а что через API?
6. Назовите 5 типичных ошибок автоматизации и способы их устранения.
7. Как организовать cleanup тестовых данных? Назовите несколько стратегий.
8. Как отличить хорошо написанный тест от плохо написанного? Какие критерии?
9. Что такое over-engineering в контексте тестового фреймворка?

### 🔴 Продвинутый уровень
10. Вам дали test suite с 300 тестами, полный перечисленных антипаттернов. Опишите стратегию рефакторинга.
11. Как внедрить стандарты качества тестового кода в команде (code review, linting, ArchUnit)?
12. Как измерить стоимость каждого антипаттерна в часах/деньгах для обоснования рефакторинга?
13. Спроектируйте фреймворк с минимальной абстракцией, который при этом покрывает все потребности.
14. Как автоматически выявлять перечисленные антипаттерны в тестовом коде (static analysis)?

---

## Практические задания

### Задание 1: Code Review
Вам дан следующий тестовый класс. Найдите все антипаттерны и предложите исправления:
```java
class Tests {
    static WebDriver d;
    @BeforeAll static void s() { d = new ChromeDriver(); }
    @Test void t1() throws Exception {
        d.get("http://localhost:8080");
        d.findElement(By.xpath("//input[1]")).sendKeys("admin");
        d.findElement(By.xpath("//input[2]")).sendKeys("admin");
        d.findElement(By.xpath("//button")).click();
        Thread.sleep(5000);
        assertTrue(d.findElement(By.xpath("//div[@class='welcome']")).isDisplayed());
    }
    @Test void t2() throws Exception {
        d.findElement(By.linkText("Users")).click();
        Thread.sleep(3000);
        assertEquals(10, d.findElements(By.className("user-row")).size());
    }
    @AfterAll static void e() { d.quit(); }
}
```

### Задание 2: Рефакторинг
Перепишите тесты из Задания 1, устранив все антипаттерны:
- Добавьте Page Objects
- Замените Thread.sleep на Explicit Waits
- Устраните зависимость между тестами
- Вынесите конфигурацию
- Добавьте Allure-аннотации

### Задание 3: Параметризация
У вас есть API endpoint `POST /api/validate`, который принимает JSON с полем `phone`
и возвращает `{ "valid": true/false }`. Напишите один параметризованный тест,
покрывающий 15 сценариев (валидные и невалидные номера телефонов).

### Задание 4: Test Pyramid
Для функциональности «Оформление заказа» (корзина, применение скидки, выбор доставки,
оплата, подтверждение) распределите тестовые сценарии по уровням Test Pyramid:
- Какие проверки на уровне Unit?
- Какие на уровне API/Integration?
- Какие на уровне UI/E2E?
- Обоснуйте каждое решение.

---

## Дополнительные ресурсы

- [Selenium Best Practices](https://www.selenium.dev/documentation/test_practices/) — официальные рекомендации Selenium
- [JUnit 5: Parameterized Tests](https://junit.org/junit5/docs/current/user-guide/#writing-tests-parameterized-tests) — параметризованные тесты
- [Google Testing Blog](https://testing.googleblog.com/) — статьи о практиках тестирования
- Martin Fowler: "Test Pyramid" — классическая статья о пирамиде тестирования
- [Clean Code by Robert C. Martin](https://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882) — принципы чистого кода, применимые к тестам
- [xUnit Test Patterns by Gerard Meszaros](http://xunitpatterns.com/) — паттерны и антипаттерны тестирования
- [ArchUnit](https://www.archunit.org/) — автоматическая проверка архитектурных правил
