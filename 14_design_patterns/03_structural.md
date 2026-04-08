# Структурные паттерны (Structural Patterns)

## Обзор

Структурные паттерны описывают, как **компоновать объекты и классы** в более крупные структуры,
сохраняя при этом гибкость и эффективность. Они помогают выстроить отношения между сущностями
так, чтобы изменение одной части системы минимально затрагивало остальные.

Для QA Automation Engineer наиболее значимы четыре структурных паттерна:

- **Adapter** — приведение несовместимых интерфейсов к единому виду.
- **Decorator** — добавление поведения к объекту без изменения его класса.
- **Proxy** — контроль доступа к объекту через объект-заместитель.
- **Facade** — предоставление простого интерфейса к сложной подсистеме.

---

## Adapter (Адаптер)

### Назначение

Adapter преобразует интерфейс одного класса в интерфейс, ожидаемый клиентом. Позволяет
объектам с несовместимыми интерфейсами **работать вместе**.

### ASCII-диаграмма

```
┌──────────────┐      ┌──────────────────┐      ┌──────────────────┐
│   Тестовый   │      │     Adapter      │      │  Внешняя библ.   │
│    код       │─────>│  «implements      │─────>│  (несовместимый  │
│              │      │   наш интерфейс»  │      │   интерфейс)     │
└──────────────┘      └──────────────────┘      └──────────────────┘
      Client         Приводит интерфейс            Adaptee
                     Adaptee к Target
```

### Применение в тестировании

1. **Адаптация разных UI-фреймворков** — единый интерфейс для Selenium, Playwright, Selenide.
2. **Адаптация HTTP-клиентов** — единый интерфейс для RestAssured, OkHttp, HttpClient.
3. **Миграция между фреймворками** — плавный переход без переписывания всех тестов.

### Пример: Adapter для HTTP-клиентов

```java
/**
 * Целевой интерфейс, который ожидают наши тесты.
 * Все тесты работают через этот интерфейс.
 */
public interface HttpClient {
    Response get(String url, Map<String, String> headers);
    Response post(String url, String body, Map<String, String> headers);
    Response put(String url, String body, Map<String, String> headers);
    Response delete(String url, Map<String, String> headers);
}

/**
 * Наша унифицированная модель ответа.
 */
public record Response(int statusCode, String body, Map<String, String> headers) {}

/**
 * Адаптер для RestAssured.
 * Оборачивает вызовы RestAssured в наш единый интерфейс HttpClient.
 */
public class RestAssuredAdapter implements HttpClient {

    @Override
    public Response get(String url, Map<String, String> headers) {
        // Адаптируем вызов RestAssured к нашему интерфейсу
        var restAssuredResponse = given()
            .headers(headers)
            .when()
            .get(url)
            .then()
            .extract().response();

        return convertResponse(restAssuredResponse);
    }

    @Override
    public Response post(String url, String body, Map<String, String> headers) {
        var restAssuredResponse = given()
            .headers(headers)
            .body(body)
            .when()
            .post(url)
            .then()
            .extract().response();

        return convertResponse(restAssuredResponse);
    }

    @Override
    public Response put(String url, String body, Map<String, String> headers) {
        var restAssuredResponse = given()
            .headers(headers)
            .body(body)
            .when()
            .put(url)
            .then()
            .extract().response();

        return convertResponse(restAssuredResponse);
    }

    @Override
    public Response delete(String url, Map<String, String> headers) {
        var restAssuredResponse = given()
            .headers(headers)
            .when()
            .delete(url)
            .then()
            .extract().response();

        return convertResponse(restAssuredResponse);
    }

    /**
     * Конвертация ответа RestAssured в нашу модель Response.
     */
    private Response convertResponse(io.restassured.response.Response raw) {
        Map<String, String> headerMap = new HashMap<>();
        raw.headers().forEach(h -> headerMap.put(h.getName(), h.getValue()));
        return new Response(raw.statusCode(), raw.body().asString(), headerMap);
    }
}

/**
 * Адаптер для стандартного Java HttpClient (java.net.http).
 * Позволяет использовать тот же интерфейс с другой реализацией.
 */
public class JavaHttpClientAdapter implements HttpClient {

    private final java.net.http.HttpClient client =
        java.net.http.HttpClient.newHttpClient();

    @Override
    public Response get(String url, Map<String, String> headers) {
        var requestBuilder = HttpRequest.newBuilder()
            .uri(URI.create(url))
            .GET();
        headers.forEach(requestBuilder::header);

        return executeRequest(requestBuilder.build());
    }

    @Override
    public Response post(String url, String body, Map<String, String> headers) {
        var requestBuilder = HttpRequest.newBuilder()
            .uri(URI.create(url))
            .POST(HttpRequest.BodyPublishers.ofString(body));
        headers.forEach(requestBuilder::header);

        return executeRequest(requestBuilder.build());
    }

    // put, delete — аналогично...

    @Override
    public Response put(String url, String body, Map<String, String> headers) {
        var requestBuilder = HttpRequest.newBuilder()
            .uri(URI.create(url))
            .PUT(HttpRequest.BodyPublishers.ofString(body));
        headers.forEach(requestBuilder::header);
        return executeRequest(requestBuilder.build());
    }

    @Override
    public Response delete(String url, Map<String, String> headers) {
        var requestBuilder = HttpRequest.newBuilder()
            .uri(URI.create(url))
            .DELETE();
        headers.forEach(requestBuilder::header);
        return executeRequest(requestBuilder.build());
    }

    private Response executeRequest(HttpRequest request) {
        try {
            var httpResponse = client.send(request,
                HttpResponse.BodyHandlers.ofString());
            Map<String, String> headerMap = new HashMap<>();
            httpResponse.headers().map().forEach(
                (k, v) -> headerMap.put(k, String.join(", ", v)));
            return new Response(httpResponse.statusCode(),
                httpResponse.body(), headerMap);
        } catch (IOException | InterruptedException e) {
            throw new RuntimeException("Ошибка выполнения HTTP-запроса", e);
        }
    }
}
```

