# SOLID для QA

## Обзор

SOLID — это пять принципов объектно-ориентированного проектирования, сформулированных
Robert C. Martin (Uncle Bob). Эти принципы помогают создавать код, который легко поддерживать,
расширять и тестировать.

Для QA Automation Engineer SOLID — не абстрактная теория. Это практическое руководство по
проектированию тестового фреймворка, которое определяет:

- Как организовать Page Object-ы.
- Как структурировать тестовые классы.
- Как проектировать утилиты и хелперы.
- Как обеспечить расширяемость без переписывания.

| Буква | Принцип                                  | Суть одной фразой                                   |
|-------|------------------------------------------|------------------------------------------------------|
| **S** | Single Responsibility Principle (SRP)    | У класса одна причина для изменения                  |
| **O** | Open/Closed Principle (OCP)              | Открыт для расширения, закрыт для модификации        |
| **L** | Liskov Substitution Principle (LSP)      | Подтипы должны быть взаимозаменяемы с базовым типом  |
| **I** | Interface Segregation Principle (ISP)    | Много специализированных интерфейсов лучше одного    |
| **D** | Dependency Inversion Principle (DIP)     | Зависьте от абстракций, а не от конкретных классов   |

---

## S — Single Responsibility Principle

### Формулировка

> У класса должна быть **одна и только одна причина для изменения**.

Класс отвечает за одну задачу. Если класс делает несколько несвязанных вещей — его нужно
разделить.

### Нарушение SRP в тестах

```java
/**
 * ПЛОХО: Page Object отвечает за всё — элементы, действия, навигацию, верификацию.
 * Причин для изменения слишком много: изменился UI, изменилась бизнес-логика,
 * изменились проверки.
 */
public class LoginPage {

    private WebDriver driver;

    public LoginPage(WebDriver driver) {
        this.driver = driver;
    }

    // Навигация
    public void open() {
        driver.get("http://localhost:8080/login");
    }

    // Действия
    public void enterUsername(String username) {
        driver.findElement(By.id("username")).sendKeys(username);
    }

    public void enterPassword(String password) {
        driver.findElement(By.id("password")).sendKeys(password);
    }

    public void clickLogin() {
        driver.findElement(By.id("loginBtn")).click();
    }

    // Верификация — НЕ ДОЛЖНА быть здесь
    public boolean isErrorDisplayed() {
        return driver.findElement(By.id("error")).isDisplayed();
    }

    public String getErrorText() {
        return driver.findElement(By.id("error")).getText();
    }

    // Работа с cookies — НЕ ДОЛЖНА быть здесь
    public void clearCookies() {
        driver.manage().deleteAllCookies();
    }

    // Генерация тестовых данных — ТОЧНО НЕ ДОЛЖНА быть здесь
    public String generateRandomEmail() {
        return "test_" + System.currentTimeMillis() + "@test.com";
    }
}
```

### Соблюдение SRP

