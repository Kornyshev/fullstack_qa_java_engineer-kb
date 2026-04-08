# Подключение Allure Report

## Введение

Allure — это фреймворк для создания наглядных и информативных отчётов о тестировании.
Он превращает сухие логи прогона в интерактивный отчёт с графиками, шагами, скриншотами
и историей запусков. Allure — стандарт де-факто для отчётности в Java-автоматизации.

> **Цель:** Подключить Allure к существующему тестовому проекту, добавить аннотации,
> настроить снятие скриншотов при падении тестов и научиться анализировать отчёт.

---

## Шаг 1: Добавление Maven-зависимостей

### Полные добавления в pom.xml

```xml
<properties>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
    <allure.version>2.25.0</allure.version>
    <aspectj.version>1.9.21</aspectj.version>
    <junit5.version>5.10.2</junit5.version>
</properties>

<dependencies>
    <!-- Allure JUnit 5 интеграция — связывает Allure с JUnit 5 -->
    <dependency>
        <groupId>io.qameta.allure</groupId>
        <artifactId>allure-junit5</artifactId>
        <version>${allure.version}</version>
        <scope>test</scope>
    </dependency>

    <!-- Allure Selenide — автоматические шаги для Selenide-действий -->
    <dependency>
        <groupId>io.qameta.allure</groupId>
        <artifactId>allure-selenide</artifactId>
        <version>${allure.version}</version>
        <scope>test</scope>
    </dependency>

    <!-- Allure REST Assured — автоматические шаги для API-запросов -->
    <dependency>
        <groupId>io.qameta.allure</groupId>
        <artifactId>allure-rest-assured</artifactId>
        <version>${allure.version}</version>
        <scope>test</scope>
    </dependency>

    <!-- AspectJ Weaver — необходим для работы @Step-аннотаций -->
    <dependency>
        <groupId>org.aspectj</groupId>
        <artifactId>aspectjweaver</artifactId>
        <version>${aspectj.version}</version>
        <scope>test</scope>
    </dependency>

    <!-- JUnit 5 -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>${junit5.version}</version>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <!-- Maven Surefire Plugin — настройка запуска тестов с AspectJ -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>3.2.5</version>
            <configuration>
                <argLine>
                    -javaagent:"${settings.localRepository}/org/aspectj/aspectjweaver/${aspectj.version}/aspectjweaver-${aspectj.version}.jar"
                </argLine>
                <systemPropertyVariables>
                    <!-- Директория для результатов Allure -->
                    <allure.results.directory>
                        ${project.build.directory}/allure-results
                    </allure.results.directory>
                </systemPropertyVariables>
            </configuration>
        </plugin>

        <!-- Allure Maven Plugin — генерация отчёта -->
        <plugin>
            <groupId>io.qameta.allure</groupId>
            <artifactId>allure-maven</artifactId>
            <version>2.12.0</version>
            <configuration>
                <reportVersion>${allure.version}</reportVersion>
                <resultsDirectory>
                    ${project.build.directory}/allure-results
                </resultsDirectory>
            </configuration>
        </plugin>
    </plugins>
</build>
```

### Проверка подключения

```bash
# Скачиваем зависимости и проверяем, что всё компилируется
mvn clean compile -DskipTests

# Если ошибок нет — зависимости подключены корректно
```

---

## Шаг 2: Аннотации Allure — иерархия отчёта

### Структура аннотаций

Allure использует иерархию для группировки тестов в отчёте:

```
@Epic        → Крупный блок функциональности (например, "Управление статьями")
  @Feature   → Конкретная функция (например, "Создание статьи")
    @Story   → Пользовательская история (например, "Пользователь создаёт статью с тегами")
```

### Код: Аннотированный тестовый класс