### Использование в тестах

```java
class ApiTest {
    // Тесты не зависят от конкретного HTTP-клиента
    private final HttpClient client = new RestAssuredAdapter();
    // Переключение: private final HttpClient client = new JavaHttpClientAdapter();

    @Test
    void shouldReturnUserById() {
        var response = client.get(
            "http://localhost:8080/api/users/1",
            Map.of("Accept", "application/json")
        );
        assertEquals(200, response.statusCode());
    }
}
```

---

## Decorator (Декоратор)

### Назначение

Decorator динамически добавляет объекту **новые обязанности**, оборачивая его в объект-декоратор.
Альтернатива наследованию для расширения функциональности.

### ASCII-диаграмма

```
┌──────────────────┐
│   Component      │ (интерфейс)
│  + execute()     │
└────────┬─────────┘
         │
    ┌────┴─────────────────┐
    │                      │
    v                      v
┌──────────────┐   ┌──────────────────────┐
│ ConcreteComp │   │   BaseDecorator      │
│ + execute()  │   │ - wrapped: Component │
└──────────────┘   │ + execute()          │
                   └────────┬─────────────┘
                            │
                   ┌────────┴────────┐
                   v                 v
          ┌────────────────┐ ┌────────────────┐
          │ LoggingDecorator│ │ScreenshotDeco │
          │ + execute()    │ │ + execute()    │
          └────────────────┘ └────────────────┘
```

### Применение в тестировании

1. **Логирование** — добавление логов к каждому действию без изменения Page Object.
2. **Скриншоты** — автоматическое снятие скриншотов при ошибках.
3. **Замер времени** — измерение продолжительности операций.
4. **Повторные попытки (retry)** — оборачивание нестабильных действий.

### Пример: Декоратор для WebDriver действий

