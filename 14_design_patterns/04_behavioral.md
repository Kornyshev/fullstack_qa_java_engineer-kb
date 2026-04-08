# Поведенческие паттерны (Behavioral Patterns)

## Обзор

Поведенческие паттерны (Behavioral Patterns) определяют **взаимодействие между объектами**
и распределение обязанностей. Они описывают не только объекты и классы, но и способы
коммуникации между ними, позволяя создавать гибкие и расширяемые системы.

Для QA Automation Engineer наиболее важны четыре поведенческих паттерна:

- **Strategy** — переключение алгоритмов (стратегия запуска, конфигурация окружения).
- **Template Method** — определение скелета алгоритма с переопределением шагов (базовый тест).
- **Observer/Listener** — реакция на события (слушатели TestNG/JUnit).
- **Chain of Responsibility** — обработка запроса цепочкой обработчиков (фильтры, валидаторы).

---

## Strategy (Стратегия)

### Назначение

Strategy инкапсулирует семейство алгоритмов, делая их **взаимозаменяемыми**. Позволяет
изменять алгоритм независимо от клиентов, которые его используют.

### ASCII-диаграмма

```
┌──────────────────────┐       ┌──────────────────────┐
│      Context         │       │    Strategy           │
│  (TestRunner)        │──────>│    «interface»        │
│  - strategy          │       ├──────────────────────┤
│  + executeTests()    │       │  + setup(): void     │
└──────────────────────┘       │  + getDriver(): ...  │
                               └──────────┬───────────┘
                                          │
                          ┌───────────────┼───────────────┐
                          v               v               v
                   ┌────────────┐  ┌────────────┐  ┌────────────┐
                   │   Local    │  │   Remote   │  │    CI      │
                   │  Strategy  │  │  Strategy  │  │  Strategy  │
                   └────────────┘  └────────────┘  └────────────┘
```

### Применение в тестировании

1. **Стратегия запуска** — локальный, удалённый (Selenium Grid), облачный (BrowserStack).
2. **Стратегия окружения** — dev, staging, production.
3. **Стратегия авторизации** — Basic Auth, OAuth2, API Key.
4. **Стратегия генерации данных** — случайные, фиксированные, из файла.

### Пример: Стратегия запуска тестов

```java
/**
 * Интерфейс стратегии запуска.
 * Каждая реализация знает, как настроить окружение для тестирования.
 */
public interface ExecutionStrategy {
    WebDriver createDriver(String browser);
    String getBaseUrl();
    void tearDown(WebDriver driver);
}

/**
 * Стратегия локального запуска — браузер на машине разработчика.
 */
public class LocalExecutionStrategy implements ExecutionStrategy {

    @Override
    public WebDriver createDriver(String browser) {
        return switch (browser.toLowerCase()) {
            case "chrome" -> {
                var options = new ChromeOptions();
                options.addArguments("--start-maximized");
                yield new ChromeDriver(options);
            }
            case "firefox" -> {
                var options = new FirefoxOptions();
                yield new FirefoxDriver(options);
            }
            default -> throw new IllegalArgumentException(
                "Неподдерживаемый браузер для локального запуска: " + browser);
        };
    }

    @Override
    public String getBaseUrl() {
        return "http://localhost:8080";
    }

    @Override
    public void tearDown(WebDriver driver) {
        if (driver != null) {
            driver.quit();
        }
    }
}

/**
 * Стратегия удалённого запуска через Selenium Grid.
 */
public class RemoteExecutionStrategy implements ExecutionStrategy {

    private final String gridUrl;

    public RemoteExecutionStrategy(String gridUrl) {
        this.gridUrl = gridUrl;
    }

    @Override
    public WebDriver createDriver(String browser) {
        Capabilities caps = switch (browser.toLowerCase()) {
            case "chrome"  -> new ChromeOptions();
            case "firefox" -> new FirefoxOptions();
            default -> throw new IllegalArgumentException(
                "Неподдерживаемый браузер: " + browser);
        };

        try {
            return new RemoteWebDriver(new URL(gridUrl), caps);
        } catch (MalformedURLException e) {
            throw new RuntimeException("Невалидный URL Grid: " + gridUrl, e);
        }
    }

    @Override
    public String getBaseUrl() {
        return "https://staging.myapp.com";
    }

    @Override
    public void tearDown(WebDriver driver) {
        if (driver != null) {
            driver.quit();
        }
    }
}

/**
 * Стратегия для CI-окружения — headless-режим, специфические настройки.
 */
public class CiExecutionStrategy implements ExecutionStrategy {

    @Override
    public WebDriver createDriver(String browser) {
        var options = new ChromeOptions();
        options.addArguments(
            "--headless=new",
            "--no-sandbox",
            "--disable-dev-shm-usage",
            "--disable-gpu",
            "--window-size=1920,1080"
        );
        return new ChromeDriver(options);
    }

    @Override
    public String getBaseUrl() {
        // URL определяется переменной окружения в CI
        return System.getenv("TEST_BASE_URL");
    }

    @Override
    public void tearDown(WebDriver driver) {
        if (driver != null) {
            driver.quit();
        }
    }
}
```