```java
/**
 * ХОРОШО: Page Object отвечает только за элементы и действия на странице.
 * Одна причина для изменения: UI страницы логина изменился.
 */
public class LoginPage {

    private final WebDriver driver;

    // Локаторы — описывают структуру страницы
    private static final By USERNAME_FIELD = By.id("username");
    private static final By PASSWORD_FIELD = By.id("password");
    private static final By LOGIN_BUTTON = By.id("loginBtn");
    private static final By ERROR_MESSAGE = By.id("error");

    public LoginPage(WebDriver driver) {
        this.driver = driver;
    }

    /**
     * Выполняет действие логина и возвращает следующую страницу.
     */
    public DashboardPage loginAs(String username, String password) {
        driver.findElement(USERNAME_FIELD).sendKeys(username);
        driver.findElement(PASSWORD_FIELD).sendKeys(password);
        driver.findElement(LOGIN_BUTTON).click();
        return new DashboardPage(driver);
    }

    /**
     * Выполняет попытку логина с невалидными данными.
     * Возвращает текущую страницу (остаёмся на логине).
     */
    public LoginPage loginWithInvalidCredentials(String username, String password) {
        driver.findElement(USERNAME_FIELD).sendKeys(username);
        driver.findElement(PASSWORD_FIELD).sendKeys(password);
        driver.findElement(LOGIN_BUTTON).click();
        return this;
    }

    public String getErrorMessage() {
        return driver.findElement(ERROR_MESSAGE).getText();
    }

    public boolean isErrorDisplayed() {
        return driver.findElement(ERROR_MESSAGE).isDisplayed();
    }
}

/**
 * Отдельный класс — генерация тестовых данных (своя ответственность).
 */
public class TestDataGenerator {

    public static String randomEmail() {
        return "test_" + UUID.randomUUID().toString().substring(0, 8) + "@test.com";
    }

    public static String randomUsername() {
        return "user_" + ThreadLocalRandom.current().nextInt(10000, 99999);
    }
}

/**
 * Отдельный класс — навигация (своя ответственность).
 */
public class Navigator {

    private final WebDriver driver;
    private final String baseUrl;

    public Navigator(WebDriver driver, String baseUrl) {
        this.driver = driver;
        this.baseUrl = baseUrl;
    }

    public LoginPage openLoginPage() {
        driver.get(baseUrl + "/login");
        return new LoginPage(driver);
    }

    public DashboardPage openDashboard() {
        driver.get(baseUrl + "/dashboard");
        return new DashboardPage(driver);
    }
}
```

---

## O — Open/Closed Principle

### Формулировка

> Программные сущности должны быть **открыты для расширения**, но **закрыты для модификации**.

Новую функциональность добавляем через расширение (наследование, реализацию интерфейсов),
а не через изменение существующего кода.

### Нарушение OCP

```java
/**
 * ПЛОХО: Добавление нового типа отчёта требует модификации существующего метода.
 * Каждый новый формат — изменение кода, риск сломать существующее.
 */
public class TestReporter {

    public void generateReport(List<TestResult> results, String format) {
        // При добавлении нового формата нужно менять этот метод
        switch (format) {
            case "html" -> generateHtmlReport(results);
            case "json" -> generateJsonReport(results);
            case "xml"  -> generateXmlReport(results);
            // Каждый новый формат — изменение этого класса
            // case "pdf" -> ...
            default -> throw new IllegalArgumentException("Неизвестный формат: " + format);
        }
    }

    private void generateHtmlReport(List<TestResult> results) { /* ... */ }
    private void generateJsonReport(List<TestResult> results) { /* ... */ }
    private void generateXmlReport(List<TestResult> results) { /* ... */ }
}
```

### Соблюдение OCP

```java
/**
 * ХОРОШО: Интерфейс для генерации отчётов.
 * Новый формат = новый класс, существующий код не меняется.
 */
public interface ReportGenerator {
    void generate(List<TestResult> results, Path outputPath);
    String getFormat();
}

public class HtmlReportGenerator implements ReportGenerator {

    @Override
    public void generate(List<TestResult> results, Path outputPath) {
        // Генерация HTML-отчёта
        var html = new StringBuilder("<!DOCTYPE html><html><body>");
        html.append("<h1>Отчёт о тестировании</h1>");
        html.append("<table><tr><th>Тест</th><th>Статус</th><th>Время</th></tr>");

        for (var result : results) {
            html.append("<tr><td>%s</td><td>%s</td><td>%d мс</td></tr>"
                .formatted(result.name(), result.status(), result.durationMs()));
        }

        html.append("</table></body></html>");

        try {
            Files.writeString(outputPath, html.toString());
        } catch (IOException e) {
            throw new UncheckedIOException("Ошибка записи HTML-отчёта", e);
        }
    }

    @Override
    public String getFormat() {
        return "html";
    }
}

public class JsonReportGenerator implements ReportGenerator {

    private final ObjectMapper mapper = new ObjectMapper();

    @Override
    public void generate(List<TestResult> results, Path outputPath) {
        try {
            mapper.writerWithDefaultPrettyPrinter()
                .writeValue(outputPath.toFile(), results);
        } catch (IOException e) {
            throw new UncheckedIOException("Ошибка записи JSON-отчёта", e);
        }
    }

    @Override
    public String getFormat() {
        return "json";
    }
}

/**
 * Новый формат — просто новый класс. Существующий код не трогаем.
 */
public class CsvReportGenerator implements ReportGenerator {

    @Override
    public void generate(List<TestResult> results, Path outputPath) {
        var csv = new StringBuilder("Тест,Статус,Время (мс)\n");
        for (var result : results) {
            csv.append("%s,%s,%d\n"
                .formatted(result.name(), result.status(), result.durationMs()));
        }
        try {
            Files.writeString(outputPath, csv.toString());
        } catch (IOException e) {
            throw new UncheckedIOException("Ошибка записи CSV-отчёта", e);
        }
    }

    @Override
    public String getFormat() {
        return "csv";
    }
}

/**
 * Реестр генераторов — автоматически находит все реализации.
 */
public class TestReporter {

    private final Map<String, ReportGenerator> generators = new HashMap<>();

    public TestReporter(List<ReportGenerator> generators) {
        for (var gen : generators) {
            this.generators.put(gen.getFormat(), gen);
        }
    }

    public void generateReport(List<TestResult> results, String format, Path output) {
        var generator = generators.get(format);
        if (generator == null) {
            throw new IllegalArgumentException("Нет генератора для формата: " + format);
        }
        generator.generate(results, output);
    }
}
```

