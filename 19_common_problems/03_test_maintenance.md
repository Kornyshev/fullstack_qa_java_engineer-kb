# Поддержка тестов: управление тестовым кодом

## Обзор

Поддержка автотестов (test maintenance) — это постоянная деятельность по обновлению, рефакторингу
и удалению тестов в ответ на изменения в тестируемом приложении, инфраструктуре и требованиях.
По мере роста тестового набора стоимость поддержки растёт экспоненциально: команда тратит больше
времени на обновление существующих тестов, чем на написание новых.

В зрелых проектах стоимость поддержки может достигать 60-70% от общих затрат на автоматизацию.
Умение эффективно управлять этой стоимостью — ключевой навык для Senior QA Engineer.
На собеседованиях часто просят описать стратегию управления большим test suite и методы
снижения стоимости поддержки.

---

## Основные причины роста стоимости поддержки

### 1. Stale Locators после UI Redesign

Самая частая причина массовых падений UI-тестов — изменение локаторов после редизайна
или рефакторинга фронтенда.

**Проблема:**
```java
// Локаторы, завязанные на структуру HTML — ломаются при любом изменении вёрстки
driver.findElement(By.xpath("//div[@class='main-content']/div[2]/form/div[3]/input"));
driver.findElement(By.cssSelector("div.header > ul > li:nth-child(4) > a"));
```

**Решение — устойчивые локаторы:**
```java
// Стратегия 1: data-testid атрибуты (договорённость с разработчиками)
driver.findElement(By.cssSelector("[data-testid='login-button']"));

// Стратегия 2: семантические локаторы
driver.findElement(By.id("submit-order"));
driver.findElement(By.name("username"));

// Стратегия 3: текстовые локаторы для стабильного контента
driver.findElement(By.linkText("Войти в систему"));
```

**Архитектурный подход — Page Object с централизованными локаторами:**
```java
/**
 * Page Object для страницы логина.
 * Все локаторы сосредоточены в одном месте — при изменении UI
 * нужно обновить только этот класс.
 */
public class LoginPage {

    // Все локаторы объявлены как константы в одном месте
    private static final By USERNAME_INPUT = By.cssSelector("[data-testid='username']");
    private static final By PASSWORD_INPUT = By.cssSelector("[data-testid='password']");
    private static final By LOGIN_BUTTON = By.cssSelector("[data-testid='login-btn']");
    private static final By ERROR_MESSAGE = By.cssSelector("[data-testid='error-msg']");

    private final WebDriver driver;
    private final WebDriverWait wait;

    public LoginPage(WebDriver driver) {
        this.driver = driver;
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(10));
    }

    // Методы инкапсулируют взаимодействие с UI
    @Step("Ввести логин: {username}")
    public LoginPage enterUsername(String username) {
        wait.until(ExpectedConditions.visibilityOfElementLocated(USERNAME_INPUT))
            .sendKeys(username);
        return this;
    }

    @Step("Ввести пароль")
    public LoginPage enterPassword(String password) {
        driver.findElement(PASSWORD_INPUT).sendKeys(password);
        return this;
    }

    @Step("Нажать кнопку 'Войти'")
    public DashboardPage clickLogin() {
        driver.findElement(LOGIN_BUTTON).click();
        return new DashboardPage(driver);
    }

    @Step("Получить текст ошибки")
    public String getErrorMessage() {
        return wait.until(ExpectedConditions.visibilityOfElementLocated(ERROR_MESSAGE))
            .getText();
    }
}
```

### 2. API Contract Changes (изменение API-контрактов)

Изменения в REST API (новые обязательные поля, изменение формата ответа, переименование endpoint)
ломают API-тесты.

**Типичные сценарии:**
- Добавили обязательное поле в запрос → тесты получают `400 Bad Request`
- Переименовали поле в ответе → десериализация падает
- Изменили URL endpoint → `404 Not Found`
- Добавили авторизацию → `401 Unauthorized`

**Стратегия защиты:**

