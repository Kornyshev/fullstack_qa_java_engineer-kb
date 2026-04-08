# Паттерны в автоматизации тестирования

## Обзор

Паттерны проектирования (Design Patterns) — это проверенные решения типичных задач разработки. В тестовой
автоматизации они решают конкретные проблемы: создание WebDriver для разных браузеров, построение сложных
тестовых данных, управление конфигурацией, расширение поведения тестов. Знание паттернов отличает
автоматизатора-инженера от автоматизатора-скриптописателя. В этом разделе — семь ключевых паттернов
с конкретными примерами из тестовой автоматизации на Java.

---

## Factory — Создание объектов

### Проблема

Создание WebDriver для разных браузеров (Chrome, Firefox, Edge, Remote) требует разной конфигурации.
Если логика разбросана по тестам, любое изменение превращается в кошмар.

### Решение

**Factory** инкапсулирует логику создания объектов. Клиентский код не знает деталей — он просто запрашивает объект.

```java
package com.company.project.driver;

import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;
import org.openqa.selenium.firefox.FirefoxDriver;
import org.openqa.selenium.firefox.FirefoxOptions;
import org.openqa.selenium.remote.RemoteWebDriver;

import java.net.MalformedURLException;
import java.net.URL;

// Фабрика создания WebDriver для разных браузеров
public class WebDriverFactory {

    // Создание драйвера по имени браузера
    public static WebDriver createDriver(String browser) {
        return switch (browser.toLowerCase()) {
            case "chrome" -> createChromeDriver();
            case "firefox" -> createFirefoxDriver();
            case "remote_chrome" -> createRemoteChromeDriver();
            default -> throw new IllegalArgumentException(
                "Неизвестный браузер: " + browser
            );
        };
    }

    private static WebDriver createChromeDriver() {
        ChromeOptions options = new ChromeOptions();
        options.addArguments("--no-sandbox");
        options.addArguments("--disable-dev-shm-usage");
        // Headless-режим — для CI/CD, где нет дисплея
        if (Boolean.getBoolean("headless")) {
            options.addArguments("--headless=new");
        }
        return new ChromeDriver(options);
    }

    private static WebDriver createFirefoxDriver() {
        FirefoxOptions options = new FirefoxOptions();
        if (Boolean.getBoolean("headless")) {
            options.addArguments("--headless");
        }
        return new FirefoxDriver(options);
    }

    private static WebDriver createRemoteChromeDriver() {
        ChromeOptions options = new ChromeOptions();
        options.setCapability("selenoid:options", Map.of(
            "enableVNC", true,
            "enableVideo", true
        ));
        try {
            return new RemoteWebDriver(
                new URL("http://selenoid:4444/wd/hub"), options
            );
        } catch (MalformedURLException e) {
            throw new RuntimeException("Некорректный URL Selenoid", e);
        }
    }
}
```

Использование:

```java
WebDriver driver = WebDriverFactory.createDriver(config.browser());
```

### Когда применять

- Создание драйверов для разных браузеров или окружений.
- Создание HTTP-клиентов с разными настройками (авторизация, заголовки).
- Создание подключений к разным базам данных.

---

## Builder — Построение сложных объектов

### Проблема

Тестовые данные (пользователи, заказы, товары) имеют множество полей. Конструкторы с 10+ параметрами
нечитаемы. Нужен способ создавать объекты по частям с понятным синтаксисом.

### Решение

**Builder** позволяет пошагово конструировать сложный объект. В Java часто реализуется через Lombok `@Builder`.

```java
package com.company.project.models;

import lombok.Builder;
import lombok.Data;

// Модель пользователя с Builder-паттерном через Lombok
@Data
@Builder
public class User {
    private String username;
    private String email;
    private String password;
    private String firstName;
    private String lastName;
    private String phone;
    private String role;
    private boolean active;
}
```

Использование в тестах:

```java
// Создание пользователя с нужными полями — остальные null
User admin = User.builder()
        .username("admin")
        .email("admin@test.com")
        .password("secure123")
        .role("ADMIN")
        .active(true)
        .build();

// Минимальный пользователь для негативного теста
User emptyUser = User.builder()
        .username("")
        .password("")
        .build();
```

### Ручная реализация Builder (без Lombok)

