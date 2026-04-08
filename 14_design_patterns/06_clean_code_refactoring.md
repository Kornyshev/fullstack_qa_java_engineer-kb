# Чистый код и рефакторинг тестов

## Обзор

Чистый код (Clean Code) — это код, который легко читать, понимать и изменять. В контексте
тестирования чистый код особенно важен: тесты — это **живая документация** проекта. Если
тесты трудно читать, никто не поймёт, что именно они проверяют, и никто не захочет их
поддерживать.

Рефакторинг — это процесс **улучшения внутренней структуры кода** без изменения его
внешнего поведения. Для тестов это означает: тесты после рефакторинга проверяют то же самое,
но стали чище, быстрее и проще в поддержке.

Ключевые принципы:

- **DRY** (Don't Repeat Yourself) — не дублируйте логику.
- **KISS** (Keep It Simple, Stupid) — не усложняйте без необходимости.
- **YAGNI** (You Ain't Gonna Need It) — не реализуйте то, что не нужно прямо сейчас.

---

## Code Smells в тестах

Code smell (запах кода) — признак потенциальной проблемы. Сам по себе smell не является
багом, но указывает на необходимость рефакторинга.

### 1. God Test (Тест-бог)

Один тест проверяет слишком много. При падении невозможно быстро определить причину.

```java
/**
 * ПЛОХО: God Test — проверяет регистрацию, логин, профиль и выход.
 * Если упадёт — где именно проблема?
 */
@Test
void testUserFlow() {
    // Регистрация
    driver.get(baseUrl + "/register");
    driver.findElement(By.id("name")).sendKeys("Test User");
    driver.findElement(By.id("email")).sendKeys("test@test.com");
    driver.findElement(By.id("password")).sendKeys("Pass123!");
    driver.findElement(By.id("confirmPassword")).sendKeys("Pass123!");
    driver.findElement(By.id("registerBtn")).click();
    assertEquals("Registration successful", driver.findElement(By.id("message")).getText());

    // Логин
    driver.get(baseUrl + "/login");
    driver.findElement(By.id("email")).sendKeys("test@test.com");
    driver.findElement(By.id("password")).sendKeys("Pass123!");
    driver.findElement(By.id("loginBtn")).click();
    assertEquals("Dashboard", driver.getTitle());

    // Профиль
    driver.findElement(By.id("profileLink")).click();
    assertEquals("Test User", driver.findElement(By.id("userName")).getText());

    // Выход
    driver.findElement(By.id("logoutBtn")).click();
    assertTrue(driver.getCurrentUrl().contains("/login"));
}
```

**Решение:** разбейте на отдельные тесты, каждый из которых проверяет одно поведение.

### 2. Copy-Paste Tests (Тесты-клоны)

Тесты с почти идентичным кодом, отличающимся парой значений.

```java
/**
 * ПЛОХО: три теста с 90% одинакового кода.
 */
@Test
void shouldLoginAsAdmin() {
    driver.get(baseUrl + "/login");
    driver.findElement(By.id("email")).sendKeys("admin@test.com");
    driver.findElement(By.id("password")).sendKeys("admin123");
    driver.findElement(By.id("loginBtn")).click();
    assertEquals("Admin Dashboard", driver.findElement(By.id("title")).getText());
}

@Test
void shouldLoginAsManager() {
    driver.get(baseUrl + "/login");
    driver.findElement(By.id("email")).sendKeys("manager@test.com");
    driver.findElement(By.id("password")).sendKeys("manager123");
    driver.findElement(By.id("loginBtn")).click();
    assertEquals("Manager Dashboard", driver.findElement(By.id("title")).getText());
}

@Test
void shouldLoginAsUser() {
    driver.get(baseUrl + "/login");
    driver.findElement(By.id("email")).sendKeys("user@test.com");
    driver.findElement(By.id("password")).sendKeys("user123");
    driver.findElement(By.id("loginBtn")).click();
    assertEquals("User Dashboard", driver.findElement(By.id("title")).getText());
}
```

**Решение:** параметризованные тесты.

```java
/**
 * ХОРОШО: один параметризованный тест вместо трёх одинаковых.
 */
@ParameterizedTest(name = "Логин как {0}: ожидается заголовок ''{3}''")
@CsvSource({
    "admin,   admin@test.com,   admin123,   Admin Dashboard",
    "manager, manager@test.com, manager123, Manager Dashboard",
    "user,    user@test.com,    user123,    User Dashboard"
})
void shouldLoginWithRole(String role, String email, String password,
                         String expectedTitle) {
    LoginPage loginPage = new LoginPage(driver);
    DashboardPage dashboard = loginPage.loginAs(email, password);
    assertEquals(expectedTitle, dashboard.getTitle(),
        "Неверный заголовок для роли: " + role);
}
```

### 3. Неинформативные имена тестов

```java
/**
 * ПЛОХО: имя теста ничего не говорит о том, что проверяется.
 */
@Test
void test1() { ... }

@Test
void testLogin() { ... }

@Test
void loginTest_v2_final() { ... }
```

```java
/**
 * ХОРОШО: имена следуют шаблону should_ExpectedBehavior_When_Condition.
 */
@Test
void shouldShowErrorMessage_WhenPasswordIsEmpty() { ... }

@Test
void shouldRedirectToDashboard_WhenCredentialsAreValid() { ... }

@Test
void shouldLockAccount_WhenThreeFailedLoginAttempts() { ... }
```

### 4. Magic Numbers и Magic Strings

```java
/**
 * ПЛОХО: магические числа и строки без объяснения.
 */
@Test
void testPagination() {
    // Что значит 10? Почему 3? Что такое "PROD-001"?
    assertEquals(10, searchPage.getResultsCount());
    searchPage.goToPage(3);
    assertEquals(10, searchPage.getResultsCount());
    assertTrue(searchPage.hasProduct("PROD-001"));
}
```

```java
/**
 * ХОРОШО: константы с говорящими именами.
 */
private static final int RESULTS_PER_PAGE = 10;
private static final int THIRD_PAGE = 3;
private static final String KNOWN_PRODUCT_ID = "PROD-001";

@Test
void shouldShowCorrectNumberOfResultsOnEachPage() {
    assertEquals(RESULTS_PER_PAGE, searchPage.getResultsCount());
    searchPage.goToPage(THIRD_PAGE);
    assertEquals(RESULTS_PER_PAGE, searchPage.getResultsCount());
    assertTrue(searchPage.hasProduct(KNOWN_PRODUCT_ID));
}
```

### 5. Закомментированные тесты

```java
/**
 * ПЛОХО: закомментированные тесты — мёртвый код.
 * Никто не знает, актуальны ли они. Никто не удалит.
 */
// @Test
// void shouldSendEmailNotification() {
//     // Этот тест не работает с новым email-сервером
//     // TODO: починить когда-нибудь
//     ...
// }
```

```java
/**
 * ХОРОШО: если тест временно не нужен — используйте @Disabled с причиной.
 * Он будет виден в отчёте, и о нём не забудут.
 */
@Test
@Disabled("Email-сервер мигрирует на новый API — JIRA-1234")
void shouldSendEmailNotification() {
    ...
}
```

### 6. Хардкод URL и окружения

```java
/**
 * ПЛОХО: URL зашит в код. Невозможно запустить в другом окружении.
 */
@Test
void testApi() {
    given()
        .baseUri("http://192.168.1.100:8080") // Захардкожено!
        .when()
        .get("/api/users")
        .then()
        .statusCode(200);
}
```

```java
/**
 * ХОРОШО: URL берётся из конфигурации.
 */
@Test
void shouldReturnAllUsers() {
    given()
        .baseUri(ConfigHolder.getInstance().getBaseUrl())
        .when()
        .get("/api/users")
        .then()
        .statusCode(200);
}
```

### 7. Неявные зависимости между тестами

```java
/**
 * ПЛОХО: test2 зависит от результата test1.
 * Если test1 не запустится — test2 упадёт.
 */
class OrderTest {
    static String orderId; // Общее состояние между тестами!

    @Test
    @Order(1)
    void test1_createOrder() {
        orderId = api.createOrder(orderData); // Сохраняем в static
    }

    @Test
    @Order(2)
    void test2_cancelOrder() {
        api.cancelOrder(orderId); // Зависимость от test1!
    }
}
```

```java
/**
 * ХОРОШО: каждый тест независим и самодостаточен.
 */
class OrderTest {

    @Test
    void shouldCancelExistingOrder() {
        // Arrange: тест сам создаёт данные
        String orderId = testDataFacade.createOrder(orderData);

        // Act
        var response = api.cancelOrder(orderId);

        // Assert
        assertEquals(200, response.statusCode());
    }
}
```

---

## DRY / KISS / YAGNI в тестовом коде

### DRY — Don't Repeat Yourself

Каждый фрагмент знания должен иметь **единственное представление** в системе.

```java
/**
 * ПЛОХО: дублирование авторизации в каждом тесте.
 */
@Test
void testGetUser() {
    String token = given().body(loginJson).post("/auth").jsonPath().getString("token");
    given().header("Authorization", "Bearer " + token).get("/users/1").then().statusCode(200);
}

@Test
void testGetOrders() {
    String token = given().body(loginJson).post("/auth").jsonPath().getString("token");
    given().header("Authorization", "Bearer " + token).get("/orders").then().statusCode(200);
}
```

```java
/**
 * ХОРОШО: авторизация вынесена в @BeforeEach или хелпер-метод.
 */
private String authToken;

@BeforeEach
void authenticate() {
    authToken = AuthHelper.getToken("admin", "password");
}

@Test
void shouldReturnUserById() {
    given()
        .header("Authorization", "Bearer " + authToken)
        .get("/users/1")
        .then().statusCode(200);
}
```

### KISS — Keep It Simple, Stupid

Не усложняйте решение без необходимости.

```java
/**
 * ПЛОХО: overengineering — абстрактная фабрика для создания одного объекта.
 */
public interface AssertionStrategyFactory {
    AssertionStrategy createStrategy(String type);
}
public interface AssertionStrategy {
    void assertResult(Object actual, Object expected);
}
// + 5 классов реализации для обычного assertEquals...

/**
 * ХОРОШО: просто assertEquals.
 */
assertEquals(expected, actual);
```

### YAGNI — You Ain't Gonna Need It

Не создавайте функциональность "про запас".

```java
/**
 * ПЛОХО: Page Object с методами, которые не используются ни в одном тесте.
 * "На будущее" = никогда.
 */
public class LoginPage {
    public void loginAs(String user, String pass) { ... }
    public void loginViaSSO() { ... }           // Нет тестов для SSO
    public void loginViaBiometric() { ... }     // Нет биометрии в приложении
    public void switchLanguage(String lang) { ... } // Не тестируем локализацию
    public void resetPassword(String email) { ... } // На другой странице
}

/**
 * ХОРОШО: только методы, которые реально используются тестами.
 */
public class LoginPage {
    public DashboardPage loginAs(String user, String pass) { ... }
    public LoginPage loginWithInvalidCredentials(String user, String pass) { ... }
    public String getErrorMessage() { ... }
}
```

---

## Техники рефакторинга тестов

### 1. Extract Method (Извлечение метода)

Длинный тест разбивается на осмысленные шаги.

**До:**
```java
@Test
void shouldCompleteCheckout() {
    // 50 строк кода: поиск товара, добавление в корзину, заполнение адреса,
    // выбор доставки, оплата, проверка подтверждения...
    driver.findElement(By.id("search")).sendKeys("laptop");
    driver.findElement(By.id("searchBtn")).click();
    driver.findElement(By.cssSelector(".product:first-child .add-btn")).click();
    driver.findElement(By.id("cart")).click();
    driver.findElement(By.id("checkout")).click();
    driver.findElement(By.id("address")).sendKeys("ул. Тестовая, 1");
    driver.findElement(By.id("city")).sendKeys("Москва");
    driver.findElement(By.id("zip")).sendKeys("123456");
    driver.findElement(By.id("nextStep")).click();
    driver.findElement(By.id("express")).click();
    driver.findElement(By.id("nextStep")).click();
    driver.findElement(By.id("cardNumber")).sendKeys("4111111111111111");
    driver.findElement(By.id("expiry")).sendKeys("12/25");
    driver.findElement(By.id("cvv")).sendKeys("123");
    driver.findElement(By.id("payBtn")).click();
    assertTrue(driver.findElement(By.id("confirmation")).isDisplayed());
}
```

**После:**
```java
@Test
void shouldCompleteCheckout() {
    searchPage.searchFor("laptop");
    searchPage.addFirstProductToCart();

    cartPage.proceedToCheckout();

    checkoutPage.fillShippingAddress("ул. Тестовая, 1", "Москва", "123456");
    checkoutPage.selectExpressDelivery();
    checkoutPage.proceedToPayment();

    paymentPage.payWithCard("4111111111111111", "12/25", "123");

    assertTrue(confirmationPage.isDisplayed(),
        "Страница подтверждения должна отобразиться после оплаты");
}
```

### 2. Extract Page Object

Локаторы и действия выносятся из теста в отдельный класс.

**До:**
```java
@Test
void shouldFilterByPrice() {
    driver.findElement(By.id("minPrice")).clear();
    driver.findElement(By.id("minPrice")).sendKeys("1000");
    driver.findElement(By.id("maxPrice")).clear();
    driver.findElement(By.id("maxPrice")).sendKeys("5000");
    driver.findElement(By.id("applyFilter")).click();

    List<WebElement> products = driver.findElements(By.className("product-price"));
    for (WebElement price : products) {
        int value = Integer.parseInt(price.getText().replaceAll("[^0-9]", ""));
        assertTrue(value >= 1000 && value <= 5000);
    }
}
```

**После:**
```java
// Page Object
public class CatalogPage {

    private final WebDriver driver;
    private static final By MIN_PRICE = By.id("minPrice");
    private static final By MAX_PRICE = By.id("maxPrice");
    private static final By APPLY_FILTER = By.id("applyFilter");
    private static final By PRODUCT_PRICES = By.className("product-price");

    public CatalogPage(WebDriver driver) {
        this.driver = driver;
    }

    public CatalogPage filterByPrice(int min, int max) {
        clearAndType(MIN_PRICE, String.valueOf(min));
        clearAndType(MAX_PRICE, String.valueOf(max));
        driver.findElement(APPLY_FILTER).click();
        return this;
    }

    public List<Integer> getDisplayedPrices() {
        return driver.findElements(PRODUCT_PRICES).stream()
            .map(el -> el.getText().replaceAll("[^0-9]", ""))
            .map(Integer::parseInt)
            .toList();
    }

    private void clearAndType(By locator, String text) {
        WebElement element = driver.findElement(locator);
        element.clear();
        element.sendKeys(text);
    }
}

// Тест
@Test
void shouldFilterProductsByPriceRange() {
    int minPrice = 1000;
    int maxPrice = 5000;

    catalogPage.filterByPrice(minPrice, maxPrice);

    List<Integer> prices = catalogPage.getDisplayedPrices();
    assertAll("Все цены должны быть в диапазоне",
        () -> assertFalse(prices.isEmpty(), "Список товаров не должен быть пуст"),
        () -> prices.forEach(price ->
            assertTrue(price >= minPrice && price <= maxPrice,
                "Цена %d вне диапазона [%d, %d]".formatted(price, minPrice, maxPrice))
        )
    );
}
```

### 3. Parameterize (Параметризация)

Вместо N одинаковых тестов — один параметризованный.

**До:**
```java
@Test void shouldReject_emptyUsername() { ... }
@Test void shouldReject_tooShortUsername() { ... }
@Test void shouldReject_usernameWithSpaces() { ... }
@Test void shouldReject_usernameWithSpecialChars() { ... }
```

**После:**
```java
@ParameterizedTest(name = "Невалидный username: ''{0}'' → ошибка ''{1}''")
@MethodSource("invalidUsernames")
void shouldRejectInvalidUsername(String username, String expectedError) {
    var page = registrationPage.enterUsername(username).submit();
    assertEquals(expectedError, page.getUsernameError());
}

static Stream<Arguments> invalidUsernames() {
    return Stream.of(
        Arguments.of("", "Имя пользователя обязательно"),
        Arguments.of("ab", "Минимум 3 символа"),
        Arguments.of("user name", "Пробелы не допускаются"),
        Arguments.of("user@name", "Спецсимволы не допускаются")
    );
}
```

### 4. Replace Conditional with Polymorphism

Избавляемся от if-else в тестах через полиморфизм.

**До:**
```java
public WebDriver createDriver(String browser) {
    if (browser.equals("chrome")) {
        ChromeOptions options = new ChromeOptions();
        options.addArguments("--start-maximized");
        if (isCI) options.addArguments("--headless");
        return new ChromeDriver(options);
    } else if (browser.equals("firefox")) {
        FirefoxOptions options = new FirefoxOptions();
        if (isCI) options.addArguments("-headless");
        return new FirefoxDriver(options);
    } else if (browser.equals("edge")) {
        // ...ещё 10 строк
    }
    throw new IllegalArgumentException("Unknown browser");
}
```

**После:**
```java
// Интерфейс + реализации (паттерн Strategy)
public interface DriverFactory {
    WebDriver create(boolean headless);
}

public class ChromeFactory implements DriverFactory {
    @Override
    public WebDriver create(boolean headless) {
        var options = new ChromeOptions();
        options.addArguments("--start-maximized");
        if (headless) options.addArguments("--headless=new");
        return new ChromeDriver(options);
    }
}

// Карта фабрик вместо if-else
private static final Map<String, DriverFactory> FACTORIES = Map.of(
    "chrome", new ChromeFactory(),
    "firefox", new FirefoxFactory(),
    "edge", new EdgeFactory()
);

public WebDriver createDriver(String browser) {
    var factory = FACTORIES.get(browser.toLowerCase());
    if (factory == null) throw new IllegalArgumentException("Неизвестный браузер: " + browser);
    return factory.create(Boolean.parseBoolean(System.getenv("CI")));
}
```

### 5. Introduce Test Fixture

Выносим общие данные в fixture-методы.

**До:**
```java
@Test
void shouldUpdateUserName() {
    // Создание пользователя — дублируется в каждом тесте
    var response = given()
        .body("""
            {"name": "John", "email": "john@test.com", "role": "USER"}
            """)
        .post("/api/users");
    String userId = response.jsonPath().getString("id");

    // Собственно тест
    given().body("""{"name": "Jane"}""")
        .put("/api/users/" + userId)
        .then().statusCode(200);
}

@Test
void shouldDeleteUser() {
    // Та же самая подготовка...
    var response = given()
        .body("""
            {"name": "John", "email": "john@test.com", "role": "USER"}
            """)
        .post("/api/users");
    String userId = response.jsonPath().getString("id");

    // Собственно тест
    given().delete("/api/users/" + userId).then().statusCode(204);
}
```

**После:**
```java
private String userId;

@BeforeEach
void createTestUser() {
    // Fixture: общая подготовка данных
    userId = userApi.createUser(
        User.builder()
            .name("John")
            .email(TestDataGenerator.randomEmail())
            .role(Role.USER)
            .build()
    );
}

@AfterEach
void deleteTestUser() {
    userApi.deleteUser(userId);
}

@Test
void shouldUpdateUserName() {
    userApi.updateUser(userId, Map.of("name", "Jane"));
    assertEquals("Jane", userApi.getUser(userId).getName());
}

@Test
void shouldDeleteUser() {
    var response = userApi.deleteUser(userId);
    assertEquals(204, response.statusCode());
    userId = null; // Предотвращаем повторное удаление в @AfterEach
}
```

---

## Когда рефакторить, а когда переписывать

### Рефакторинг — постепенное улучшение

Выбирайте рефакторинг, когда:

- Тесты **работают** и проверяют нужные сценарии.
- Проблема — в структуре кода, а не в логике.
- Изменения можно делать **небольшими шагами**.
- Есть возможность проверять каждый шаг (тесты проходят после каждого изменения).

### Переписывание — с чистого листа

Выбирайте переписывание, когда:

- Тесты **не работают** и давно не поддерживаются.
- Архитектура фреймворка **принципиально неверна** (например, все тесты в одном классе).
- Миграция на другую технологию (Selenium -> Playwright).
- Стоимость рефакторинга **превышает стоимость** написания заново.

### Правило бойскаута

> Оставляй код чище, чем он был, когда ты его нашёл.

При каждом прикосновении к тесту — улучшайте его немного. Не нужно рефакторить весь файл
сразу. Достаточно:
- Дать осмысленное имя переменной.
- Вынести magic string в константу.
- Заменить `Thread.sleep()` на Explicit Wait.

---

## Чек-лист чистого теста

```
[ ] Имя теста описывает ожидаемое поведение
[ ] Тест проверяет ОДНО поведение
[ ] Тест независим от других тестов
[ ] Тест не содержит Thread.sleep()
[ ] Тест не содержит хардкод URL, паролей, ID
[ ] Тест не содержит magic numbers / magic strings
[ ] Тест следует паттерну Arrange-Act-Assert (Given-When-Then)
[ ] Тест использует Page Object (для UI) или API-клиент (для API)
[ ] Тест имеет понятное сообщение об ошибке при падении
[ ] Тест очищает за собой (teardown/cleanup)
[ ] Нет закомментированного кода
[ ] Нет дублирования с другими тестами
```

---

## Связь с тестированием

Чистый код в тестах — это не эстетика. Это **прямое влияние на качество** тестирования:

| Проблема                     | Последствие                                          |
|------------------------------|------------------------------------------------------|
| Неинформативные имена        | Невозможно понять, что сломалось, из отчёта          |
| God Test                     | Падение одного теста блокирует проверку нескольких фич |
| Copy-paste                   | Баг в локаторе нужно исправлять в 20 местах          |
| Хардкод окружения            | Тесты работают только на машине автора               |
| Magic numbers                | Через месяц никто не помнит, почему `assertEquals(42, ...)` |
| Зависимости между тестами    | Нестабильный порядок запуска = flaky tests            |

---

## Типичные ошибки

1. **Рефакторинг без safety net** — рефакторинг тестов, когда нет уверенности, что они
   проверяли что-то полезное. Сначала убедитесь, что тесты валидны.
2. **Слишком большой рефакторинг** — попытка исправить всё сразу. Лучше маленькими шагами.
3. **DRY любой ценой** — чрезмерное вынесение общего кода делает тесты нечитаемыми. Небольшое
   дублирование в тестах допустимо, если повышает читаемость.
4. **Абстракции ради абстракций** — создание BaseBaseBaseTest с тремя уровнями наследования
   усложняет понимание. KISS.
5. **Отсутствие сообщений в assert** — `assertTrue(result)` без сообщения. При падении
   видно только "expected true, got false", без контекста.
6. **Рефакторинг без коммита** — изменение 50 файлов одним коммитом. Рефакторьте и
   коммитьте маленькими порциями.

---

## Вопросы на интервью

### Уровень Junior

- Что такое чистый код? Назовите 3-4 признака чистого теста.
- Что такое DRY? Как он применяется в тестах?
- Почему имена тестов важны? Какой формат именования вы используете?
- Что такое magic number? Приведите пример из тестового кода.
- Чем плох `Thread.sleep()` в тестах? Какие альтернативы?

### Уровень Middle

- Какие code smells в тестах вы встречали на практике? Как исправляли?
- Как вы решаете проблему дублирования в тестах? Где граница между DRY и читаемостью?
- Что такое паттерн Arrange-Act-Assert? Почему он важен?
- Когда стоит рефакторить тесты, а когда переписать с нуля? Какие критерии?
- Как вы организуете тестовые данные, чтобы избежать зависимостей между тестами?
- Как параметризованные тесты помогают в борьбе с дублированием?

### Уровень Senior

- Как вы организуете процесс рефакторинга в большом тестовом проекте (1000+ тестов)?
- Как убедиться, что рефакторинг не сломал существующие тесты? Какой safety net используете?
- Как вы балансируете между YAGNI и подготовкой фреймворка к будущим изменениям?
- Как внедрить культуру чистого кода в команде QA? Какие инструменты и практики?
- Какие метрики кода (code metrics) вы используете для оценки качества тестового кода?
- Как вы принимаете решение о допустимом уровне абстракции в тестовом фреймворке?

---

## Практические задания

1. **Code Review.** Проведите ревью следующего теста. Найдите все code smells и предложите
   исправления:
   ```java
   @Test
   void test1() {
       WebDriver d = new ChromeDriver();
       d.get("http://192.168.1.55:3000/login");
       d.findElement(By.xpath("/html/body/div[2]/form/input[1]")).sendKeys("admin");
       d.findElement(By.xpath("/html/body/div[2]/form/input[2]")).sendKeys("123");
       d.findElement(By.xpath("/html/body/div[2]/form/button")).click();
       Thread.sleep(5000);
       String t = d.findElement(By.xpath("/html/body/div[1]/h1")).getText();
       assertTrue(t.contains("Welcome"));
       d.findElement(By.xpath("/html/body/div[3]/a[2]")).click();
       Thread.sleep(3000);
       int count = d.findElements(By.tagName("tr")).size() - 1;
       assertEquals(5, count);
       d.findElement(By.xpath("/html/body/div[3]/a[5]")).click();
       Thread.sleep(2000);
       assertTrue(d.getPageSource().contains("Settings"));
       d.quit();
   }
   ```

2. **Рефакторинг Copy-Paste.** Возьмите три одинаковых теста для разных ролей пользователей
   и преобразуйте их в один параметризованный тест с использованием `@MethodSource`.

3. **Extract Page Object.** Возьмите тест с 30+ строками прямых обращений к `driver.findElement()`
   и выделите Page Object. Убедитесь, что тест стал занимать не более 10 строк.

4. **Рефакторинг именования.** В проекте есть тесты: `test1`, `test2`, `testLogin`, `loginTest2`,
   `testLoginNew`. Переименуйте их, следуя конвенции `should_ExpectedBehavior_When_Condition`.

5. **Применение принципов.** Возьмите существующий тестовый класс из своего проекта и
   примените чек-лист чистого теста. Зафиксируйте все нарушения. Исправьте их пошагово,
   делая коммит после каждого изменения.

6. **Устранение зависимостей.** Найдите в проекте тесты, зависящие друг от друга (через
   static-поля, порядок запуска, общие данные). Сделайте каждый тест полностью независимым.

---

## Дополнительные ресурсы

- **"Clean Code"** — Robert C. Martin — фундаментальная книга о чистом коде.
- **"Clean Code in Test Automation"** — статьи о применении Clean Code к тестам.
- **"Refactoring: Improving the Design of Existing Code"** — Martin Fowler — каталог
  техник рефакторинга с примерами.
- **"xUnit Test Patterns"** — Gerard Meszaros — паттерны и антипаттерны тестирования.
- **SonarQube** — инструмент статического анализа, обнаруживающий code smells.
- **Checkstyle / PMD** — инструменты для проверки стиля и качества Java-кода.
- **"The Art of Unit Testing"** — Roy Osherove — принципы написания хороших тестов.
- **"Working Effectively with Legacy Code"** — Michael Feathers — техники работы с
  legacy-кодом и безопасный рефакторинг.