```java
/**
 * Фабрика тестовых объектов — централизованное создание запросов.
 * При изменении контракта обновляем только фабрику.
 */
public class RequestFactory {

    // Метод создаёт валидный объект запроса со всеми обязательными полями
    public static CreateUserRequest validCreateUserRequest() {
        return CreateUserRequest.builder()
            .firstName("Test")
            .lastName("User")
            .email("test_" + System.currentTimeMillis() + "@example.com")
            .phone("+79001234567")
            .role("USER")
            // Новое обязательное поле — добавляем ОДИН РАЗ здесь,
            // а не в каждом из 50 тестов
            .acceptTerms(true)
            .build();
    }

    // Запрос с кастомными данными — для конкретных тестовых сценариев
    public static CreateUserRequest createUserRequest(
            Consumer<CreateUserRequest.CreateUserRequestBuilder> customizer) {
        var builder = CreateUserRequest.builder()
            .firstName("Test")
            .lastName("User")
            .email("test_" + System.currentTimeMillis() + "@example.com")
            .phone("+79001234567")
            .role("USER")
            .acceptTerms(true);

        customizer.accept(builder);
        return builder.build();
    }
}

// Использование в тестах — чисто и устойчиво к изменениям контракта
class UserApiTest {

    @Test
    void testCreateUser() {
        // Стандартный запрос — берём из фабрики
        CreateUserRequest request = RequestFactory.validCreateUserRequest();
        Response response = apiClient.createUser(request);
        assertEquals(201, response.getStatusCode());
    }

    @Test
    void testCreateUserWithInvalidEmail() {
        // Кастомизируем только нужное поле
        CreateUserRequest request = RequestFactory.createUserRequest(
            builder -> builder.email("invalid-email")
        );
        Response response = apiClient.createUser(request);
        assertEquals(400, response.getStatusCode());
    }
}
```

### 3. Test Suite Growth Management (управление ростом test suite)

По мере роста проекта количество тестов увеличивается, и без управления suite превращается
в хаотичную массу, где невозможно понять, что покрыто, а что нет.

**Метрики для управления ростом:**

| Метрика | Формула | Целевое значение |
|---------|---------|-----------------|
| Тестов на фичу | Количество тестов / Количество фич | 5-15 |
| Время прогона | Общее время suite | < 30 мин для smoke, < 2 ч для regression |
| Test Maintenance Ratio | Время поддержки / Время написания новых | < 1:1 |
| Dead Test Rate | Тесты, не запускавшиеся 30+ дней / Всего тестов | < 5% |

**Организация test suite:**

```
tests/
├── smoke/                  # Критические сценарии (5-10 мин)
│   ├── LoginSmokeTest
│   └── PaymentSmokeTest
├── regression/             # Полная регрессия (1-2 часа)
│   ├── user/
│   │   ├── UserRegistrationTest
│   │   └── UserProfileTest
│   ├── payment/
│   │   ├── PaymentProcessingTest
│   │   └── RefundTest
│   └── catalog/
│       └── ProductSearchTest
├── integration/            # Интеграционные тесты
│   ├── PaymentGatewayIntTest
│   └── EmailServiceIntTest
└── e2e/                    # Сквозные сценарии
    ├── PurchaseFlowE2ETest
    └── RegistrationFlowE2ETest
```

### 4. Tech Debt в тестовом коде

Технический долг накапливается в тестовом коде так же, как в продуктовом:
копипаста, устаревшие паттерны, отсутствие рефакторинга.

**Признаки технического долга в тестах:**

```java
// ПЛОХО — копипаста: один и тот же setup в каждом тесте
class OrderTest1 {
    @Test
    void test1() {
        WebDriver driver = new ChromeDriver();
        driver.get("http://app.test.com");
        driver.findElement(By.id("username")).sendKeys("admin");
        driver.findElement(By.id("password")).sendKeys("admin123");
        driver.findElement(By.id("login-btn")).click();
        // ... тест
        driver.quit();
    }
}
class OrderTest2 {
    @Test
    void test2() {
        WebDriver driver = new ChromeDriver();
        driver.get("http://app.test.com");
        driver.findElement(By.id("username")).sendKeys("admin");
        driver.findElement(By.id("password")).sendKeys("admin123");
        driver.findElement(By.id("login-btn")).click();
        // ... тест
        driver.quit();
    }
}

// ХОРОШО — общий базовый класс и Page Objects
abstract class BaseUiTest {

    protected WebDriver driver;
    protected LoginPage loginPage;

    @BeforeEach
    void setUp() {
        driver = DriverFactory.createDriver();
        loginPage = new LoginPage(driver);
    }

    @AfterEach
    void tearDown() {
        if (driver != null) {
            driver.quit();
        }
    }

    protected DashboardPage loginAsAdmin() {
        return loginPage
            .enterUsername(TestUsers.ADMIN.getUsername())
            .enterPassword(TestUsers.ADMIN.getPassword())
            .clickLogin();
    }
}

class OrderTest extends BaseUiTest {
    @Test
    void testCreateOrder() {
        DashboardPage dashboard = loginAsAdmin();
        // ... чистый тестовый код
    }
}
```

---

## Когда удалять тесты