---

## L — Liskov Substitution Principle

### Формулировка

> Объекты в программе можно **заменить их наследниками** без изменения корректности программы.

Подкласс должен полностью соответствовать контракту базового класса.

### Нарушение LSP

```java
/**
 * ПЛОХО: Подкласс нарушает контракт базового Page Object.
 */
public abstract class BasePage {
    protected WebDriver driver;

    public BasePage(WebDriver driver) {
        this.driver = driver;
    }

    /**
     * Контракт: возвращает заголовок страницы.
     */
    public abstract String getPageTitle();

    /**
     * Контракт: навигация — открывает URL страницы.
     */
    public abstract void navigateTo();
}

/**
 * Нормальная реализация — соблюдает контракт.
 */
public class HomePage extends BasePage {

    public HomePage(WebDriver driver) {
        super(driver);
    }

    @Override
    public String getPageTitle() {
        return driver.findElement(By.tagName("h1")).getText();
    }

    @Override
    public void navigateTo() {
        driver.get("http://localhost:8080/");
    }
}

/**
 * ПЛОХАЯ реализация — нарушает контракт.
 * Popup не имеет URL, поэтому navigateTo() бросает исключение.
 */
public class ConfirmationPopup extends BasePage {

    public ConfirmationPopup(WebDriver driver) {
        super(driver);
    }

    @Override
    public String getPageTitle() {
        return driver.findElement(By.className("popup-title")).getText();
    }

    @Override
    public void navigateTo() {
        // Нарушение LSP: popup нельзя открыть по URL
        throw new UnsupportedOperationException("Popup не имеет URL");
    }
}
```

### Соблюдение LSP

```java
/**
 * ХОРОШО: Разделяем абстракции — страница и компонент.
 */
public interface UiComponent {
    /** Возвращает заголовок компонента. */
    String getTitle();
    /** Проверяет, отображается ли компонент. */
    boolean isDisplayed();
}

/**
 * Навигируемая страница — расширение UiComponent.
 */
public interface NavigablePage extends UiComponent {
    /** Открывает страницу по URL. */
    void navigateTo();
    /** Возвращает URL страницы. */
    String getUrl();
}

/**
 * Домашняя страница — соблюдает контракт NavigablePage.
 */
public class HomePage implements NavigablePage {

    private final WebDriver driver;

    public HomePage(WebDriver driver) {
        this.driver = driver;
    }

    @Override
    public String getTitle() {
        return driver.findElement(By.tagName("h1")).getText();
    }

    @Override
    public boolean isDisplayed() {
        return driver.findElement(By.id("home-content")).isDisplayed();
    }

    @Override
    public void navigateTo() {
        driver.get("http://localhost:8080/");
    }

    @Override
    public String getUrl() {
        return "/";
    }
}

/**
 * Popup — реализует только UiComponent, без navigateTo().
 * Не нарушает LSP, потому что не обещает навигацию.
 */
public class ConfirmationPopup implements UiComponent {

    private final WebDriver driver;

    public ConfirmationPopup(WebDriver driver) {
        this.driver = driver;
    }

    @Override
    public String getTitle() {
        return driver.findElement(By.className("popup-title")).getText();
    }

    @Override
    public boolean isDisplayed() {
        return driver.findElement(By.className("popup")).isDisplayed();
    }

    public void confirm() {
        driver.findElement(By.id("confirm-btn")).click();
    }

    public void cancel() {
        driver.findElement(By.id("cancel-btn")).click();
    }
}
```