```java
public class RequestSpecBuilder {

    private String baseUri;
    private String basePath;
    private String authToken;
    private Map<String, String> headers = new HashMap<>();
    private ContentType contentType = ContentType.JSON;

    // Каждый метод возвращает this — для цепочки вызовов
    public RequestSpecBuilder baseUri(String baseUri) {
        this.baseUri = baseUri;
        return this;
    }

    public RequestSpecBuilder basePath(String basePath) {
        this.basePath = basePath;
        return this;
    }

    public RequestSpecBuilder authToken(String token) {
        this.authToken = token;
        return this;
    }

    public RequestSpecBuilder header(String name, String value) {
        this.headers.put(name, value);
        return this;
    }

    // Финализирующий метод — собирает объект
    public RequestSpecification build() {
        var spec = RestAssured.given()
                .baseUri(baseUri)
                .basePath(basePath)
                .contentType(contentType);

        if (authToken != null) {
            spec.header("Authorization", "Bearer " + authToken);
        }
        headers.forEach(spec::header);
        return spec;
    }
}
```

```java
// Использование: читается как предложение
RequestSpecification spec = new RequestSpecBuilder()
        .baseUri("https://api.myshop.com")
        .basePath("/v1/users")
        .authToken(token)
        .header("X-Request-Id", UUID.randomUUID().toString())
        .build();
```

---

## Strategy — Разные стратегии выполнения

### Проблема

Необходимо поддерживать разные стратегии: авторизация через UI или API, генерация данных через фабрику
или из файла, запуск браузера локально или удалённо. Условные конструкции `if/else` загрязняют код.

### Решение

**Strategy** определяет семейство алгоритмов и позволяет переключать их без изменения клиентского кода.

```java
// Интерфейс стратегии авторизации
public interface AuthStrategy {
    String authenticate(String username, String password);
}

// Стратегия 1: авторизация через API (быстро)
public class ApiAuthStrategy implements AuthStrategy {

    @Override
    public String authenticate(String username, String password) {
        // Получаем токен через REST API — быстро и надёжно
        return given()
                .contentType(ContentType.JSON)
                .body(Map.of("username", username, "password", password))
                .post("/auth/login")
                .then()
                .extract()
                .path("token");
    }
}

// Стратегия 2: авторизация через UI (медленно, но проверяет UI)
public class UiAuthStrategy implements AuthStrategy {

    @Override
    public String authenticate(String username, String password) {
        // Проходим авторизацию через браузер
        LoginPage loginPage = new LoginPage();
        loginPage.enterUsername(username)
                 .enterPassword(password)
                 .clickLogin();
        // Извлекаем токен из cookies/localStorage
        return WebDriverRunner.getWebDriver()
                .manage().getCookieNamed("auth_token").getValue();
    }
}
```

Контекст, использующий стратегию:

```java
// Контекст авторизации — переключаемая стратегия
public class AuthManager {

    private final AuthStrategy strategy;

    public AuthManager(AuthStrategy strategy) {
        this.strategy = strategy;
    }

    // Фабричный метод для создания нужной стратегии
    public static AuthManager create(String type) {
        AuthStrategy strategy = switch (type) {
            case "api" -> new ApiAuthStrategy();
            case "ui" -> new UiAuthStrategy();
            default -> throw new IllegalArgumentException(
                "Неизвестная стратегия: " + type);
        };
        return new AuthManager(strategy);
    }

    public String login(String username, String password) {
        return strategy.authenticate(username, password);
    }
}
```

```java
// В тесте — стратегия определяется конфигурацией
AuthManager auth = AuthManager.create(config.authStrategy());
String token = auth.login("user", "pass");
```

---

## Decorator — Расширение поведения

### Проблема

Нужно добавить логирование, скриншоты, замер времени к существующим действиям без изменения исходного кода.

### Решение

**Decorator** оборачивает объект, добавляя новую функциональность. В автоматизации часто используется
для логирования шагов и создания скриншотов.

