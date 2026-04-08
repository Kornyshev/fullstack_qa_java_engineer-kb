# ООП для QA

## Обзор

Объектно-ориентированное программирование (ООП) — парадигма, на которой построен весь инструментарий автоматизации
тестирования на Java. Page Object Model, фреймворки для API-тестов, конфигурационные слои — всё базируется на
принципах ООП. Понимание инкапсуляции, наследования, полиморфизма и абстракции позволяет строить тестовые фреймворки,
которые легко поддерживать, расширять и масштабировать. В этом разделе каждый принцип ООП рассмотрен через призму
автоматизации тестирования, а принципы SOLID показаны на реальных примерах из тестовых проектов.

---

## Четыре принципа ООП

### 1. Инкапсуляция (Encapsulation)

Инкапсуляция — сокрытие внутренней реализации и предоставление контролируемого доступа через публичные методы.
В тестовой автоматизации это означает, что детали взаимодействия с UI или API скрыты за понятным интерфейсом.

```java
/**
 * Page Object для страницы логина.
 * Локаторы инкапсулированы — тест не знает деталей вёрстки.
 */
public class LoginPage {
    // Приватные поля — детали реализации скрыты
    private final WebDriver driver;
    private final By usernameField = By.id("username");
    private final By passwordField = By.id("password");
    private final By loginButton = By.cssSelector("[data-testid='login-btn']");
    private final By errorMessage = By.className("error-msg");

    public LoginPage(WebDriver driver) {
        this.driver = driver;
    }

    // Публичные методы — контракт для тестов
    public void enterUsername(String username) {
        driver.findElement(usernameField).clear();
        driver.findElement(usernameField).sendKeys(username);
    }

    public void enterPassword(String password) {
        driver.findElement(passwordField).clear();
        driver.findElement(passwordField).sendKeys(password);
    }

    public DashboardPage clickLogin() {
        driver.findElement(loginButton).click();
        return new DashboardPage(driver);
    }

    public String getErrorMessage() {
        return driver.findElement(errorMessage).getText();
    }

    // Высокоуровневый метод — инкапсулирует весь сценарий
    public DashboardPage loginAs(String username, String password) {
        enterUsername(username);
        enterPassword(password);
        return clickLogin();
    }
}
```

```java
@Test
void testSuccessfulLogin() {
    LoginPage loginPage = new LoginPage(driver);
    // Тест читается как бизнес-сценарий, не знает о локаторах
    DashboardPage dashboard = loginPage.loginAs("admin", "secret123");
    Assertions.assertTrue(dashboard.isDisplayed());
}
```

**Преимущества:** при изменении вёрстки меняется только Page Object, тесты остаются нетронутыми.

---

### 2. Наследование (Inheritance)

Наследование позволяет создавать иерархии классов, вынося общую логику в базовый класс.
В тестовой автоматизации это используется для базовых классов тестов и Page Object'ов.

```java
/**
 * Базовый класс для всех Page Object'ов.
 * Содержит общие методы — ожидания, скриншоты, навигацию.
 */
public abstract class BasePage {
    protected final WebDriver driver;
    protected final WebDriverWait wait;

    public BasePage(WebDriver driver) {
        this.driver = driver;
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(10));
    }

    // Общие методы для всех страниц
    protected WebElement waitForElement(By locator) {
        return wait.until(ExpectedConditions.visibilityOfElementLocated(locator));
    }

    protected void click(By locator) {
        waitForElement(locator).click();
    }

    protected void type(By locator, String text) {
        WebElement element = waitForElement(locator);
        element.clear();
        element.sendKeys(text);
    }

    public String getPageTitle() {
        return driver.getTitle();
    }

    // Абстрактный метод — каждая страница определяет сама
    public abstract boolean isDisplayed();
}
```

```java
/**
 * Конкретная страница наследует BasePage.
 * Получает все общие методы бесплатно.
 */
public class ProductPage extends BasePage {
    private final By productTitle = By.cssSelector("h1.product-name");
    private final By addToCartBtn = By.id("add-to-cart");
    private final By priceLabel = By.className("price");

    public ProductPage(WebDriver driver) {
        super(driver); // Вызов конструктора родителя
    }

    @Override
    public boolean isDisplayed() {
        return waitForElement(productTitle).isDisplayed();
    }

    public String getProductName() {
        return waitForElement(productTitle).getText();
    }

    public CartPage addToCart() {
        click(addToCartBtn); // Метод из BasePage
        return new CartPage(driver);
    }
}
```