### Выбор стратегии по конфигурации

```java
/**
 * Фабрика для выбора стратегии на основе системного свойства.
 */
public final class ExecutionStrategyFactory {

    private ExecutionStrategyFactory() {}

    public static ExecutionStrategy getStrategy() {
        String mode = System.getProperty("execution.mode", "local");

        return switch (mode.toLowerCase()) {
            case "local"  -> new LocalExecutionStrategy();
            case "remote" -> new RemoteExecutionStrategy(
                System.getProperty("grid.url", "http://localhost:4444/wd/hub"));
            case "ci"     -> new CiExecutionStrategy();
            default -> throw new IllegalArgumentException(
                "Неизвестный режим выполнения: " + mode);
        };
    }
}
```

### Использование в тестах

```java
class SearchTest {

    private WebDriver driver;
    private final ExecutionStrategy strategy = ExecutionStrategyFactory.getStrategy();

    @BeforeEach
    void setUp() {
        // Стратегия определяет, как создаётся драйвер и какой URL используется
        driver = strategy.createDriver("chrome");
        driver.get(strategy.getBaseUrl());
    }

    @AfterEach
    void tearDown() {
        strategy.tearDown(driver);
    }

    @Test
    void shouldFindProduct() {
        var searchPage = new SearchPage(driver);
        var results = searchPage.search("Java книга");
        assertFalse(results.isEmpty(), "Результаты поиска не должны быть пустыми");
    }
}
```

Запуск с разными стратегиями:
```bash
# Локально
mvn test -Dexecution.mode=local

# На Selenium Grid
mvn test -Dexecution.mode=remote -Dgrid.url=http://grid:4444/wd/hub

# В CI
mvn test -Dexecution.mode=ci
```

---

## Template Method (Шаблонный метод)

### Назначение

Template Method определяет **скелет алгоритма** в базовом классе, позволяя подклассам
переопределять отдельные шаги без изменения общей структуры.

### ASCII-диаграмма

```
┌────────────────────────────────────┐
│       AbstractBaseTest             │
│  (шаблонный метод)                 │
├────────────────────────────────────┤
│ + runTest()  ──────────────────┐   │
│   │ 1. setUp()         (hook) │   │
│   │ 2. prepareData()   (hook) │   │
│   │ 3. executeTest()   (abstract)│
│   │ 4. verifyResults() (abstract)│
│   │ 5. tearDown()      (hook) │   │
│   └───────────────────────────┘   │
└────────────────┬──────────────────┘
                 │
        ┌────────┴────────┐
        v                 v
┌──────────────┐   ┌──────────────┐
│   UITest     │   │   APITest    │
│ executeTest()│   │ executeTest()│
│ verifyRes()  │   │ verifyRes()  │
└──────────────┘   └──────────────┘
```

### Применение в тестировании