```java
// Базовый интерфейс действия
public interface TestAction {
    void execute();
    String getDescription();
}

// Конкретное действие — клик по элементу
public class ClickAction implements TestAction {

    private final SelenideElement element;
    private final String description;

    public ClickAction(SelenideElement element, String description) {
        this.element = element;
        this.description = description;
    }

    @Override
    public void execute() {
        element.click();
    }

    @Override
    public String getDescription() {
        return description;
    }
}

// Декоратор: добавляет логирование
public class LoggingDecorator implements TestAction {

    private static final Logger log = LoggerFactory.getLogger(LoggingDecorator.class);
    private final TestAction wrappedAction;

    public LoggingDecorator(TestAction action) {
        this.wrappedAction = action;
    }

    @Override
    public void execute() {
        log.info("Выполняется: {}", wrappedAction.getDescription());
        long start = System.currentTimeMillis();
        wrappedAction.execute();
        long duration = System.currentTimeMillis() - start;
        log.info("Завершено за {} мс: {}", duration, wrappedAction.getDescription());
    }

    @Override
    public String getDescription() {
        return wrappedAction.getDescription();
    }
}

// Декоратор: добавляет скриншот при ошибке
public class ScreenshotDecorator implements TestAction {

    private final TestAction wrappedAction;

    public ScreenshotDecorator(TestAction action) {
        this.wrappedAction = action;
    }

    @Override
    public void execute() {
        try {
            wrappedAction.execute();
        } catch (Exception e) {
            // При ошибке — делаем скриншот и прикрепляем к Allure
            Allure.addAttachment(
                "Скриншот ошибки",
                "image/png",
                new ByteArrayInputStream(
                    Screenshots.takeScreenShotAsBytes()
                ),
                ".png"
            );
            throw e;
        }
    }

    @Override
    public String getDescription() {
        return wrappedAction.getDescription();
    }
}
```

Использование — декораторы можно комбинировать:

```java
// Оборачиваем действие в несколько декораторов
TestAction action = new ClickAction(
    $(".submit-btn"), "Нажатие кнопки 'Отправить'"
);
TestAction decorated = new ScreenshotDecorator(
    new LoggingDecorator(action)
);
decorated.execute();
```

> **На практике** в Selenide/Allure декорирование уже встроено через `SelenideLogger` и `AllureSelenide`.
> Но понимание паттерна помогает на интервью и при создании custom-расширений.

---

## Chain of Responsibility — Цепочка обработчиков

### Проблема

При подготовке тестового окружения нужно выполнить цепочку действий: очистить БД, создать пользователя,
настроить права, загрузить тестовые данные. Каждый шаг может быть опциональным и зависит от контекста.

### Решение

**Chain of Responsibility** передаёт запрос по цепочке обработчиков. Каждый обработчик решает,
обработать запрос или передать дальше.

```java
// Абстрактный обработчик подготовки тестового окружения
public abstract class SetupHandler {

    private SetupHandler next;

    // Установка следующего обработчика в цепочке
    public SetupHandler setNext(SetupHandler handler) {
        this.next = handler;
        return handler; // Для цепочки вызовов
    }

    public void handle(TestContext context) {
        doHandle(context);
        if (next != null) {
            next.handle(context);
        }
    }

    // Конкретная логика обработки — реализуется в наследниках
    protected abstract void doHandle(TestContext context);
}

// Обработчик 1: очистка базы данных
public class DatabaseCleanupHandler extends SetupHandler {

    @Override
    protected void doHandle(TestContext context) {
        if (context.requiresCleanDb()) {
            // Очищаем таблицы перед тестом
            DbUtils.truncateTables("orders", "cart_items", "users");
            log.info("БД очищена");
        }
    }
}

// Обработчик 2: создание тестового пользователя
public class UserCreationHandler extends SetupHandler {

    @Override
    protected void doHandle(TestContext context) {
        if (context.requiresUser()) {
            User user = UserFactory.randomUser();
            ApiClient.createUser(user);
            context.setCreatedUser(user);
            log.info("Пользователь создан: {}", user.getUsername());
        }
    }
}

// Обработчик 3: настройка тестовых данных
public class TestDataHandler extends SetupHandler {

    @Override
    protected void doHandle(TestContext context) {
        if (context.requiresProducts()) {
            List<Product> products = ProductFactory.createBatch(5);
            products.forEach(ApiClient::createProduct);
            context.setProducts(products);
            log.info("Создано {} товаров", products.size());
        }
    }
}
```

Построение цепочки:

```java
// Собираем цепочку обработчиков
SetupHandler chain = new DatabaseCleanupHandler();
chain.setNext(new UserCreationHandler())
     .setNext(new TestDataHandler());

// Запускаем: каждый обработчик выполняет свою часть
TestContext context = new TestContext()
        .requireCleanDb(true)
        .requireUser(true)
        .requireProducts(true);

chain.handle(context);
```

> **На практике** Chain of Responsibility реализуется через TestNG/JUnit listeners, Allure lifecycle listeners,
> Spring `@TestExecutionListeners`.

---

## Singleton — Единственный экземпляр

### Проблема

Конфигурация, подключение к БД, HTTP-клиент — должны существовать в единственном экземпляре.
Множественные подключения расходуют ресурсы и создают проблемы с thread safety.