Удаление тестов — это нормальная и необходимая практика. Тесты не являются священными.

### Критерии для удаления

| Критерий | Описание | Пример |
|----------|----------|--------|
| **Obsolete** | Функциональность удалена | Тест на старую версию оплаты |
| **Redundant** | Дублирует другой тест | 3 UI-теста проверяют одно и то же |
| **Low Value** | Стоимость поддержки > ценности | Тест на trivial getter |
| **Permanently Flaky** | Не удаётся стабилизировать | Тест на стороннюю интеграцию |
| **Superseded** | Заменён лучшим тестом | E2E-тест, заменённый API-тестом |

### Процесс принятия решения

```java
/**
 * Аудит тестов — определение кандидатов на удаление.
 * Запускается периодически (раз в спринт/месяц).
 */
public class TestAudit {

    // Критерии для отнесения теста к кандидатам на удаление
    public List<TestAuditResult> findDeletionCandidates(List<TestInfo> tests) {
        return tests.stream()
            .map(test -> {
                List<String> reasons = new ArrayList<>();

                // Тест не запускался более 30 дней
                if (test.getLastRunDate().isBefore(
                        LocalDate.now().minusDays(30))) {
                    reasons.add("Не запускался более 30 дней");
                }

                // Тест находится в карантине более 2 недель
                if (test.isQuarantined() && test.getQuarantineDate()
                        .isBefore(LocalDate.now().minusWeeks(2))) {
                    reasons.add("В карантине более 2 недель");
                }

                // Flakiness Rate > 50%
                if (test.getFlakinessRate() > 0.5) {
                    reasons.add("Flakiness Rate > 50%");
                }

                return new TestAuditResult(test, reasons);
            })
            .filter(result -> !result.getReasons().isEmpty())
            .collect(Collectors.toList());
    }
}
```

---

## Практики Code Review для тестов

### Чек-лист для ревью тестового кода

```
## Общее
□ Тест имеет осмысленное имя, описывающее сценарий
□ Один тест проверяет одну вещь (Single Responsibility)
□ Тест независим от других тестов (нет зависимости от порядка)
□ Тестовые данные изолированы (уникальные для каждого теста)

## Структура
□ Используется паттерн Arrange-Act-Assert (Given-When-Then)
□ Setup и Teardown вынесены в @BeforeEach / @AfterEach
□ Общая логика вынесена в базовый класс или утилиты
□ Нет копипасты между тестами

## Локаторы (UI-тесты)
□ Используются устойчивые локаторы (id, data-testid)
□ Локаторы определены в Page Object, не в тестах
□ Нет хрупких xpath с позиционными индексами

## Assertions
□ Assertion messages информативны
□ Используется один логический assert (не 10 подряд)
□ Нет скрытых assertions (assert внутри утилитных методов без @Step)

## Ожидания
□ Нет Thread.sleep
□ Используются Explicit Waits с осмысленными таймаутами
□ Ожидания конкретных условий, а не фиксированного времени

## Данные
□ Нет хардкода (URL, credentials, test data)
□ Тестовые данные генерируются или берутся из конфигурации
□ Cleanup данных в tearDown (даже при падении теста)
```

### Автоматизация code review

```java
/**
 * Кастомное правило для архитектурных тестов (ArchUnit).
 * Проверяет, что тестовый код соответствует архитектурным стандартам.
 */
@AnalyzeClasses(packages = "com.example.tests")
class TestArchitectureRules {

    // Тесты не должны напрямую создавать WebDriver
    @ArchTest
    static final ArchRule noDirectDriverCreation =
        noClasses()
            .that().haveNameMatching(".*Test")
            .should().callConstructor(ChromeDriver.class)
            .because("Используйте DriverFactory для создания драйвера");

    // Тесты не должны использовать Thread.sleep
    @ArchTest
    static final ArchRule noThreadSleep =
        noClasses()
            .that().haveNameMatching(".*Test")
            .should().callMethod(Thread.class, "sleep", long.class)
            .because("Используйте WebDriverWait вместо Thread.sleep");

    // Page Objects должны быть в пакете pages
    @ArchTest
    static final ArchRule pageObjectsInCorrectPackage =
        classes()
            .that().haveNameMatching(".*Page")
            .should().resideInAPackage("..pages..")
            .because("Page Objects должны находиться в пакете pages");
}
```

---

## Стратегии рефакторинга тестов

### 1. Extract Common Steps