```java
/**
 * Интерфейс тестового шага.
 */
public interface TestStep {
    void execute();
    String getDescription();
}

/**
 * Конкретный шаг — клик по элементу.
 */
public class ClickStep implements TestStep {

    private final WebDriver driver;
    private final By locator;

    public ClickStep(WebDriver driver, By locator) {
        this.driver = driver;
        this.locator = locator;
    }

    @Override
    public void execute() {
        driver.findElement(locator).click();
    }

    @Override
    public String getDescription() {
        return "Клик по элементу: " + locator;
    }
}

/**
 * Базовый декоратор — делегирует всё оборачиваемому шагу.
 */
public abstract class StepDecorator implements TestStep {

    protected final TestStep wrappedStep;

    protected StepDecorator(TestStep wrappedStep) {
        this.wrappedStep = wrappedStep;
    }

    @Override
    public void execute() {
        wrappedStep.execute();
    }

    @Override
    public String getDescription() {
        return wrappedStep.getDescription();
    }
}

/**
 * Декоратор: добавляет логирование до и после шага.
 */
public class LoggingDecorator extends StepDecorator {

    private static final Logger log = LoggerFactory.getLogger(LoggingDecorator.class);

    public LoggingDecorator(TestStep wrappedStep) {
        super(wrappedStep);
    }

    @Override
    public void execute() {
        log.info("Начало шага: {}", getDescription());
        long start = System.currentTimeMillis();

        wrappedStep.execute();

        long duration = System.currentTimeMillis() - start;
        log.info("Шаг завершён за {} мс: {}", duration, getDescription());
    }
}

/**
 * Декоратор: делает скриншот при ошибке.
 */
public class ScreenshotOnFailureDecorator extends StepDecorator {

    private final WebDriver driver;

    public ScreenshotOnFailureDecorator(TestStep wrappedStep, WebDriver driver) {
        super(wrappedStep);
        this.driver = driver;
    }

    @Override
    public void execute() {
        try {
            wrappedStep.execute();
        } catch (Exception e) {
            // Снимаем скриншот перед пробросом исключения
            takeScreenshot();
            throw e;
        }
    }

    private void takeScreenshot() {
        if (driver instanceof TakesScreenshot ts) {
            File screenshot = ts.getScreenshotAs(OutputType.FILE);
            String fileName = "screenshot_" + System.currentTimeMillis() + ".png";
            try {
                Files.copy(screenshot.toPath(), Path.of("target/screenshots", fileName));
            } catch (IOException e) {
                // Логируем, но не скрываем оригинальную ошибку
                System.err.println("Не удалось сохранить скриншот: " + e.getMessage());
            }
        }
    }
}

/**
 * Декоратор: повторная попытка при нестабильном элементе.
 */
public class RetryDecorator extends StepDecorator {

    private final int maxRetries;
    private final Duration delayBetweenRetries;

    public RetryDecorator(TestStep wrappedStep, int maxRetries, Duration delay) {
        super(wrappedStep);
        this.maxRetries = maxRetries;
        this.delayBetweenRetries = delay;
    }

    @Override
    public void execute() {
        int attempt = 0;
        while (true) {
            try {
                wrappedStep.execute();
                return; // Успех — выходим
            } catch (Exception e) {
                attempt++;
                if (attempt >= maxRetries) {
                    throw e; // Исчерпаны попытки — пробрасываем
                }
                try {
                    Thread.sleep(delayBetweenRetries.toMillis());
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    throw new RuntimeException(ie);
                }
            }
        }
    }
}
```

### Комбинирование декораторов

```java
// Оборачиваем шаг несколькими декораторами — каждый добавляет поведение
TestStep clickLogin = new ClickStep(driver, By.id("loginBtn"));

// Добавляем retry, потом логирование, потом скриншот при ошибке
TestStep enhancedStep = new ScreenshotOnFailureDecorator(
    new LoggingDecorator(
        new RetryDecorator(clickLogin, 3, Duration.ofSeconds(1))
    ),
    driver
);

enhancedStep.execute();
// Порядок: скриншот → логирование → retry → клик
```

---

## Proxy (Заместитель)

### Назначение

Proxy предоставляет суррогатный объект, контролирующий доступ к другому объекту.
Позволяет выполнять действия **до или после** обращения к реальному объекту.

### ASCII-диаграмма

```
┌──────────────┐       ┌──────────────────┐       ┌──────────────────┐
│    Client    │──────>│     Proxy        │──────>│  RealSubject     │
│  (тест)      │       │ - realSubject    │       │  (тяжёлый объект)│
│              │       │ + request()      │       │  + request()     │
└──────────────┘       └──────────────────┘       └──────────────────┘
                        Контролирует доступ,
                        создаёт по требованию
```

### Применение в тестировании

1. **Lazy Initialization** — отложенное создание тяжёлых объектов (WebDriver, соединения с БД).
2. **Кэширование** — кэширование результатов запросов в тестах.
3. **Логирование** — прозрачное логирование всех обращений.
4. **Защитный прокси** — контроль доступа к ресурсам.

### Пример: Lazy Page Object