1. **Базовый класс теста** — общая логика setup/teardown, конкретные тесты переопределяют шаги.
2. **Шаблон отчёта** — общая структура, разный контент для разных типов тестов.
3. **Стандартный тестовый сценарий** — precondition, action, verification, cleanup.

### Пример: Базовый класс для UI-тестов

```java
/**
 * Абстрактный базовый класс для всех UI-тестов.
 * Определяет общий жизненный цикл теста.
 */
public abstract class BaseUiTest {

    protected WebDriver driver;
    protected SoftAssertions softly;

    /**
     * Шаблонный метод — определяет порядок шагов.
     * Final — подклассы не могут изменить порядок.
     */
    @BeforeEach
    final void baseSetUp() {
        // Шаг 1: Создание драйвера (общий для всех)
        driver = ExecutionStrategyFactory.getStrategy()
            .createDriver(getRequiredBrowser());
        driver.manage().timeouts().implicitlyWait(Duration.ofSeconds(5));

        // Шаг 2: Инициализация soft assertions
        softly = new SoftAssertions();

        // Шаг 3: Вызов хука подкласса для дополнительной настройки
        setUp();
    }

    @AfterEach
    final void baseTearDown() {
        // Шаг 1: Вызов хука подкласса для очистки
        tearDown();

        // Шаг 2: Скриншот при ошибке (общий для всех)
        // Шаг 3: Закрытие драйвера
        if (driver != null) {
            driver.quit();
        }

        // Шаг 4: Проверка soft assertions
        softly.assertAll();
    }

    /**
     * Хук для подкласса — дополнительная настройка.
     * По умолчанию ничего не делает.
     */
    protected void setUp() {
        // Подклассы могут переопределить
    }

    /**
     * Хук для подкласса — дополнительная очистка.
     */
    protected void tearDown() {
        // Подклассы могут переопределить
    }

    /**
     * Абстрактный метод — подкласс обязан указать браузер.
     */
    protected abstract String getRequiredBrowser();
}

/**
 * Конкретный тестовый класс для страницы логина.
 */
class LoginPageTest extends BaseUiTest {

    private LoginPage loginPage;

    @Override
    protected String getRequiredBrowser() {
        return "chrome"; // Данный тест требует Chrome
    }

    @Override
    protected void setUp() {
        // Дополнительная настройка — открываем страницу логина
        driver.get("http://localhost:8080/login");
        loginPage = new LoginPage(driver);
    }

    @Test
    void shouldShowErrorForInvalidCredentials() {
        loginPage.loginAs("wrong", "wrong");
        softly.assertThat(loginPage.getErrorMessage())
            .isEqualTo("Invalid credentials");
    }

    @Test
    void shouldLoginSuccessfully() {
        var dashboard = loginPage.loginAs("admin", "password");
        softly.assertThat(dashboard.getTitle())
            .isEqualTo("Dashboard");
    }
}
```

### Пример: Базовый класс для API-тестов

```java
/**
 * Шаблонный метод для API-тестов.
 */
public abstract class BaseApiTest {

    protected HttpClient apiClient;
    protected String authToken;

    @BeforeEach
    final void baseSetUp() {
        // Общая инициализация: создание клиента
        apiClient = createHttpClient();

        // Аутентификация (если нужна)
        if (requiresAuth()) {
            authToken = authenticate();
        }

        // Хук для подкласса
        setUp();
    }

    @AfterEach
    final void baseTearDown() {
        cleanUp();
    }

    /**
     * Подкласс определяет, нужна ли аутентификация.
     */
    protected boolean requiresAuth() {
        return true; // По умолчанию — да
    }

    /**
     * Подкласс может переопределить способ создания клиента.
     */
    protected HttpClient createHttpClient() {
        return new RestAssuredAdapter();
    }

    protected void setUp() {}
    protected void cleanUp() {}

    /**
     * Общая логика аутентификации.
     */
    private String authenticate() {
        var response = apiClient.post(
            getBaseUrl() + "/api/auth/login",
            """
            {"username": "%s", "password": "%s"}
            """.formatted(getTestUsername(), getTestPassword()),
            Map.of("Content-Type", "application/json")
        );
        // Извлечение токена из ответа
        return JsonPath.from(response.body()).getString("token");
    }

    protected abstract String getBaseUrl();
    protected String getTestUsername() { return "testuser"; }
    protected String getTestPassword() { return "testpass"; }

    /**
     * Утилита — добавляет заголовок авторизации.
     */
    protected Map<String, String> authHeaders() {
        return Map.of(
            "Authorization", "Bearer " + authToken,
            "Content-Type", "application/json"
        );
    }
}

/**
 * Конкретные API-тесты.
 */
class UserApiTest extends BaseApiTest {

    @Override
    protected String getBaseUrl() {
        return "http://localhost:8080";
    }

    @Test
    void shouldGetCurrentUser() {
        var response = apiClient.get(
            getBaseUrl() + "/api/users/me", authHeaders());
        assertEquals(200, response.statusCode());
    }
}
```