```java
/**
 * Базовый класс тестов — настройка и очистка WebDriver.
 */
public abstract class BaseTest {
    protected WebDriver driver;

    @BeforeEach
    void setUp() {
        driver = new ChromeDriver();
        driver.manage().window().maximize();
    }

    @AfterEach
    void tearDown() {
        if (driver != null) {
            driver.quit();
        }
    }
}

// Конкретный тестовый класс
class LoginTest extends BaseTest {
    @Test
    void testLogin() {
        driver.get("https://example.com/login");
        LoginPage loginPage = new LoginPage(driver);
        // ... тестовая логика
    }
}
```

---

### 3. Полиморфизм (Polymorphism)

Полиморфизм позволяет использовать объекты разных классов через единый интерфейс.
В тестовой автоматизации это ключ к созданию гибких фреймворков.

```java
/**
 * Интерфейс для отправки уведомлений о результатах тестов.
 * Полиморфизм: один интерфейс — разные реализации.
 */
public interface TestReporter {
    void reportResult(String testName, boolean passed, String details);
}

public class SlackReporter implements TestReporter {
    @Override
    public void reportResult(String testName, boolean passed, String details) {
        // Отправляем уведомление в Slack
        String emoji = passed ? ":white_check_mark:" : ":x:";
        slackClient.sendMessage("#qa-results",
                emoji + " " + testName + ": " + details);
    }
}

public class EmailReporter implements TestReporter {
    @Override
    public void reportResult(String testName, boolean passed, String details) {
        // Отправляем email
        emailService.send("qa@company.com",
                "Test Result: " + testName,
                "Status: " + (passed ? "PASSED" : "FAILED") + "\n" + details);
    }
}

public class ConsoleReporter implements TestReporter {
    @Override
    public void reportResult(String testName, boolean passed, String details) {
        System.out.printf("[%s] %s: %s%n",
                passed ? "PASS" : "FAIL", testName, details);
    }
}
```

```java
// Использование — код работает с любым репортером
public class TestRunner {
    // Список репортеров — полиморфизм в действии
    private final List<TestReporter> reporters;

    public TestRunner(List<TestReporter> reporters) {
        this.reporters = reporters;
    }

    public void onTestFinished(String testName, boolean passed, String details) {
        // Каждый репортер обрабатывает результат по-своему
        for (TestReporter reporter : reporters) {
            reporter.reportResult(testName, passed, details);
        }
    }
}
```

---

### 4. Абстракция (Abstraction)

Абстракция — выделение существенных характеристик объекта и отбрасывание несущественных деталей.
Абстрактные классы и интерфейсы — основные инструменты абстракции в Java.

```java
/**
 * Абстракция API-клиента.
 * Тесты работают с абстракцией, не зная деталей HTTP.
 */
public interface ApiClient {
    Response get(String endpoint);
    Response post(String endpoint, Object body);
    Response put(String endpoint, Object body);
    Response delete(String endpoint);
}

// Реализация через RestAssured
public class RestAssuredApiClient implements ApiClient {
    private final RequestSpecification spec;

    public RestAssuredApiClient(String baseUrl, String token) {
        this.spec = given()
                .baseUri(baseUrl)
                .header("Authorization", "Bearer " + token)
                .contentType(ContentType.JSON);
    }

    @Override
    public Response get(String endpoint) {
        return spec.get(endpoint);
    }

    @Override
    public Response post(String endpoint, Object body) {
        return spec.body(body).post(endpoint);
    }

    // ... остальные методы
}

// Тест не зависит от конкретной HTTP-библиотеки
@Test
void testCreateUser(ApiClient client) {
    var user = new User("John", "john@test.com");
    Response response = client.post("/users", user);
    Assertions.assertEquals(201, response.getStatusCode());
}
```

---

## Abstract class vs Interface