```java
/**
 * Интерфейс страницы.
 */
public interface Page {
    void open();
    String getTitle();
    boolean isLoaded();
}

/**
 * Реальная реализация — тяжёлый объект с инициализацией элементов.
 */
public class DashboardPage implements Page {

    private final WebDriver driver;
    private final WebElement header;
    private final WebElement sidebar;
    private final List<WebElement> widgets;

    public DashboardPage(WebDriver driver) {
        this.driver = driver;
        // Инициализация — может быть долгой
        PageFactory.initElements(driver, this);
        this.header = driver.findElement(By.id("header"));
        this.sidebar = driver.findElement(By.id("sidebar"));
        this.widgets = driver.findElements(By.className("widget"));
    }

    @Override
    public void open() {
        driver.get(ConfigHolder.getInstance().getBaseUrl() + "/dashboard");
    }

    @Override
    public String getTitle() {
        return header.getText();
    }

    @Override
    public boolean isLoaded() {
        return header.isDisplayed();
    }
}

/**
 * Proxy с ленивой инициализацией.
 * Реальный Page Object создаётся только при первом обращении.
 */
public class LazyPageProxy implements Page {

    private final WebDriver driver;
    private DashboardPage realPage; // Создаётся лениво

    public LazyPageProxy(WebDriver driver) {
        this.driver = driver;
        // Реальная страница НЕ создаётся в конструкторе
    }

    @Override
    public void open() {
        getRealPage().open();
    }

    @Override
    public String getTitle() {
        return getRealPage().getTitle();
    }

    @Override
    public boolean isLoaded() {
        return getRealPage().isLoaded();
    }

    /**
     * Ленивая инициализация — создаём реальный объект только при первом вызове.
     */
    private DashboardPage getRealPage() {
        if (realPage == null) {
            realPage = new DashboardPage(driver);
        }
        return realPage;
    }
}
```

### Пример: Кэширующий Proxy для API

```java
/**
 * Proxy, кэширующий ответы API для ускорения тестов.
 * Полезен, когда один и тот же запрос выполняется многократно.
 */
public class CachingHttpClientProxy implements HttpClient {

    private final HttpClient realClient;
    private final Map<String, Response> cache = new ConcurrentHashMap<>();

    public CachingHttpClientProxy(HttpClient realClient) {
        this.realClient = realClient;
    }

    @Override
    public Response get(String url, Map<String, String> headers) {
        // GET-запросы можно кэшировать
        return cache.computeIfAbsent(url, key ->
            realClient.get(key, headers));
    }

    @Override
    public Response post(String url, String body, Map<String, String> headers) {
        // POST не кэшируем — передаём напрямую
        return realClient.post(url, body, headers);
    }

    @Override
    public Response put(String url, String body, Map<String, String> headers) {
        cache.remove(url); // Инвалидируем кэш при обновлении
        return realClient.put(url, body, headers);
    }

    @Override
    public Response delete(String url, Map<String, String> headers) {
        cache.remove(url); // Инвалидируем кэш при удалении
        return realClient.delete(url, headers);
    }

    public void clearCache() {
        cache.clear();
    }
}
```

---

## Facade (Фасад)

### Назначение

Facade предоставляет **унифицированный упрощённый интерфейс** к группе интерфейсов подсистемы.
Определяет высокоуровневый интерфейс, упрощающий использование подсистемы.

### ASCII-диаграмма

```
┌──────────────┐
│   Тестовый   │
│    код       │
└──────┬───────┘
       │ Простой интерфейс
       v
┌──────────────────────────┐
│        Facade            │
│  + login(user, pwd)      │
│  + createOrder(data)     │
│  + verifyEmail(email)    │
└──────┬───────────────────┘
       │ Координирует подсистемы
       │
  ┌────┼────────┬──────────┐
  v    v        v          v
┌────┐┌──────┐┌──────┐┌──────────┐
│Auth││Orders││Email ││ Database │
│API ││ API  ││Server││ Client   │
└────┘└──────┘└──────┘└──────────┘
```

### Применение в тестировании

1. **Упрощение сложного API фреймворка** — один метод вместо цепочки вызовов.
2. **E2E-сценарии** — фасад оркестрирует несколько подсистем (API, UI, БД).
3. **Подготовка тестовых данных** — фасад скрывает сложность создания связанных сущностей.

### Пример: TestDataFacade