### Использование в тестах

```java
class NavigationTest {

    /**
     * Метод принимает NavigablePage — любая реализация подходит.
     * ConfirmationPopup сюда не попадёт — он не NavigablePage.
     */
    void verifyPageLoads(NavigablePage page) {
        page.navigateTo();
        assertTrue(page.isDisplayed(),
            "Страница " + page.getUrl() + " должна загрузиться");
    }

    @Test
    void allPagesShouldLoad() {
        // Все NavigablePage взаимозаменяемы — LSP соблюдён
        List<NavigablePage> pages = List.of(
            new HomePage(driver),
            new ProfilePage(driver),
            new SettingsPage(driver)
        );

        for (var page : pages) {
            verifyPageLoads(page);
        }
    }
}
```

---

## I — Interface Segregation Principle

### Формулировка

> Клиенты не должны зависеть от интерфейсов, которые они **не используют**.

Лучше иметь много маленьких специализированных интерфейсов, чем один большой.

### Нарушение ISP

```java
/**
 * ПЛОХО: "толстый" интерфейс. Не каждый тест использует все возможности.
 * API-тесту не нужен takeScreenshot(), UI-тесту не нужен executeQuery().
 */
public interface TestFramework {
    void openBrowser(String browser);
    void closeBrowser();
    void navigateTo(String url);
    void click(By locator);
    void type(By locator, String text);
    byte[] takeScreenshot();
    Response sendHttpRequest(String method, String url, String body);
    ResultSet executeQuery(String sql);
    void sendMessage(String channel, String message); // Slack-уведомления
}
```

### Соблюдение ISP

```java
/**
 * ХОРОШО: Разделённые интерфейсы — каждый клиент зависит только от того, что использует.
 */

/** Интерфейс для управления браузером. */
public interface BrowserActions {
    void openBrowser(String browser);
    void closeBrowser();
    void navigateTo(String url);
}

/** Интерфейс для взаимодействия с UI-элементами. */
public interface ElementActions {
    void click(By locator);
    void type(By locator, String text);
    String getText(By locator);
    boolean isDisplayed(By locator);
}

/** Интерфейс для снятия скриншотов. */
public interface Screenshotable {
    byte[] takeScreenshot();
    byte[] takeElementScreenshot(By locator);
}

/** Интерфейс для HTTP-запросов. */
public interface HttpActions {
    Response get(String url, Map<String, String> headers);
    Response post(String url, String body, Map<String, String> headers);
    Response put(String url, String body, Map<String, String> headers);
    Response delete(String url, Map<String, String> headers);
}

/** Интерфейс для работы с БД. */
public interface DatabaseActions {
    ResultSet executeQuery(String sql);
    int executeUpdate(String sql);
}

/** Интерфейс для уведомлений. */
public interface Notifier {
    void sendNotification(String channel, String message);
}
```

### Использование сегрегированных интерфейсов