```java
// ДО рефакторинга — повторяющиеся шаги в каждом тесте
@Test
void testPurchase() {
    api.post("/auth/login", loginBody).then().statusCode(200);
    String token = api.post("/auth/login", loginBody).extract().path("token");
    api.given().header("Authorization", "Bearer " + token)
        .post("/cart/add", cartItem).then().statusCode(200);
    // ...
}

// ПОСЛЕ рефакторинга — общие шаги в хелпере
@Test
void testPurchase() {
    String token = authHelper.loginAndGetToken("user", "pass");
    cartHelper.addItem(token, "product-123");
    // ... только бизнес-логика теста
}
```

### 2. Parameterized Tests вместо копипасты

```java
// ДО — 5 отдельных тестов с одинаковой структурой
@Test void testValidEmail1() { validateEmail("user@mail.com", true); }
@Test void testValidEmail2() { validateEmail("admin@corp.ru", true); }
@Test void testInvalidEmail1() { validateEmail("noatsign", false); }
@Test void testInvalidEmail2() { validateEmail("@nodomain", false); }
@Test void testInvalidEmail3() { validateEmail("", false); }

// ПОСЛЕ — один параметризованный тест
@ParameterizedTest(name = "Email \"{0}\" → валидный: {1}")
@CsvSource({
    "user@mail.com, true",
    "admin@corp.ru, true",
    "noatsign, false",
    "@nodomain, false",
    "'', false"
})
void testEmailValidation(String email, boolean expectedValid) {
    assertEquals(expectedValid, validator.isValid(email));
}
```

### 3. Test Data Builders

```java
/**
 * Builder для тестовых данных — чистая альтернатива конструкторам с 10 параметрами.
 * Позволяет задавать только релевантные для теста поля.
 */
public class TestOrderBuilder {

    private String orderId = "ORD-" + UUID.randomUUID().toString().substring(0, 8);
    private String userId = "USER-" + UUID.randomUUID().toString().substring(0, 8);
    private BigDecimal amount = new BigDecimal("99.99");
    private String currency = "RUB";
    private OrderStatus status = OrderStatus.NEW;
    private List<OrderItem> items = List.of(
        new OrderItem("product-1", 1, new BigDecimal("99.99"))
    );

    public TestOrderBuilder withAmount(BigDecimal amount) {
        this.amount = amount;
        return this;
    }

    public TestOrderBuilder withStatus(OrderStatus status) {
        this.status = status;
        return this;
    }

    public TestOrderBuilder withItems(List<OrderItem> items) {
        this.items = items;
        return this;
    }

    public Order build() {
        return new Order(orderId, userId, amount, currency, status, items);
    }
}

// В тесте указываем только то, что важно для сценария
@Test
void testHighValueOrderRequiresApproval() {
    Order order = new TestOrderBuilder()
        .withAmount(new BigDecimal("100000.00"))
        .build();

    OrderResult result = orderService.process(order);
    assertEquals(OrderStatus.PENDING_APPROVAL, result.getStatus());
}
```

---

## Метрики стоимости поддержки тестов

### Ключевые метрики

| Метрика | Описание | Как считать |
|---------|----------|-------------|
| **Test Maintenance Cost** | Время на поддержку vs написание новых тестов | Трекинг задач в Jira |
| **Broken Test Fix Time** | Среднее время исправления сломанного теста | От падения до зелёного |
| **Test Code Churn** | Частота изменений тестового кода | git log --stat |
| **Locator Stability** | Процент локаторов, менявшихся за спринт | Анализ diff в Page Objects |
| **Test Suite Execution Time** | Общее время прогона | CI метрики |
| **Tests per Feature** | Среднее количество тестов на фичу | Count / features |

### Трекинг стоимости

```java
/**
 * Анализатор стоимости поддержки тестов.
 * Использует данные из git-истории и CI для расчёта метрик.
 */
public class MaintenanceCostAnalyzer {

    // Расчёт Test Code Churn — как часто меняется тестовый код
    public Map<String, Integer> calculateChurn(
            List<GitCommit> commits, String testDirectory) {
        Map<String, Integer> fileChanges = new HashMap<>();

        for (GitCommit commit : commits) {
            commit.getChangedFiles().stream()
                .filter(f -> f.startsWith(testDirectory))
                .forEach(f -> fileChanges.merge(f, 1, Integer::sum));
        }

        return fileChanges.entrySet().stream()
            .sorted(Map.Entry.<String, Integer>comparingByValue().reversed())
            .collect(Collectors.toMap(
                Map.Entry::getKey, Map.Entry::getValue,
                (a, b) -> a, LinkedHashMap::new
            ));
    }

    // Файлы с высоким churn — кандидаты на рефакторинг
    public List<String> getHighChurnFiles(
            Map<String, Integer> churn, int threshold) {
        return churn.entrySet().stream()
            .filter(e -> e.getValue() > threshold)
            .map(Map.Entry::getKey)
            .collect(Collectors.toList());
    }
}
```