```java
/**
 * Фасад для подготовки тестовых данных.
 * Скрывает сложность взаимодействия с несколькими подсистемами:
 * API, база данных, email-сервер.
 */
public class TestDataFacade {

    private final HttpClient apiClient;
    private final DatabaseClient dbClient;
    private final MailHogClient mailClient;

    public TestDataFacade(HttpClient apiClient, DatabaseClient dbClient,
                          MailHogClient mailClient) {
        this.apiClient = apiClient;
        this.dbClient = dbClient;
        this.mailClient = mailClient;
    }

    /**
     * Создаёт пользователя, активирует его и возвращает готовый аккаунт.
     * Один вызов вместо 5-6 шагов.
     */
    public ActiveUser createActiveUser(String name, String email) {
        // Шаг 1: Регистрация через API
        var registerResponse = apiClient.post(
            "/api/register",
            """
            {"name": "%s", "email": "%s", "password": "TestPass123!"}
            """.formatted(name, email),
            Map.of("Content-Type", "application/json")
        );

        String userId = extractUserId(registerResponse);

        // Шаг 2: Получение письма подтверждения из MailHog
        String confirmationLink = mailClient.getLatestConfirmationLink(email);

        // Шаг 3: Подтверждение email
        apiClient.get(confirmationLink, Map.of());

        // Шаг 4: Проверка статуса в БД
        boolean isActive = dbClient.isUserActive(userId);
        if (!isActive) {
            throw new IllegalStateException(
                "Пользователь не активирован после подтверждения email");
        }

        return new ActiveUser(userId, name, email, "TestPass123!");
    }

    /**
     * Создаёт пользователя с оформленным заказом — готовый сценарий для тестирования.
     */
    public OrderContext createUserWithOrder(String productId, int quantity) {
        // Создаём пользователя
        var user = createActiveUser(
            "Test User " + System.currentTimeMillis(),
            "test_" + System.currentTimeMillis() + "@test.com"
        );

        // Авторизуемся
        String token = authenticate(user.email(), user.password());

        // Создаём заказ
        var orderResponse = apiClient.post(
            "/api/orders",
            """
            {"productId": "%s", "quantity": %d}
            """.formatted(productId, quantity),
            Map.of("Authorization", "Bearer " + token,
                   "Content-Type", "application/json")
        );

        String orderId = extractOrderId(orderResponse);
        return new OrderContext(user, orderId, token);
    }

    /**
     * Полная очистка тестовых данных после теста.
     */
    public void cleanup(String userId) {
        dbClient.deleteUserOrders(userId);
        dbClient.deleteUser(userId);
        mailClient.deleteMessagesFor(userId);
    }

    private String authenticate(String email, String password) {
        var response = apiClient.post("/api/login",
            """
            {"email": "%s", "password": "%s"}
            """.formatted(email, password),
            Map.of("Content-Type", "application/json"));
        return extractToken(response);
    }

    private String extractUserId(Response response) { /* ... */ return ""; }
    private String extractOrderId(Response response) { /* ... */ return ""; }
    private String extractToken(Response response) { /* ... */ return ""; }
}

record ActiveUser(String id, String name, String email, String password) {}
record OrderContext(ActiveUser user, String orderId, String token) {}
```

### Использование Facade в тестах

```java
class OrderFlowTest {

    private TestDataFacade testData;

    @BeforeEach
    void setUp() {
        testData = new TestDataFacade(apiClient, dbClient, mailClient);
    }

    @Test
    void shouldCancelOrder() {
        // Весь setup — один вызов фасада
        var context = testData.createUserWithOrder("PROD-001", 2);

        // Тест фокусируется только на бизнес-логике
        var response = apiClient.post(
            "/api/orders/" + context.orderId() + "/cancel",
            "",
            Map.of("Authorization", "Bearer " + context.token())
        );

        assertEquals(200, response.statusCode());
    }

    @AfterEach
    void tearDown() {
        // Очистка — тоже через фасад
        testData.cleanup(context.user().id());
    }
}
```

---

## Связь с тестированием

| Паттерн       | Проблема в тестировании                          | Решение                                           |
|---------------|--------------------------------------------------|---------------------------------------------------|
| **Adapter**   | Зависимость тестов от конкретной библиотеки      | Единый интерфейс, реализация заменяема            |
| **Decorator** | Нужно логирование/скриншоты без изменения кода   | Оборачивание объекта дополнительным поведением     |
| **Proxy**     | Тяжёлая инициализация, лишние запросы            | Ленивое создание и кэширование                    |
| **Facade**    | Сложная многошаговая подготовка данных           | Один метод скрывает всю сложность                 |