---

## Observer / Listener (Наблюдатель)

### Назначение

Observer определяет зависимость типа "один ко многим": при изменении состояния одного объекта
все зависимые **автоматически уведомляются и обновляются**.

### ASCII-диаграмма

```
┌──────────────────┐        ┌─────────────────────┐
│   TestRunner     │        │  TestListener        │
│  (Subject)       │───────>│  «interface»         │
│  - listeners[]   │  1..*  ├─────────────────────┤
│  + addListener() │        │ + onTestStart()      │
│  + notify()      │        │ + onTestSuccess()    │
└──────────────────┘        │ + onTestFailure()    │
                            └──────────┬──────────┘
                                       │
                          ┌────────────┼────────────┐
                          v            v            v
                   ┌────────────┐┌──────────┐┌────────────┐
                   │ Logging    ││Screenshot││ Allure     │
                   │ Listener   ││ Listener ││ Listener   │
                   └────────────┘└──────────┘└────────────┘
```

### Применение в тестировании

1. **TestNG Listeners** — `ITestListener`, `ISuiteListener`, `IInvokedMethodListener`.
2. **JUnit 5 Extensions** — `TestWatcher`, `BeforeEachCallback`, `AfterEachCallback`.
3. **Allure Reporting** — слушатели, собирающие данные для отчёта.
4. **Custom Listeners** — логирование, скриншоты при падении, уведомления в Slack.

### Пример: JUnit 5 Extension (Observer)

```java
/**
 * JUnit 5 Extension — аналог Observer.
 * Реагирует на события жизненного цикла тестов.
 */
public class ScreenshotOnFailureExtension implements TestWatcher, AfterTestExecutionCallback {

    private static final Logger log =
        LoggerFactory.getLogger(ScreenshotOnFailureExtension.class);

    /**
     * Вызывается после каждого теста.
     * Если тест провалился — сохраняем скриншот.
     */
    @Override
    public void afterTestExecution(ExtensionContext context) throws Exception {
        // Проверяем, было ли исключение
        context.getExecutionException().ifPresent(throwable -> {
            log.error("Тест упал: {}", context.getDisplayName(), throwable);
            takeScreenshot(context);
        });
    }

    @Override
    public void testFailed(ExtensionContext context, Throwable cause) {
        log.error("FAILED: {} — {}", context.getDisplayName(), cause.getMessage());
    }

    @Override
    public void testSuccessful(ExtensionContext context) {
        log.info("PASSED: {}", context.getDisplayName());
    }

    @Override
    public void testAborted(ExtensionContext context, Throwable cause) {
        log.warn("ABORTED: {}", context.getDisplayName());
    }

    @Override
    public void testDisabled(ExtensionContext context, Optional<String> reason) {
        log.info("DISABLED: {} — причина: {}",
            context.getDisplayName(), reason.orElse("не указана"));
    }

    private void takeScreenshot(ExtensionContext context) {
        // Получаем WebDriver из контекста теста
        context.getTestInstance().ifPresent(instance -> {
            try {
                var driverField = instance.getClass().getDeclaredField("driver");
                driverField.setAccessible(true);
                var driver = (WebDriver) driverField.get(instance);

                if (driver instanceof TakesScreenshot ts) {
                    byte[] screenshot = ts.getScreenshotAs(OutputType.BYTES);
                    String fileName = context.getDisplayName()
                        .replaceAll("[^a-zA-Z0-9]", "_") + ".png";
                    Path path = Path.of("target", "screenshots", fileName);
                    Files.createDirectories(path.getParent());
                    Files.write(path, screenshot);
                    log.info("Скриншот сохранён: {}", path);
                }
            } catch (Exception e) {
                log.warn("Не удалось сохранить скриншот: {}", e.getMessage());
            }
        });
    }
}
```