```java
@Epic("Управление статьями")
@Feature("Создание статьи")
public class CreateArticleTest extends BaseTest {

    @Test
    @Story("Создание статьи с обязательными полями")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("Создание статьи с заголовком, описанием и телом")
    @Description("Проверяем, что авторизованный пользователь может создать " +
                 "статью, заполнив все обязательные поля")
    void testCreateArticleWithRequiredFields() {
        loginPage.loginAs("user@test.com", "password");
        editorPage.open();
        editorPage.fillTitle("Test Article");
        editorPage.fillDescription("Short description");
        editorPage.fillBody("Article body content");
        editorPage.submit();

        articlePage.verifyTitle("Test Article");
    }

    @Test
    @Story("Создание статьи с тегами")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("Создание статьи с несколькими тегами")
    void testCreateArticleWithTags() {
        loginPage.loginAs("user@test.com", "password");
        editorPage.open();
        editorPage.fillTitle("Tagged Article");
        editorPage.fillDescription("Article with tags");
        editorPage.fillBody("Content");
        editorPage.addTag("java");
        editorPage.addTag("automation");
        editorPage.submit();

        articlePage.verifyTags("java", "automation");
    }

    @Test
    @Story("Валидация при создании статьи")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("Ошибка при создании статьи без заголовка")
    void testCreateArticleWithoutTitle() {
        loginPage.loginAs("user@test.com", "password");
        editorPage.open();
        editorPage.fillDescription("Description only");
        editorPage.fillBody("Body only");
        editorPage.submit();

        editorPage.verifyErrorMessage("title can't be blank");
    }
}
```

---

## Шаг 3: Аннотация @Step — детализация шагов

### Код: Page Object с @Step

```java
public class LoginPage extends BasePage {

    private final By emailInput = By.cssSelector("input[placeholder='Email']");
    private final By passwordInput = By.cssSelector("input[placeholder='Password']");
    private final By submitButton = By.cssSelector("button[type='submit']");
    private final By errorMessages = By.cssSelector(".error-messages li");

    public LoginPage(WebDriver driver) {
        super(driver);
    }

    // @Step отображается как шаг в Allure-отчёте
    // Параметры метода автоматически подставляются в текст шага
    @Step("Открыть страницу логина")
    public LoginPage open() {
        driver.get(ConfigReader.getBaseUrl() + "/login");
        return this;
    }

    @Step("Ввести email: {email}")
    public LoginPage enterEmail(String email) {
        type(emailInput, email);
        return this;
    }

    @Step("Ввести пароль")
    public LoginPage enterPassword(String password) {
        // Пароль не выводим в отчёт из соображений безопасности
        type(passwordInput, password);
        return this;
    }

    @Step("Нажать кнопку «Sign in»")
    public void clickSignIn() {
        click(submitButton);
    }

    // Составной шаг — объединяет несколько действий
    @Step("Авторизоваться как {email}")
    public void loginAs(String email, String password) {
        open();
        enterEmail(email);
        enterPassword(password);
        clickSignIn();
    }

    @Step("Проверить сообщение об ошибке: {expectedMessage}")
    public void verifyErrorMessage(String expectedMessage) {
        WebElement error = waitForElement(errorMessages);
        assertEquals(expectedMessage, error.getText(),
                     "Сообщение об ошибке не совпадает");
    }
}
```

### Вложенные шаги в утилитах

```java
public class DataGenerator {

    @Step("Сгенерировать уникальный email")
    public static String uniqueEmail() {
        String email = "test_" + System.currentTimeMillis() + "@test.com";
        // Allure.addAttachment позволяет прикрепить данные к шагу
        Allure.addAttachment("Сгенерированный email", "text/plain", email);
        return email;
    }

    @Step("Сгенерировать данные для статьи")
    public static Map<String, String> articleData() {
        Faker faker = new Faker();
        Map<String, String> data = new HashMap<>();
        data.put("title", faker.book().title() + " " + System.currentTimeMillis());
        data.put("description", faker.lorem().sentence());
        data.put("body", faker.lorem().paragraph());

        // Прикрепляем сгенерированные данные к отчёту для отладки
        Allure.addAttachment("Данные статьи", "application/json",
            new ObjectMapper().writeValueAsString(data));

        return data;
    }
}
```

---

## Шаг 4: Скриншоты при падении теста

### Код: TestWatcher для JUnit 5

```java
public class AllureScreenshotExtension implements TestWatcher {

    @Override
    public void testFailed(ExtensionContext context, Throwable cause) {
        // Получаем WebDriver из контекста теста
        Object testInstance = context.getRequiredTestInstance();

        if (testInstance instanceof BaseTest) {
            WebDriver driver = ((BaseTest) testInstance).getDriver();

            if (driver != null) {
                attachScreenshot(driver);
                attachPageSource(driver);
                attachBrowserLogs(driver);
            }
        }
    }

    @Attachment(value = "Скриншот при падении", type = "image/png")
    private byte[] attachScreenshot(WebDriver driver) {
        return ((TakesScreenshot) driver).getScreenshotAs(OutputType.BYTES);
    }

    @Attachment(value = "HTML-код страницы", type = "text/html")
    private String attachPageSource(WebDriver driver) {
        return driver.getPageSource();
    }

    @Attachment(value = "Логи браузера", type = "text/plain")
    private String attachBrowserLogs(WebDriver driver) {
        try {
            // Собираем логи консоли браузера — помогает при отладке JS-ошибок
            return driver.manage().logs()
                .get(LogType.BROWSER)
                .getAll()
                .stream()
                .map(LogEntry::toString)
                .collect(Collectors.joining("\n"));
        } catch (Exception e) {
            return "Логи браузера недоступны: " + e.getMessage();
        }
    }
}
```