### Решение

**Singleton** гарантирует единственный экземпляр класса. В Java 17+ предпочтительна реализация через `enum`
или thread-safe lazy initialization.

```java
// Singleton через enum — самый простой и безопасный вариант
public enum DatabaseConnection {
    INSTANCE;

    private final DataSource dataSource;

    DatabaseConnection() {
        // Инициализация пула соединений — один раз
        HikariConfig hikariConfig = new HikariConfig();
        hikariConfig.setJdbcUrl(ConfigProvider.getConfig().dbUrl());
        hikariConfig.setUsername(ConfigProvider.getConfig().dbUsername());
        hikariConfig.setPassword(ConfigProvider.getConfig().dbPassword());
        hikariConfig.setMaximumPoolSize(10);
        this.dataSource = new HikariDataSource(hikariConfig);
    }

    public Connection getConnection() throws SQLException {
        return dataSource.getConnection();
    }
}
```

```java
// Использование — один пул соединений для всех тестов
try (Connection conn = DatabaseConnection.INSTANCE.getConnection()) {
    // Выполнение SQL-запросов
}
```

### Singleton через double-checked locking (для классов, где enum неудобен)

```java
public class ConfigProvider {

    private static volatile ProjectConfig config;

    public static ProjectConfig getConfig() {
        if (config == null) {
            synchronized (ConfigProvider.class) {
                if (config == null) {
                    config = ConfigFactory.create(ProjectConfig.class);
                }
            }
        }
        return config;
    }

    private ConfigProvider() { }
}
```

### Когда Singleton — антипаттерн

- **В тестах** — Singleton с mutable state делает тесты зависимыми друг от друга.
- **Параллельный запуск** — Singleton WebDriver невозможно использовать параллельно (используйте ThreadLocal).
- **Тестирование самого Singleton** — трудно подменить (mock) в юнит-тестах.

---

## Adapter — Адаптер разных интерфейсов

### Проблема

Приложение имеет десктопную версию (Swing/JavaFX) и веб-версию. Тесты должны работать с обоими UI,
но API взаимодействия совершенно разные.

### Решение

**Adapter** приводит интерфейс одного класса к интерфейсу, который ожидает клиент.

```java
// Единый интерфейс для работы с формой логина (независимо от реализации UI)
public interface LoginForm {
    void enterUsername(String username);
    void enterPassword(String password);
    void submit();
    String getErrorMessage();
}

// Адаптер для веб-интерфейса (Selenide)
public class WebLoginFormAdapter implements LoginForm {

    @Override
    public void enterUsername(String username) {
        $("#username").setValue(username);
    }

    @Override
    public void enterPassword(String password) {
        $("#password").setValue(password);
    }

    @Override
    public void submit() {
        $("[data-testid='login-btn']").click();
    }

    @Override
    public String getErrorMessage() {
        return $(".error-message").getText();
    }
}

// Адаптер для мобильного интерфейса (Appium)
public class MobileLoginFormAdapter implements LoginForm {

    private final AppiumDriver driver;

    public MobileLoginFormAdapter(AppiumDriver driver) {
        this.driver = driver;
    }

    @Override
    public void enterUsername(String username) {
        driver.findElement(AppiumBy.accessibilityId("usernameField"))
              .sendKeys(username);
    }

    @Override
    public void enterPassword(String password) {
        driver.findElement(AppiumBy.accessibilityId("passwordField"))
              .sendKeys(password);
    }

    @Override
    public void submit() {
        driver.findElement(AppiumBy.accessibilityId("loginButton"))
              .click();
    }

    @Override
    public String getErrorMessage() {
        return driver.findElement(AppiumBy.accessibilityId("errorLabel"))
                     .getText();
    }
}
```

Тесты работают с абстракцией, не зная деталей:

```java
// Тест не зависит от типа UI
public class LoginTest {

    private final LoginForm loginForm;

    public LoginTest(LoginForm loginForm) {
        this.loginForm = loginForm;
    }

    @Test
    void invalidPasswordShowsError() {
        loginForm.enterUsername("user");
        loginForm.enterPassword("wrong");
        loginForm.submit();
        assertThat(loginForm.getErrorMessage())
                .contains("Неверный пароль");
    }
}
```

---

## Сводная таблица паттернов