### Пример: TestNG Listener

```java
/**
 * TestNG Listener — классический Observer.
 * Автоматически добавляет информацию в Allure-отчёт.
 */
public class AllureTestListener implements ITestListener {

    @Override
    public void onTestStart(ITestResult result) {
        // Добавляем метаинформацию в отчёт Allure
        Allure.step("Начало теста: " + result.getName());
        Allure.label("thread", Thread.currentThread().getName());
    }

    @Override
    public void onTestSuccess(ITestResult result) {
        long duration = result.getEndMillis() - result.getStartMillis();
        Allure.step("Тест пройден за " + duration + " мс");
    }

    @Override
    public void onTestFailure(ITestResult result) {
        // Добавляем скриншот к отчёту
        Allure.addAttachment("Скриншот при падении", "image/png",
            captureScreenshot(), ".png");

        // Добавляем информацию об ошибке
        Allure.addAttachment("Ошибка", "text/plain",
            result.getThrowable().getMessage());
    }

    @Override
    public void onTestSkipped(ITestResult result) {
        Allure.step("Тест пропущен: " + result.getSkipMessage());
    }

    private byte[] captureScreenshot() {
        WebDriver driver = WebDriverManager.getDriver();
        if (driver instanceof TakesScreenshot ts) {
            return ts.getScreenshotAs(OutputType.BYTES);
        }
        return new byte[0];
    }
}
```

### Регистрация слушателей

```java
// JUnit 5 — через аннотацию
@ExtendWith(ScreenshotOnFailureExtension.class)
class MyTest { ... }

// TestNG — через аннотацию
@Listeners(AllureTestListener.class)
class MyTestNGTest { ... }

// TestNG — через testng.xml
// <listeners>
//     <listener class-name="com.example.AllureTestListener"/>
// </listeners>
```

---

## Chain of Responsibility (Цепочка обязанностей)

### Назначение

Chain of Responsibility позволяет **передавать запрос по цепочке обработчиков**. Каждый
обработчик решает, обработать запрос самостоятельно или передать следующему.

### ASCII-диаграмма

```
Запрос ──> [Handler A] ──> [Handler B] ──> [Handler C] ──> Результат
             │                 │                │
             v                 v                v
          Обработать?      Обработать?      Обработать?
          Нет → дальше     Да → обработать  (последний в цепи)
```

### Применение в тестировании

1. **Фильтры тестов** — по тегам, по приоритету, по окружению.
2. **Валидация тестовых данных** — цепочка проверок перед использованием.
3. **Обработка ошибок** — разные обработчики для разных типов исключений.
4. **Middleware для API-тестов** — добавление заголовков, логирование, аутентификация.

### Пример: Цепочка валидации тестовых данных