### Код: BaseTest с подключённым Extension

```java
// Подключаем Extension ко всем тестам через наследование
@ExtendWith(AllureScreenshotExtension.class)
public abstract class BaseTest {

    protected WebDriver driver;
    protected LoginPage loginPage;
    protected EditorPage editorPage;
    protected ArticlePage articlePage;

    @BeforeEach
    void setUp() {
        driver = WebDriverFactory.createDriver();
        loginPage = new LoginPage(driver);
        editorPage = new EditorPage(driver);
        articlePage = new ArticlePage(driver);
    }

    @AfterEach
    void tearDown() {
        if (driver != null) {
            driver.quit();
        }
    }

    // Метод для доступа к driver из Extension
    public WebDriver getDriver() {
        return driver;
    }
}
```

---

## Шаг 5: Файл environment.properties

Allure может отображать информацию об окружении в отчёте. Для этого создайте файл
`allure-results/environment.properties` после прогона тестов.

### Файл: src/test/resources/allure.properties

```properties
# Директория для результатов — Allure будет искать данные здесь
allure.results.directory=target/allure-results
```

### Код: Генерация environment.properties

```java
public class AllureEnvironmentWriter {

    // Вызывается один раз перед стартом тестов
    public static void writeEnvironment() {
        Properties props = new Properties();
        props.setProperty("Browser", ConfigReader.getBrowser());
        props.setProperty("Browser.Version", getBrowserVersion());
        props.setProperty("Base.URL", ConfigReader.getBaseUrl());
        props.setProperty("OS", System.getProperty("os.name"));
        props.setProperty("Java.Version", System.getProperty("java.version"));
        props.setProperty("Timestamp", LocalDateTime.now().toString());

        Path resultsDir = Paths.get("target", "allure-results");
        try {
            Files.createDirectories(resultsDir);
            try (OutputStream out = Files.newOutputStream(
                    resultsDir.resolve("environment.properties"))) {
                props.store(out, "Allure Environment");
            }
        } catch (IOException e) {
            throw new RuntimeException(
                "Не удалось записать environment.properties: " + e.getMessage());
        }
    }
}
```

---

## Шаг 6: Файл categories.json — классификация дефектов

Создайте файл `src/test/resources/categories.json` для группировки упавших тестов по категориям.

```json
[
  {
    "name": "Дефекты продукта",
    "description": "Тесты, упавшие из-за ошибки в приложении",
    "matchedStatuses": ["failed"],
    "messageRegex": ".*AssertionError.*"
  },
  {
    "name": "Проблемы инфраструктуры",
    "description": "Тесты, упавшие из-за проблем с окружением",
    "matchedStatuses": ["broken"],
    "messageRegex": ".*TimeoutException.*|.*ConnectionRefused.*|.*WebDriverException.*"
  },
  {
    "name": "Известные дефекты",
    "description": "Тесты, связанные с известными багами",
    "matchedStatuses": ["failed"],
    "flaky": false,
    "traceRegex": ".*KnownIssue.*"
  },
  {
    "name": "Нестабильные тесты (Flaky)",
    "description": "Тесты, которые иногда падают без видимой причины",
    "matchedStatuses": ["failed", "broken"],
    "flaky": true
  }
]
```

### Копирование categories.json в результаты

```java
// Добавьте в AllureEnvironmentWriter
public static void copyCategories() {
    try (InputStream input = AllureEnvironmentWriter.class.getClassLoader()
            .getResourceAsStream("categories.json")) {
        Path target = Paths.get("target", "allure-results", "categories.json");
        Files.createDirectories(target.getParent());
        Files.copy(input, target, StandardCopyOption.REPLACE_EXISTING);
    } catch (IOException e) {
        System.err.println("Не удалось скопировать categories.json: " + e.getMessage());
    }
}
```

---