```java
/**
 * UI-тест зависит только от нужных интерфейсов.
 */
class LoginUiTest {

    // Зависим только от того, что используем
    private final BrowserActions browser;
    private final ElementActions elements;
    private final Screenshotable screenshots;

    LoginUiTest(BrowserActions browser, ElementActions elements,
                Screenshotable screenshots) {
        this.browser = browser;
        this.elements = elements;
        this.screenshots = screenshots;
    }

    void testLogin() {
        browser.navigateTo("/login");
        elements.type(By.id("username"), "admin");
        elements.type(By.id("password"), "secret");
        elements.click(By.id("loginBtn"));
        // Используем только нужные интерфейсы
    }
}

/**
 * API-тест зависит только от HttpActions.
 * Не знает о браузере и скриншотах.
 */
class UserApiTest {

    private final HttpActions http;

    UserApiTest(HttpActions http) {
        this.http = http;
    }

    void testGetUser() {
        var response = http.get("/api/users/1", Map.of());
        assertEquals(200, response.statusCode());
    }
}

/**
 * Конкретная реализация может реализовать несколько интерфейсов.
 */
public class SeleniumDriver implements BrowserActions, ElementActions, Screenshotable {

    private WebDriver driver;

    @Override
    public void openBrowser(String browser) {
        this.driver = switch (browser) {
            case "chrome"  -> new ChromeDriver();
            case "firefox" -> new FirefoxDriver();
            default -> throw new IllegalArgumentException("Неизвестный браузер: " + browser);
        };
    }

    @Override
    public void closeBrowser() { driver.quit(); }

    @Override
    public void navigateTo(String url) { driver.get(url); }

    @Override
    public void click(By locator) { driver.findElement(locator).click(); }

    @Override
    public void type(By locator, String text) { driver.findElement(locator).sendKeys(text); }

    @Override
    public String getText(By locator) { return driver.findElement(locator).getText(); }

    @Override
    public boolean isDisplayed(By locator) { return driver.findElement(locator).isDisplayed(); }

    @Override
    public byte[] takeScreenshot() {
        return ((TakesScreenshot) driver).getScreenshotAs(OutputType.BYTES);
    }

    @Override
    public byte[] takeElementScreenshot(By locator) {
        return driver.findElement(locator).getScreenshotAs(OutputType.BYTES);
    }
}
```

---

## D — Dependency Inversion Principle

### Формулировка

> Модули верхнего уровня не должны зависеть от модулей нижнего уровня.
> Оба должны зависеть от **абстракций**. Абстракции не должны зависеть от деталей.

### Нарушение DIP

```java
/**
 * ПЛОХО: Тест жёстко привязан к конкретным реализациям.
 * Нельзя подменить ChromeDriver на MockDriver.
 * Нельзя заменить RestAssured на другой HTTP-клиент.
 */
public class UserFlowTest {

    // Зависимость от конкретных классов — нарушение DIP
    private ChromeDriver driver = new ChromeDriver();
    private RestAssuredAdapter apiClient = new RestAssuredAdapter();
    private PostgresClient dbClient = new PostgresClient("jdbc:postgresql://...");

    @Test
    void shouldCreateUserViaApi() {
        // Тест привязан к Chrome, RestAssured, PostgreSQL
        // Невозможно запустить в другом окружении без изменения кода
    }
}
```

### Соблюдение DIP

```java
/**
 * ХОРОШО: Тест зависит от абстракций (интерфейсов).
 * Конкретные реализации инжектируются извне.
 */
public class UserFlowTest {

    // Зависимости от абстракций
    private final WebDriver driver;
    private final HttpClient apiClient;
    private final DatabaseClient dbClient;

    /**
     * Инъекция зависимостей через конструктор.
     */
    public UserFlowTest(WebDriver driver, HttpClient apiClient,
                        DatabaseClient dbClient) {
        this.driver = driver;     // Может быть Chrome, Firefox, Mock
        this.apiClient = apiClient; // Может быть RestAssured, OkHttp, Mock
        this.dbClient = dbClient;  // Может быть Postgres, H2, Mock
    }

    @Test
    void shouldCreateUserViaApi() {
        // Тест работает с абстракциями — реализация заменяема
        var response = apiClient.post("/api/users", userJson, headers);
        assertEquals(201, response.statusCode());

        // Проверяем через другую абстракцию
        var user = dbClient.findUserByEmail("test@test.com");
        assertNotNull(user);
    }
}
```

### Пример: DIP с WebDriver