| Критерий                  | Abstract class                    | Interface                          |
|---------------------------|-----------------------------------|------------------------------------|
| Множественное наследование| Только один                       | Несколько интерфейсов              |
| Конструктор               | Может иметь                       | Не может                          |
| Поля                      | Любые (в т.ч. non-static, non-final)| Только `public static final`   |
| Методы                    | Абстрактные и конкретные          | `default`, `static`, абстрактные   |
| Модификаторы доступа      | Любые                             | Методы `public` по умолчанию       |
| Когда использовать        | Общее состояние + поведение       | Контракт без состояния             |

### Пример: когда что выбирать

```java
// Interface — контракт "что делать" (без состояния)
public interface Clickable {
    void click();
    boolean isEnabled();
}

// Abstract class — общее состояние + частичная реализация
public abstract class BaseComponent implements Clickable {
    protected final WebDriver driver;
    protected final By locator;

    public BaseComponent(WebDriver driver, By locator) {
        this.driver = driver;
        this.locator = locator;
    }

    @Override
    public void click() {
        driver.findElement(locator).click();
    }

    @Override
    public boolean isEnabled() {
        return driver.findElement(locator).isEnabled();
    }

    // Абстрактный метод — подклассы реализуют сами
    public abstract String getValue();
}

public class Button extends BaseComponent {
    public Button(WebDriver driver, By locator) {
        super(driver, locator);
    }

    @Override
    public String getValue() {
        return driver.findElement(locator).getText();
    }
}

public class InputField extends BaseComponent {
    public InputField(WebDriver driver, By locator) {
        super(driver, locator);
    }

    @Override
    public String getValue() {
        return driver.findElement(locator).getAttribute("value");
    }
}
```

---

## Принципы SOLID в автоматизации тестирования

### S — Single Responsibility Principle (SRP)

**Каждый класс должен иметь одну причину для изменения.**

```java
// ПЛОХО: Page Object отвечает за UI, данные и assertions
public class LoginPageBad {
    public void login(String user, String pass) { /* ... */ }
    public User getUserFromDb(String user) { /* ... */ }    // Не его дело!
    public void assertLoginSuccess() { /* ... */ }           // Не его дело!
}

// ХОРОШО: каждый класс — одна ответственность
public class LoginPage {       // Только взаимодействие с UI
    public DashboardPage loginAs(String user, String pass) { /* ... */ }
    public String getErrorMessage() { /* ... */ }
}

public class UserRepository {  // Только работа с данными
    public User findByUsername(String username) { /* ... */ }
}

// Assertions — в самом тесте
@Test
void testLoginSuccess() {
    DashboardPage dashboard = loginPage.loginAs("admin", "pass");
    Assertions.assertTrue(dashboard.isDisplayed());
}
```

### O — Open/Closed Principle (OCP)

**Класс должен быть открыт для расширения, но закрыт для модификации.**

```java
// Базовый класс для шагов теста — закрыт для модификации
public abstract class TestStep {
    private final String description;

    protected TestStep(String description) {
        this.description = description;
    }

    // Template Method
    public final void execute() {
        logStart();
        performAction();  // Точка расширения
        logEnd();
    }

    protected abstract void performAction();

    private void logStart() {
        System.out.println("Начало шага: " + description);
    }

    private void logEnd() {
        System.out.println("Конец шага: " + description);
    }
}

// Расширяем без модификации базового класса
public class ClickStep extends TestStep {
    private final WebElement element;

    public ClickStep(WebElement element) {
        super("Клик по элементу: " + element);
        this.element = element;
    }

    @Override
    protected void performAction() {
        element.click();
    }
}

public class TypeStep extends TestStep {
    private final WebElement element;
    private final String text;

    public TypeStep(WebElement element, String text) {
        super("Ввод текста: " + text);
        this.element = element;
        this.text = text;
    }

    @Override
    protected void performAction() {
        element.clear();
        element.sendKeys(text);
    }
}
```

### L — Liskov Substitution Principle (LSP)

**Объекты подклассов должны быть заменяемы на объекты базового класса без нарушения корректности.**