```java
/**
 * Абстрактный обработчик в цепочке валидации.
 */
public abstract class ValidationHandler {

    private ValidationHandler next;

    /**
     * Устанавливает следующий обработчик в цепочке.
     * Возвращает следующий — для fluent-построения цепочки.
     */
    public ValidationHandler setNext(ValidationHandler handler) {
        this.next = handler;
        return handler; // Возвращаем next для chaining
    }

    /**
     * Обрабатывает запрос — либо сам, либо передаёт дальше.
     */
    public ValidationResult handle(TestData data) {
        ValidationResult result = validate(data);
        if (!result.isValid()) {
            return result; // Валидация не пройдена — останавливаемся
        }
        if (next != null) {
            return next.handle(data); // Передаём следующему
        }
        return ValidationResult.ok(); // Все проверки пройдены
    }

    protected abstract ValidationResult validate(TestData data);
}

/**
 * Проверка обязательных полей.
 */
public class RequiredFieldsValidator extends ValidationHandler {

    @Override
    protected ValidationResult validate(TestData data) {
        if (data.getName() == null || data.getName().isBlank()) {
            return ValidationResult.fail("Поле 'name' обязательно");
        }
        if (data.getEmail() == null || data.getEmail().isBlank()) {
            return ValidationResult.fail("Поле 'email' обязательно");
        }
        return ValidationResult.ok();
    }
}

/**
 * Проверка формата email.
 */
public class EmailFormatValidator extends ValidationHandler {

    private static final Pattern EMAIL_PATTERN =
        Pattern.compile("^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+$");

    @Override
    protected ValidationResult validate(TestData data) {
        if (!EMAIL_PATTERN.matcher(data.getEmail()).matches()) {
            return ValidationResult.fail(
                "Невалидный формат email: " + data.getEmail());
        }
        return ValidationResult.ok();
    }
}

/**
 * Проверка уникальности данных в БД.
 */
public class UniquenessValidator extends ValidationHandler {

    private final DatabaseClient dbClient;

    public UniquenessValidator(DatabaseClient dbClient) {
        this.dbClient = dbClient;
    }

    @Override
    protected ValidationResult validate(TestData data) {
        if (dbClient.existsByEmail(data.getEmail())) {
            return ValidationResult.fail(
                "Пользователь с email " + data.getEmail() + " уже существует");
        }
        return ValidationResult.ok();
    }
}

/**
 * Результат валидации.
 */
public record ValidationResult(boolean isValid, String message) {
    public static ValidationResult ok() {
        return new ValidationResult(true, "OK");
    }
    public static ValidationResult fail(String message) {
        return new ValidationResult(false, message);
    }
}
```

### Построение и использование цепочки

```java
class UserRegistrationTest {

    private ValidationHandler validationChain;

    @BeforeEach
    void setUp() {
        // Строим цепочку: обязательные поля → формат email → уникальность
        var requiredFields = new RequiredFieldsValidator();
        var emailFormat = new EmailFormatValidator();
        var uniqueness = new UniquenessValidator(dbClient);

        requiredFields.setNext(emailFormat).setNext(uniqueness);
        validationChain = requiredFields;
    }

    @Test
    void shouldRejectEmptyName() {
        var data = new TestData(null, "test@test.com");
        var result = validationChain.handle(data);

        assertFalse(result.isValid());
        assertEquals("Поле 'name' обязательно", result.message());
    }

    @Test
    void shouldRejectInvalidEmail() {
        var data = new TestData("John", "not-an-email");
        var result = validationChain.handle(data);

        assertFalse(result.isValid());
        assertTrue(result.message().contains("Невалидный формат email"));
    }

    @Test
    void shouldPassAllValidations() {
        var data = new TestData("John", "john@test.com");
        var result = validationChain.handle(data);

        assertTrue(result.isValid());
    }
}
```

---

## Связь с тестированием

| Паттерн                       | Проблема в тестировании                     | Решение                                     |
|-------------------------------|---------------------------------------------|----------------------------------------------|
| **Strategy**                  | Один и тот же тест должен работать в разных окружениях | Переключение стратегий без изменения тестов  |
| **Template Method**           | Дублирование setup/teardown между тестами   | Общий скелет в базовом классе                |
| **Observer**                  | Нужны скриншоты, логи, отчёты без изменения тестов | Слушатели реагируют на события               |
| **Chain of Responsibility**   | Сложная многоступенчатая валидация          | Цепочка независимых валидаторов              |

Поведенческие паттерны особенно важны для **масштабирования** тестовой инфраструктуры.
Они позволяют добавлять новое поведение (новый listener, новая стратегия, новый валидатор)
без изменения существующего кода, что соответствует принципу Open/Closed из SOLID.