---

## Связь с тестированием

Поддержка тестов — неотъемлемая часть жизненного цикла тестирования:

- **Актуальность покрытия** — без обновления тесты перестают отражать реальное поведение системы
- **Доверие к результатам** — устаревшие тесты дают ложные срабатывания или пропускают баги
- **Скорость поставки** — тяжёлый в поддержке suite замедляет CI/CD
- **Стоимость QA** — чем выше стоимость поддержки, тем меньше ресурсов на новые тесты
- **Мотивация команды** — бесконечное «чинить тесты» демотивирует

---

## Типичные ошибки

1. **Не выделять время на рефакторинг тестов** — «у нас нет времени на рефакторинг»
2. **Хранить все локаторы в тестах, а не в Page Objects** — любое изменение UI → правки в 50 тестах
3. **Не удалять устаревшие тесты** — suite растёт, время прогона увеличивается, доверие падает
4. **Копировать тесты вместо параметризации** — 10 одинаковых тестов с разными данными
5. **Не проводить code review тестов** — к тестовому коду применяют меньше стандартов
6. **Не отслеживать метрики поддержки** — без данных невозможно управлять стоимостью
7. **Игнорировать технический долг в тестах** — «это же тесты, не продуктовый код»
8. **Не договариваться с разработчиками о data-testid** — QA и DEV работают изолированно

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Что такое Page Object Pattern? Зачем он нужен?
2. Почему важно использовать устойчивые локаторы?
3. Когда стоит удалять тесты?
4. Что такое Test Data Builder?

### 🟡 Средний уровень
5. Как организовать test suite для проекта с 500+ тестами?
6. Какие метрики вы используете для оценки стоимости поддержки тестов?
7. Как проводить code review тестового кода? На что обращаете внимание?
8. Как параметризованные тесты помогают снизить дублирование?
9. Как минимизировать влияние изменения API-контрактов на тесты?

### 🔴 Продвинутый уровень
10. Опишите процесс рефакторинга test suite из 1000+ тестов с высоким техническим долгом.
11. Как выстроить процесс совместной работы QA и DEV для поддержания testability?
12. Как обосновать выделение спринта на рефакторинг тестов?
13. Как организовать архитектурные тесты (ArchUnit) для тестового кода?
14. Какова оптимальная стратегия тестирования (Test Pyramid) для снижения стоимости поддержки?

---

## Практические задания

### Задание 1: Рефакторинг Page Object
Дан Page Object с 50 локаторами и 30 методами. 20 локаторов используются только в одном тесте.
Предложите стратегию рефакторинга: какие паттерны применить, как декомпозировать класс.

### Задание 2: Аудит test suite
Напишите утилиту, которая:
- Анализирует git-историю тестовых файлов
- Выявляет файлы с высоким churn (> 10 коммитов за месяц)
- Определяет тесты, не запускавшиеся более 30 дней
- Генерирует отчёт с рекомендациями (удалить, рефакторить, оставить)

### Задание 3: Request Factory
Спроектируйте `RequestFactory` для REST API с 10 эндпоинтами. Каждый метод фабрики
должен создавать валидный запрос с дефолтными значениями, но позволять кастомизировать
любое поле. Используйте паттерн Builder.

### Задание 4: ArchUnit правила
Напишите набор ArchUnit-правил, которые проверяют:
- Все тесты наследуют `BaseTest`
- Page Objects находятся в пакете `pages`
- Нет прямых обращений к `WebDriver` из тестовых классов
- Нет `Thread.sleep` в тестовом коде
- Все тестовые классы имеют суффикс `Test`

---

## Дополнительные ресурсы

- [Page Object Pattern — Selenium](https://www.selenium.dev/documentation/test_practices/encouraged/page_object_models/) — официальная документация
- [ArchUnit User Guide](https://www.archunit.org/userguide/html/000_Index.html) — архитектурные тесты
- [JUnit 5: Parameterized Tests](https://junit.org/junit5/docs/current/user-guide/#writing-tests-parameterized-tests) — параметризованные тесты
- Martin Fowler: "PageObject" — описание паттерна от автора
- xUnit Patterns: "Test Maintenance" — паттерны поддержки тестов
- [Effective Testing with RSpec 3](https://pragprog.com/titles/rspec3/effective-testing-with-rspec-3/) — принципы применимы к любому языку