```java
// Базовый класс для данных
public class TestUser {
    protected String username;
    protected String password;

    public TestUser(String username, String password) {
        this.username = username;
        this.password = password;
    }

    public boolean canLogin() {
        return username != null && password != null;
    }
}

// ПРАВИЛЬНО: подкласс расширяет, не нарушая контракт
public class AdminUser extends TestUser {
    private final List<String> permissions;

    public AdminUser(String username, String password, List<String> permissions) {
        super(username, password);
        this.permissions = permissions;
    }

    public boolean hasPermission(String permission) {
        return permissions.contains(permission);
    }

    // canLogin() работает так же, как в базовом классе
}

// Тест работает с любым TestUser
@Test
void testAnyUserCanLogin(TestUser user) {
    if (user.canLogin()) {
        loginPage.loginAs(user.getUsername(), user.getPassword());
    }
}
```

### I — Interface Segregation Principle (ISP)

**Клиенты не должны зависеть от интерфейсов, которые они не используют.**

```java
// ПЛОХО: один жирный интерфейс
public interface TestFramework {
    void runUiTest();
    void runApiTest();
    void runPerformanceTest();
    void generateReport();
    void sendNotification();
}

// ХОРОШО: маленькие специализированные интерфейсы
public interface UiTestRunner {
    void runUiTest();
}

public interface ApiTestRunner {
    void runApiTest();
}

public interface ReportGenerator {
    void generateReport();
}

public interface Notifier {
    void sendNotification();
}

// Класс реализует только нужные интерфейсы
public class ApiTestSuite implements ApiTestRunner, ReportGenerator {
    @Override
    public void runApiTest() { /* ... */ }

    @Override
    public void generateReport() { /* ... */ }
}
```

### D — Dependency Inversion Principle (DIP)

**Модули верхнего уровня не должны зависеть от модулей нижнего уровня. Оба должны зависеть от абстракций.**

```java
// Абстракция — интерфейс хранилища тестовых данных
public interface TestDataProvider {
    Map<String, String> getTestData(String testCase);
}

// Реализация: данные из JSON-файла
public class JsonTestDataProvider implements TestDataProvider {
    @Override
    public Map<String, String> getTestData(String testCase) {
        // Чтение из JSON
        return readFromJson(testCase);
    }
}

// Реализация: данные из базы данных
public class DbTestDataProvider implements TestDataProvider {
    @Override
    public Map<String, String> getTestData(String testCase) {
        // Чтение из БД
        return readFromDatabase(testCase);
    }
}

// Тест зависит от абстракции, а не от конкретной реализации
public class OrderTest {
    private final TestDataProvider dataProvider;

    // Внедрение зависимости через конструктор
    public OrderTest(TestDataProvider dataProvider) {
        this.dataProvider = dataProvider;
    }

    @Test
    void testCreateOrder() {
        Map<String, String> data = dataProvider.getTestData("create_order");
        // ... тестовая логика
    }
}
```

---

## Связь с тестированием

| Концепция        | Применение в QA                                                 |
|------------------|-----------------------------------------------------------------|
| Инкапсуляция     | Page Object скрывает локаторы, тесты работают с методами        |
| Наследование     | BasePage, BaseTest — общая логика в родителе                    |
| Полиморфизм     | Разные репортеры, драйверы, провайдеры данных через интерфейс   |
| Абстракция       | API-клиенты, компоненты UI — скрытие деталей реализации         |
| SRP              | Page Object — только UI, тест — только логика проверки          |
| OCP              | Новые шаги/стратегии без изменения существующего кода           |
| LSP              | Подклассы Page Object не ломают тесты, использующие родителя    |
| ISP              | Отдельные интерфейсы для runner, reporter, data provider        |
| DIP              | Тесты зависят от абстракций, а не от конкретных реализаций      |

---

## Типичные ошибки

1. **Бог-объект (God Object)** — один Page Object на 2000 строк, который делает всё.
   Разделяйте на компоненты: Header, Footer, Sidebar.

2. **Глубокая иерархия наследования** — `BasePage -> AuthPage -> AdminPage -> SuperAdminPage`.
   Предпочитайте композицию: `AdminPage` содержит `AuthComponent`.

3. **Нарушение инкапсуляции** — публичные локаторы в Page Object.
   Тест не должен знать про `By.id("submit-btn")`.

4. **Игнорирование SOLID** — изменение конфигурации требует правок в десятках тестов.
   Используйте dependency injection и абстракции.