## Шаг 7: Генерация и анализ отчёта

### Команды для генерации

```bash
# 1. Запуск тестов — результаты попадают в target/allure-results
mvn clean test

# 2. Генерация HTML-отчёта и открытие в браузере
mvn allure:serve

# 3. Или генерация отчёта без открытия (для CI/CD)
mvn allure:report

# 4. Отчёт будет в target/site/allure-maven-plugin/index.html
```

### Альтернатива: Allure CLI

```bash
# Установка Allure CLI (macOS)
brew install allure

# Генерация и открытие отчёта
allure serve target/allure-results

# Генерация отчёта в директорию
allure generate target/allure-results -o target/allure-report --clean
```

### Что анализировать в отчёте

| Раздел | Что смотреть |
|--------|-------------|
| **Overview** | Общий процент прохождения, тренд (если есть история) |
| **Suites** | Группировка по тестовым классам, время выполнения каждого теста |
| **Graphs** | Распределение по severity, длительность тестов |
| **Behaviors** | Группировка по Epic/Feature/Story — бизнес-представление |
| **Categories** | Классификация падений: дефекты продукта vs инфраструктурные проблемы |
| **Timeline** | Хронология выполнения тестов — видно параллелизм |
| **Packages** | Группировка по пакетам Java |

---

## Шаг 8: Allure для REST Assured

### Настройка фильтра

```java
public class ApiTest {

    @BeforeAll
    static void setupAllureFilter() {
        // Фильтр автоматически логирует все HTTP-запросы и ответы в Allure
        RestAssured.filters(new AllureRestAssured());
    }

    @Test
    @Epic("API")
    @Feature("Авторизация")
    @Story("Успешный логин")
    void testLoginApi() {
        String body = "{\"user\":{\"email\":\"user@test.com\"," +
                      "\"password\":\"password123\"}}";

        given()
            .contentType(ContentType.JSON)
            .body(body)
        .when()
            .post("/api/users/login")
        .then()
            .statusCode(200)
            .body("user.token", notNullValue());
        // В Allure-отчёте будут видны: URL, заголовки, тело запроса и ответа
    }
}
```

---

## Практическое задание

### Задание 1: Базовое подключение Allure

1. Добавьте зависимости Allure в существующий Maven-проект
2. Аннотируйте минимум 5 тестов: `@Epic`, `@Feature`, `@Story`, `@Severity`
3. Сгенерируйте отчёт командой `mvn clean test allure:serve`
4. Сделайте скриншот раздела Behaviors в отчёте

**Ожидаемый результат:** Allure-отчёт открывается в браузере, тесты сгруппированы
по Epic/Feature/Story, severity отображается корректно.

### Задание 2: Шаги и скриншоты

1. Добавьте `@Step` аннотации во все методы Page Object-классов
2. Реализуйте `AllureScreenshotExtension` для снятия скриншотов при падении
3. Намеренно сломайте один тест и убедитесь, что скриншот прикрепляется к отчёту
4. Добавьте прикрепление HTML-кода страницы и логов браузера

**Ожидаемый результат:** В упавшем тесте видны все шаги, скриншот момента падения,
HTML страницы и логи браузера.

### Задание 3: Полная настройка

1. Создайте `categories.json` с минимум 3 категориями
2. Реализуйте генерацию `environment.properties`
3. Настройте `AllureRestAssured` фильтр для API-тестов
4. Сгенерируйте финальный отчёт и проверьте все разделы

**Критерии оценки:**
- Отчёт содержит корректную иерархию Epic/Feature/Story
- Упавшие тесты содержат скриншоты и логи
- Раздел Categories отображает пользовательские категории
- Раздел Environment показывает информацию об окружении
- API-тесты содержат логи HTTP-запросов и ответов

---

## Чек-лист самопроверки

- [ ] pom.xml содержит все необходимые зависимости Allure
- [ ] AspectJ Weaver подключён через maven-surefire-plugin
- [ ] Тесты аннотированы @Epic, @Feature, @Story, @Severity
- [ ] Page Object-методы аннотированы @Step
- [ ] Скриншоты снимаются автоматически при падении теста
- [ ] Создан categories.json для классификации падений
- [ ] Environment.properties генерируется при запуске
- [ ] Allure-отчёт генерируется командой `mvn allure:serve`
- [ ] API-тесты логируют HTTP-запросы через AllureRestAssured
- [ ] Отчёт содержит данные во всех ключевых разделах