| Паттерн | Задача в автоматизации | Пример |
|---------|----------------------|--------|
| **Factory** | Создание объектов с разными конфигурациями | WebDriver для Chrome/Firefox/Remote |
| **Builder** | Конструирование сложных тестовых данных | User, Order, Request |
| **Strategy** | Переключение алгоритмов | Auth через UI vs API |
| **Decorator** | Добавление поведения | Логирование, скриншоты |
| **Chain of Responsibility** | Последовательная обработка | Setup: clean DB → create user → load data |
| **Singleton** | Единственный экземпляр | Config, DB connection pool |
| **Adapter** | Унификация разных интерфейсов | Web vs Mobile UI |

---

## Связь с тестированием

Паттерны проектирования в тестовой автоматизации:

- **Снижают хрупкость** — изменения изолированы в одном месте (Factory, Adapter).
- **Улучшают читаемость** — Builder делает создание данных понятным.
- **Обеспечивают расширяемость** — Strategy позволяет добавить новый алгоритм без изменения существующего кода.
- **Упрощают отладку** — Decorator с логированием показывает каждый шаг выполнения.
- **Масштабируют фреймворк** — Chain of Responsibility позволяет наращивать pipeline подготовки данных.

---

## Типичные ошибки

1. **Паттерн ради паттерна** — применение Singleton там, где достаточно обычного поля.
2. **Перекомплексование** — цепочка из 5 декораторов, когда хватило бы простого `try/catch`.
3. **Нарушение Single Responsibility** — Factory, которая не только создаёт, но и настраивает и валидирует.
4. **Mutable Singleton** — Singleton с изменяемым состоянием ломается при параллельных тестах.
5. **God Object вместо Strategy** — один класс с `if/else` на 500 строк вместо набора стратегий.
6. **Незнание встроенных реализаций** — ручная реализация Decorator, когда Selenide/Allure уже предоставляют listener-механизм.

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Какие паттерны проектирования вы использовали в тестовой автоматизации?
2. Что такое Factory Pattern? Приведите пример из автоматизации.
3. Зачем нужен Builder Pattern? Как его реализовать через Lombok?
4. Что такое Singleton? Когда он полезен, а когда вреден?
5. Что такое Page Object Pattern? (Это тоже паттерн!)

### 🟡 Средний уровень
6. Как бы вы реализовали создание WebDriver для разных браузеров?
7. В чём разница между Strategy и Factory?
8. Как Decorator Pattern используется в Allure?
9. Приведите пример Chain of Responsibility в тестовом фреймворке.
10. Как Adapter Pattern помогает при тестировании web и mobile?

### 🔴 Продвинутый уровень
11. Как обеспечить thread safety для Singleton в параллельных тестах?
12. Как совместить Factory и Strategy для создания гибкой системы инициализации?
13. Какие паттерны GoF вы считаете вредными в тестовой автоматизации? Почему?
14. Как реализовать Plugin-архитектуру для тестового фреймворка?
15. Расскажите о паттерне Screenplay и чем он отличается от Page Object.

---

## Практические задания

### Задание 1: WebDriverFactory
Реализуйте Factory для создания WebDriver с поддержкой Chrome, Firefox и Remote (Selenoid).
Конфигурация должна читаться из Owner-интерфейса.

### Задание 2: Builder для тестовых данных
Создайте модели `User`, `Product`, `Order` с `@Builder` (Lombok). Реализуйте фабрику с предустановленными
Builder-ами: `UserFactory.admin()`, `UserFactory.random()`, `ProductFactory.inStock()`.

### Задание 3: Strategy для авторизации
Реализуйте `AuthStrategy` с двумя реализациями (API и UI). Стратегия выбирается через конфигурацию.
Напишите тесты, которые используют обе стратегии.

### Задание 4: Комбинирование паттернов
Создайте мини-фреймворк, в котором:
- `WebDriverFactory` (Factory) создаёт драйвер
- `ConfigProvider` (Singleton) хранит конфигурацию
- `User.builder()` (Builder) создаёт тестовые данные
- `AuthManager` (Strategy) управляет авторизацией

---

## Дополнительные ресурсы

- [Refactoring Guru — Design Patterns](https://refactoring.guru/design-patterns)
- [Head First Design Patterns](https://www.oreilly.com/library/view/head-first-design/9781492077992/) — O'Reilly
- [Test Automation Patterns Wiki](https://testautomationpatterns.org/)
- [Selenium Page Object Pattern](https://www.selenium.dev/documentation/test_practices/encouraged/page_object_models/)
- [Baeldung — Design Patterns in Java](https://www.baeldung.com/design-patterns-series)