5. **Наследование вместо композиции** — `LoginTest extends LoginPage`.
   Тест использует Page Object, а не наследуется от него.

6. **Пустые интерфейсы-маркеры без необходимости** — создание интерфейсов ради интерфейсов.
   Интерфейс должен определять полезный контракт.

7. **Забытый `@Override`** — без аннотации опечатка в имени метода создаёт новый метод
   вместо переопределения. Всегда используйте `@Override`.

---

## Вопросы на интервью

### 🟢 Базовый уровень

1. Назовите четыре принципа ООП и кратко объясните каждый.
2. Что такое Page Object Model? Как он использует инкапсуляцию?
3. Чем абстрактный класс отличается от интерфейса?
4. Что такое наследование? Зачем нужен `BaseTest` в тестовом фреймворке?
5. Что означает ключевое слово `super`?
6. Можно ли создать экземпляр абстрактного класса?

### 🟡 Средний уровень

7. Расшифруйте SOLID. Приведите пример SRP в тестовом проекте.
8. Что такое полиморфизм? Покажите пример в контексте автоматизации.
9. Чем композиция отличается от наследования? Когда что выбирать?
10. Объясните DIP на примере TestDataProvider.
11. Что такое `default`-методы в интерфейсах? Когда их использовать?
12. Как OCP помогает расширять тестовый фреймворк?

### 🔴 Продвинутый уровень

13. Как нарушение LSP может сломать тесты? Приведите конкретный пример.
14. Объясните, как реализовать Strategy Pattern для выбора среды тестирования.
15. Как ISP применяется при проектировании API тестового фреймворка?
16. Чем `sealed` классы (Java 17+) полезны в тестовом коде?
17. Спроектируйте иерархию классов для кросс-браузерного тестового фреймворка,
    соблюдая все принципы SOLID.

---

## Практические задания

### Задание 1: Page Object с инкапсуляцией
Создайте Page Object для страницы регистрации с полями: имя, email, пароль, подтверждение пароля,
чекбокс "Согласен с условиями", кнопка "Зарегистрироваться". Все локаторы — приватные.
Публичные методы должны формировать fluent API (цепочка вызовов).

### Задание 2: Наследование в тестовом фреймворке
Создайте иерархию: `BasePage` -> `AuthenticatedPage` -> `AdminDashboardPage`.
`BasePage` — общие методы. `AuthenticatedPage` — проверка авторизации, header.
`AdminDashboardPage` — специфичные методы админки.

### Задание 3: SOLID-рефакторинг
Дан класс, нарушающий все принципы SOLID. Проведите рефакторинг:

```java
// Рефакторинг этого класса
public class TestHelper {
    WebDriver driver = new ChromeDriver();

    public void loginAndCheckOrder(String user, String pass, String orderId) {
        driver.get("http://localhost/login");
        driver.findElement(By.id("user")).sendKeys(user);
        driver.findElement(By.id("pass")).sendKeys(pass);
        driver.findElement(By.id("login-btn")).click();
        driver.get("http://localhost/orders/" + orderId);
        String status = driver.findElement(By.id("status")).getText();
        assert status.equals("completed");
        driver.quit();
    }
}
```

### Задание 4: Strategy Pattern для тестовых данных
Реализуйте `TestDataProvider` с тремя стратегиями: `JsonProvider`, `CsvProvider`, `InMemoryProvider`.
Напишите тесты, которые работают одинаково с любым провайдером.

---

## Дополнительные ресурсы

- [Oracle Java Tutorial: OOP Concepts](https://docs.oracle.com/javase/tutorial/java/concepts/)
- [Baeldung: SOLID Principles in Java](https://www.baeldung.com/solid-principles)
- [Selenium Page Object Model](https://www.selenium.dev/documentation/test_practices/encouraged/page_object_models/)
- [Head First Design Patterns](https://www.oreilly.com/library/view/head-first-design/9781492077992/)
- [Clean Code, Robert C. Martin](https://www.oreilly.com/library/view/clean-code-a/9780136083238/)
- [Effective Java, Joshua Bloch — Chapter 4: Classes and Interfaces](https://www.oreilly.com/library/view/effective-java/9780134686097/)