```java
/**
 * Принцип DIP в действии: Selenium WebDriver — классический пример.
 * WebDriver — интерфейс (абстракция), ChromeDriver/FirefoxDriver — реализации.
 */

// ВСЕ тесты и Page Object-ы работают с интерфейсом WebDriver
public class LoginPage {

    // Зависимость от абстракции WebDriver, а не от ChromeDriver
    private final WebDriver driver;

    public LoginPage(WebDriver driver) {
        this.driver = driver; // Можно передать Chrome, Firefox, Remote, Mock
    }

    public DashboardPage loginAs(String username, String password) {
        driver.findElement(By.id("username")).sendKeys(username);
        driver.findElement(By.id("password")).sendKeys(password);
        driver.findElement(By.id("loginBtn")).click();
        return new DashboardPage(driver);
    }
}

/**
 * Провайдер конфигурации — собирает зависимости.
 */
public class TestContext {

    private static final ExecutionStrategy strategy =
        ExecutionStrategyFactory.getStrategy();

    // Фабричный метод — создаёт абстракцию, конкретика скрыта
    public static WebDriver createDriver() {
        return strategy.createDriver(
            ConfigHolder.getInstance().getProperty("browser"));
    }

    public static HttpClient createHttpClient() {
        return new RestAssuredAdapter(); // Легко заменить
    }

    public static DatabaseClient createDbClient() {
        String url = ConfigHolder.getInstance().getProperty("db.url");
        return new JdbcDatabaseClient(url);
    }
}
```

### DIP с JUnit 5 ParameterResolver

```java
/**
 * JUnit 5 Extension для инъекции зависимостей в тесты.
 */
public class TestContextExtension implements ParameterResolver {

    @Override
    public boolean supportsParameter(ParameterContext paramCtx,
                                      ExtensionContext extCtx) {
        Class<?> type = paramCtx.getParameter().getType();
        return type == WebDriver.class
            || type == HttpClient.class
            || type == DatabaseClient.class;
    }

    @Override
    public Object resolveParameter(ParameterContext paramCtx,
                                    ExtensionContext extCtx) {
        Class<?> type = paramCtx.getParameter().getType();

        if (type == WebDriver.class) {
            return TestContext.createDriver();
        }
        if (type == HttpClient.class) {
            return TestContext.createHttpClient();
        }
        if (type == DatabaseClient.class) {
            return TestContext.createDbClient();
        }

        throw new IllegalArgumentException("Неподдерживаемый тип: " + type);
    }
}

/**
 * Тест получает зависимости через DIP — чистый и тестируемый.
 */
@ExtendWith(TestContextExtension.class)
class UserTest {

    @Test
    void shouldDisplayUserProfile(WebDriver driver) {
        // driver — абстракция; конкретная реализация определяется контекстом
        var loginPage = new LoginPage(driver);
        var dashboard = loginPage.loginAs("admin", "password");
        assertEquals("Dashboard", dashboard.getTitle());
    }

    @Test
    void shouldReturnUserViaApi(HttpClient client) {
        var response = client.get("/api/users/1", Map.of());
        assertEquals(200, response.statusCode());
    }
}
```

---

## Связь с тестированием

| Принцип | Как проявляется в тестировании                                              |
|---------|-----------------------------------------------------------------------------|
| **SRP** | Один Page Object — одна страница. Один тестовый класс — одна фича.          |
| **OCP** | Новый браузер/формат отчёта — новый класс, без изменения существующих.       |
| **LSP** | Любая реализация Page может использоваться в тесте без сюрпризов.           |
| **ISP** | API-тесту не нужен BrowserActions; UI-тесту не нужен DatabaseActions.        |
| **DIP** | Тесты зависят от WebDriver (интерфейс), а не от ChromeDriver (реализация). |

Соблюдение SOLID в тестовом коде:
- **Снижает стоимость поддержки** — изменения локализованы.
- **Упрощает расширение** — новый функционал не ломает существующий.
- **Облегчает тестирование самого фреймворка** — зависимости можно подменить моками.
- **Улучшает читаемость** — каждый класс имеет понятную ответственность.