---

## Типичные ошибки

1. **God Strategy** — стратегия, которая делает слишком много. Каждая стратегия должна
   отвечать только за свою область (создание драйвера, но не за тестовые данные).
2. **Глубокая иерархия Template Method** — более 2-3 уровней наследования делают код
   трудным для понимания. Предпочитайте композицию.
3. **Listener с побочными эффектами** — слушатель, который изменяет состояние теста,
   делает поведение непредсказуемым. Listener должен только наблюдать.
4. **Слишком длинная цепочка** — Chain of Responsibility из 10 обработчиков трудно отлаживать.
   Группируйте связанные проверки.
5. **Хардкод стратегии** — `new LocalStrategy()` вместо выбора через конфигурацию лишает
   паттерн смысла.
6. **Забытый `next` в цепочке** — обработчик не передаёт запрос дальше, и часть цепочки
   не выполняется.

---

## Вопросы на интервью

### Уровень Junior

- Что такое паттерн Strategy? Приведите пример из тестирования.
- Что такое Template Method? Как он используется в базовом классе теста?
- Что такое TestNG/JUnit Listener? Для чего он нужен?
- Чем Strategy отличается от простого `if-else`?

### Уровень Middle

- Как реализовать переключение между локальным и удалённым запуском тестов с помощью Strategy?
- Как правильно организовать иерархию наследования базовых классов тестов (Template Method)?
- Как написать JUnit 5 Extension, который делает скриншот при падении теста?
- Как реализовать Chain of Responsibility для валидации тестовых данных?
- Чем Observer отличается от Mediator?

### Уровень Senior

- Как вы решаете проблему «взрыва стратегий», когда комбинаций параметров становится слишком много?
- Когда Template Method — антипаттерн? Как заменить наследование композицией?
- Как организовать систему плагинов для тестового фреймворка с помощью Observer?
- Как Chain of Responsibility реализуется в Servlet Filter и Spring Interceptor?
  Как применить этот подход в тестах?
- Как комбинировать Strategy и Template Method в одном фреймворке?

---

## Практические задания

1. **Strategy: Multi-environment.** Реализуйте три стратегии окружения (dev, staging, prod)
   с разными URL, credentials и таймаутами. Стратегия должна выбираться через системное свойство.

2. **Template Method: BaseTest.** Создайте базовый класс `BaseTest` с шаблонным методом,
   включающим: настройку логирования, создание драйвера, открытие начальной страницы,
   очистку cookies после теста, закрытие драйвера. Напишите два подкласса с реальными тестами.

3. **Observer: Custom Listener.** Напишите JUnit 5 Extension, который:
   - Логирует начало и конец каждого теста.
   - При падении сохраняет скриншот и HTML-страницы.
   - Собирает статистику (количество passed/failed/skipped) и выводит итог после всех тестов.

4. **Chain of Responsibility: Request Pipeline.** Создайте цепочку обработчиков для
   HTTP-запросов в API-тестах: `AuthHandler` (добавляет токен) -> `LoggingHandler`
   (логирует запрос) -> `RetryHandler` (повторяет при 5xx ошибках).

5. **Комбинация.** Спроектируйте фреймворк, в котором:
   - Strategy определяет окружение.
   - Template Method задаёт жизненный цикл теста.
   - Observer собирает отчётность.
   - Chain of Responsibility валидирует предусловия теста.

---

## Дополнительные ресурсы

- **Refactoring Guru — Behavioral Patterns** — интерактивные примеры с UML-диаграммами.
- **JUnit 5 User Guide — Extensions** — официальная документация по механизму расширений.
- **TestNG Documentation — Listeners** — описание всех типов слушателей.
- **"Head First Design Patterns"** — главы Strategy, Observer, Template Method с наглядными примерами.
- **Spring Framework — HandlerInterceptor** — реальная реализация Chain of Responsibility.
- **Allure Framework Source Code** — примеры использования Observer в отчётности.