Структурные паттерны помогают QA-инженеру строить **модульный и гибкий** тестовый фреймворк.
При изменении внешних зависимостей (обновление Selenium, смена HTTP-клиента) затрагивается
только адаптер, а тесты остаются без изменений.

---

## Типичные ошибки

1. **Adapter с утечкой абстракции** — адаптер раскрывает детали реализации обёрнутого объекта,
   что делает замену невозможной.
2. **Слишком много декораторов** — цепочка из 5-6 декораторов становится трудной для отладки;
   при ошибке непонятно, на каком уровне она произошла.
3. **Proxy без необходимости** — создание прокси для лёгких объектов добавляет сложность
   без выигрыша в производительности.
4. **God Facade** — фасад, который делает слишком много, нарушая Single Responsibility.
   Лучше иметь несколько маленьких фасадов (UserFacade, OrderFacade).
5. **Нарушение контракта в Adapter** — адаптер должен полностью реализовывать целевой
   интерфейс; методы, бросающие `UnsupportedOperationException`, — антипаттерн.
6. **Смешение Decorator и Proxy** — декоратор добавляет поведение, прокси контролирует доступ.
   Путаница ведёт к неясному коду.

---

## Вопросы на интервью

### Уровень Junior

- Что такое паттерн Adapter? Приведите пример из жизни (не из программирования).
- Чем Decorator отличается от наследования? Когда предпочесть декоратор?
- Что такое Facade? Какую проблему он решает?
- Как Proxy используется для ленивой инициализации?

### Уровень Middle

- Как бы вы реализовали переход с RestAssured на другой HTTP-клиент без переписывания тестов?
- Как добавить логирование ко всем Page Object-ам, не модифицируя их код?
- Чем Decorator отличается от Proxy? Когда использовать какой?
- Как Facade помогает при подготовке тестовых данных для E2E-тестов?
- Приведите пример использования Adapter в вашем тестовом фреймворке.

### Уровень Senior

- Как спроектировать фреймворк, который позволяет менять UI-библиотеку (Selenium, Playwright)
  без изменения тестов? Какие паттерны примените?
- Как вы решаете проблему "множества слоёв декораторов" при отладке?
- Когда Facade становится антипаттерном? Как распознать God Facade?
- Как структурные паттерны помогают при миграции тестового фреймворка?
- Как реализовать динамическое добавление декораторов на основе конфигурации?

---

## Практические задания

1. **Adapter.** Создайте интерфейс `BrowserDriver` с методами `openUrl()`, `click()`,
   `type()`, `getText()`. Реализуйте два адаптера: один для Selenium WebDriver, другой —
   заглушку (stub) для unit-тестирования Page Object-ов без браузера.

2. **Decorator.** Реализуйте базовый интерфейс `ApiAction` с методом `execute()`. Создайте
   три декоратора: `LoggingDecorator`, `TimingDecorator`, `RetryDecorator`. Покажите, как
   комбинировать их в разном порядке.

3. **Proxy.** Реализуйте `CachingProxy` для API-клиента, который кэширует GET-запросы
   и инвалидирует кэш при POST/PUT/DELETE. Напишите тест, проверяющий, что повторный
   GET не вызывает реальный запрос.

4. **Facade.** Создайте `TestEnvironmentFacade`, который одним вызовом:
   - Поднимает тестовую БД (или подключается к существующей).
   - Применяет миграции.
   - Загружает тестовые данные.
   - Возвращает готовый контекст для тестов.

5. **Комбинация.** Спроектируйте мини-фреймворк для API-тестирования, использующий все
   четыре паттерна: Adapter для HTTP-клиента, Decorator для логирования, Proxy для кэширования,
   Facade для подготовки тестовых данных.

---

## Дополнительные ресурсы

- **Refactoring Guru — Structural Patterns** — интерактивные диаграммы и примеры.
- **"Design Patterns: Elements of Reusable Object-Oriented Software"** — главы 4 (Structural).
- **Selenium PageFactory** — пример Proxy в реальном фреймворке (использует Java Dynamic Proxy).
- **"Working Effectively with Legacy Code"** — Michael Feathers — техники работы с Adapter
  при обёртывании legacy-кода.
- **Java Dynamic Proxy (java.lang.reflect.Proxy)** — встроенный механизм создания прокси в Java.
- **Spring AOP** — пример Decorator/Proxy на уровне фреймворка.