---

## Типичные ошибки

1. **SRP: God Page Object** — один Page Object для страницы с 50 элементами и 30 методами.
   Лучше разделить на компоненты (HeaderComponent, SidebarComponent, ContentComponent).
2. **OCP: switch по типу в тестах** — вместо полиморфизма используются длинные switch/if-else
   для определения поведения.
3. **LSP: пустые реализации** — метод возвращает `null` или бросает `UnsupportedOperationException`
   вместо реального поведения.
4. **ISP: «божественный» интерфейс** — один интерфейс TestHelper с 20 методами, из которых
   каждый тест использует 2-3.
5. **DIP: `new` внутри теста** — `new ChromeDriver()` вместо получения драйвера из фабрики
   или через инъекцию зависимостей.
6. **Чрезмерное абстрагирование** — создание интерфейсов с единственной реализацией без
   перспективы расширения. SOLID — не самоцель, а инструмент.

---

## Вопросы на интервью

### Уровень Junior

- Что такое SOLID? Назовите все пять принципов.
- Что означает Single Responsibility в контексте Page Object?
- Почему тесты должны зависеть от WebDriver (интерфейс), а не от ChromeDriver (класс)?
- Что такое Dependency Inversion? Приведите простой пример.

### Уровень Middle

- Как принцип Open/Closed помогает при добавлении поддержки нового браузера?
- Как нарушение LSP проявляется в тестовом фреймворке? Приведите пример.
- Как разделить «толстый» интерфейс TestHelper, применив ISP?
- Как реализовать инъекцию зависимостей в JUnit 5 тесты?
- Как SRP влияет на организацию тестовых данных?

### Уровень Senior

- Как бы вы провели рефакторинг legacy-фреймворка, нарушающего все SOLID-принципы?
  С какого принципа начали бы?
- Когда SOLID-принципы конфликтуют друг с другом? Как расставить приоритеты?
- Как SOLID связан с тестируемостью кода? Как вы используете SOLID при code review?
- Как реализовать полноценный DI-контейнер для тестового фреймворка?
- Как принципы SOLID соотносятся с GoF-паттернами? Приведите примеры.

---

## Практические задания

1. **SRP: Рефакторинг Page Object.** Возьмите "толстый" Page Object (20+ методов) и разделите
   его на компоненты по SRP. Каждый компонент — отдельный класс с одной ответственностью.

2. **OCP: Расширяемый отчёт.** Создайте систему генерации отчётов с интерфейсом `ReportGenerator`.
   Реализуйте HTML, JSON, CSV форматы. Добавьте новый формат (XML) без изменения существующего кода.

3. **LSP: Иерархия Page.** Создайте интерфейсы `UiComponent`, `NavigablePage`, `FormPage`.
   Реализуйте 3-4 страницы, убедитесь, что каждая полностью соответствует контракту своего
   интерфейса. Напишите тест, который работает с коллекцией `NavigablePage`.

4. **ISP: Разделение интерфейсов.** Возьмите «толстый» интерфейс `TestFramework` с 15 методами.
   Разделите его на 4-5 специализированных интерфейсов. Покажите, что каждый тест зависит
   только от нужных интерфейсов.

5. **DIP: Инъекция зависимостей.** Перепишите тест, использующий `new ChromeDriver()` и
   `new RestAssuredAdapter()`, так, чтобы все зависимости приходили извне. Реализуйте
   JUnit 5 Extension для автоматической инъекции.

---

## Дополнительные ресурсы

- **"Clean Architecture"** — Robert C. Martin — глубокое погружение в SOLID.
- **"Agile Software Development: Principles, Patterns, and Practices"** — Robert C. Martin —
  оригинальное описание SOLID-принципов.
- **Baeldung — SOLID Principles in Java** — практические примеры на Java.
- **Refactoring Guru — SOLID** — визуальные объяснения каждого принципа.
- **"Growing Object-Oriented Software, Guided by Tests"** — Freeman, Pryce — TDD и SOLID.
- **Selenium WebDriver API** — эталонный пример DIP: интерфейс WebDriver и его реализации.
