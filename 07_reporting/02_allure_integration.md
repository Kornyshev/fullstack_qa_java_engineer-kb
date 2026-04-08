# Интеграция Allure с тестовыми фреймворками

## Обзор

Allure Framework сам по себе — лишь генератор отчётов. Чтобы он работал, необходимо
интегрировать его с тестовым фреймворком (JUnit 5, TestNG, Cucumber), системой сборки
(Maven, Gradle), а также настроить автоматическое создание вложений — скриншотов,
логов, page source и видео.

В этом разделе рассматривается полный цикл интеграции: от зависимостей в `pom.xml`
до рабочего примера с автоматическими скриншотами при падении теста.

Ключевые компоненты интеграции:
- **allure-junit5** / **allure-testng** — адаптеры для тестовых фреймворков
- **AspectJ Weaver** — обеспечивает работу аннотации `@Step` на произвольных методах
- **TestWatcher** (JUnit 5) / **ITestListener** (TestNG) — перехват падений для создания вложений
- **Maven Surefire Plugin** / **Gradle Allure Plugin** — запуск тестов с поддержкой Allure
- **Allure CLI** — `allure serve` и `allure generate` для просмотра отчётов

---

## Интеграция с JUnit 5

### Зависимости Maven

```xml
<properties>
    <java.version>17</java.version>
    <allure.version>2.25.0</allure.version>
    <aspectj.version>1.9.21</aspectj.version>
    <junit.version>5.10.2</junit.version>
    <selenium.version>4.17.0</selenium.version>
</properties>

<dependencies>
    <!-- JUnit 5 -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>${junit.version}</version>
        <scope>test</scope>
    </dependency>

    <!-- Allure JUnit 5 адаптер -->
    <dependency>
        <groupId>io.qameta.allure</groupId>
        <artifactId>allure-junit5</artifactId>
        <version>${allure.version}</version>
        <scope>test</scope>
    </dependency>

    <!-- Allure Selenide (если используется Selenide) -->
    <dependency>
        <groupId>io.qameta.allure</groupId>
        <artifactId>allure-selenide</artifactId>
        <version>${allure.version}</version>
        <scope>test</scope>
    </dependency>

    <!-- Allure REST Assured (для API-тестов) -->
    <dependency>
        <groupId>io.qameta.allure</groupId>
        <artifactId>allure-rest-assured</artifactId>
        <version>${allure.version}</version>
        <scope>test</scope>
    </dependency>

    <!-- Selenium WebDriver -->
    <dependency>
        <groupId>org.seleniumhq.selenium</groupId>
        <artifactId>selenium-java</artifactId>
        <version>${selenium.version}</version>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <!-- Maven Surefire Plugin с AspectJ Weaver -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>3.2.5</version>
            <configuration>
                <argLine>
                    -javaagent:"${settings.localRepository}/org/aspectj/aspectjweaver/${aspectj.version}/aspectjweaver-${aspectj.version}.jar"
                </argLine>
                <systemPropertyVariables>
                    <allure.results.directory>
                        ${project.build.directory}/allure-results
                    </allure.results.directory>
                </systemPropertyVariables>
            </configuration>
            <dependencies>
                <dependency>
                    <groupId>org.aspectj</groupId>
                    <artifactId>aspectjweaver</artifactId>
                    <version>${aspectj.version}</version>
                </dependency>
            </dependencies>
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

### AllureJunit5 Listener

Listener `AllureJunit5` подключается автоматически через ServiceLoader.
JUnit 5 обнаруживает его через файл:
`META-INF/services/org.junit.jupiter.api.extension.Extension`

Этот listener перехватывает жизненный цикл теста и записывает результаты
в `allure-results/`.

Для автоматической регистрации расширений нужно добавить в
`src/test/resources/junit-platform.properties`:

```properties
# Автоматическое обнаружение расширений (включая Allure)
junit.jupiter.extensions.autodetection.enabled=true
```

---

## Интеграция с TestNG

### Зависимости Maven

```xml
<dependency>
    <groupId>io.qameta.allure</groupId>
    <artifactId>allure-testng</artifactId>
    <version>${allure.version}</version>
    <scope>test</scope>
</dependency>
```

### Подключение через testng.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE suite SYSTEM "http://testng.org/testng-1.0.dtd">
<suite name="Test Suite">
    <listeners>
        <!-- Allure listener для TestNG -->
        <listener class-name="io.qameta.allure.testng.AllureTestNg"/>
    </listeners>
    <test name="Regression Tests">
        <classes>
            <class name="com.example.tests.LoginTests"/>
        </classes>
    </test>
</suite>
```

### Скриншоты при падении через ITestListener (TestNG)

```java
import io.qameta.allure.Allure;
import org.testng.ITestListener;
import org.testng.ITestResult;

public class AllureTestNgListener implements ITestListener {

    @Override
    public void onTestFailure(ITestResult result) {
        // Получаем WebDriver из тестового класса
        Object testInstance = result.getInstance();
        if (testInstance instanceof BaseTest baseTest) {
            WebDriver driver = baseTest.getDriver();

            // Делаем скриншот
            byte[] screenshot = ((TakesScreenshot) driver)
                    .getScreenshotAs(OutputType.BYTES);
            Allure.addAttachment("Скриншот при падении",
                    "image/png", new ByteArrayInputStream(screenshot), ".png");

            // Прикрепляем page source
            Allure.addAttachment("Page Source",
                    "text/html", driver.getPageSource(), ".html");

            // Прикрепляем URL страницы
            Allure.addAttachment("URL",
                    "text/plain", driver.getCurrentUrl(), ".txt");
        }
    }
}
```

---

## Скриншоты при падении (JUnit 5 TestWatcher)

### Полный рабочий пример

```java
import io.qameta.allure.Allure;
import org.junit.jupiter.api.extension.ExtensionContext;
import org.junit.jupiter.api.extension.TestWatcher;
import org.openqa.selenium.OutputType;
import org.openqa.selenium.TakesScreenshot;
import org.openqa.selenium.WebDriver;

import java.io.ByteArrayInputStream;

/**
 * JUnit 5 расширение для автоматического создания скриншотов при падении теста.
 * Регистрируется через @ExtendWith на базовом тестовом классе.
 */
public class AllureScreenshotExtension implements TestWatcher {

    @Override
    public void testFailed(ExtensionContext context, Throwable cause) {
        // Получаем экземпляр тестового класса
        Object testInstance = context.getRequiredTestInstance();

        if (testInstance instanceof BaseTest baseTest) {
            WebDriver driver = baseTest.getDriver();

            if (driver != null) {
                attachScreenshot(driver);
                attachPageSource(driver);
                attachBrowserUrl(driver);
                attachBrowserConsoleLogs(driver);
            }
        }
    }

    // Прикрепляем скриншот
    private void attachScreenshot(WebDriver driver) {
        try {
            byte[] screenshot = ((TakesScreenshot) driver)
                    .getScreenshotAs(OutputType.BYTES);
            Allure.addAttachment(
                    "Скриншот при падении",
                    "image/png",
                    new ByteArrayInputStream(screenshot),
                    ".png"
            );
        } catch (Exception e) {
            Allure.addAttachment(
                    "Ошибка скриншота",
                    "text/plain",
                    "Не удалось сделать скриншот: " + e.getMessage()
            );
        }
    }

    // Прикрепляем HTML-код страницы
    private void attachPageSource(WebDriver driver) {
        try {
            String pageSource = driver.getPageSource();
            Allure.addAttachment(
                    "Page Source",
                    "text/html",
                    pageSource
            );
        } catch (Exception e) {
            // Молча пропускаем, если не удалось получить page source
        }
    }

    // Прикрепляем текущий URL
    private void attachBrowserUrl(WebDriver driver) {
        try {
            Allure.addAttachment("URL", "text/plain", driver.getCurrentUrl());
        } catch (Exception e) {
            // Молча пропускаем
        }
    }

    // Прикрепляем логи консоли браузера
    private void attachBrowserConsoleLogs(WebDriver driver) {
        try {
            var logEntries = driver.manage().logs().get("browser");
            if (logEntries != null && !logEntries.getAll().isEmpty()) {
                StringBuilder sb = new StringBuilder();
                logEntries.getAll().forEach(entry ->
                        sb.append(String.format("[%s] %s%n",
                                entry.getLevel(), entry.getMessage())));
                Allure.addAttachment(
                        "Browser Console Logs",
                        "text/plain",
                        sb.toString()
                );
            }
        } catch (Exception e) {
            // Не все браузеры поддерживают получение логов
        }
    }
}
```

### Базовый тестовый класс

```java
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.extension.ExtendWith;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;

@ExtendWith(AllureScreenshotExtension.class)
public abstract class BaseTest {

    private WebDriver driver;

    @BeforeEach
    void setUp() {
        ChromeOptions options = new ChromeOptions();
        // Включаем логирование браузера для Allure
        options.setCapability("goog:loggingPrefs",
                Map.of("browser", "ALL", "performance", "ALL"));
        driver = new ChromeDriver(options);
        driver.manage().window().maximize();
    }

    @AfterEach
    void tearDown() {
        if (driver != null) {
            driver.quit();
        }
    }

    public WebDriver getDriver() {
        return driver;
    }
}
```

---

## Прикрепление вложений

### Прикрепление логов

```java
import io.qameta.allure.Allure;
import java.io.ByteArrayInputStream;
import java.nio.charset.StandardCharsets;

public class AllureLogger {

    // Накопитель логов для текущего теста (ThreadLocal для параллельного запуска)
    private static final ThreadLocal<StringBuilder> LOG_BUFFER =
            ThreadLocal.withInitial(StringBuilder::new);

    // Добавляем строку в лог
    public static void log(String message) {
        LOG_BUFFER.get()
                .append(String.format("[%s] %s%n",
                        java.time.LocalDateTime.now(), message));
    }

    // Прикрепляем накопленные логи к отчёту
    public static void attachLogs() {
        String logs = LOG_BUFFER.get().toString();
        if (!logs.isEmpty()) {
            Allure.addAttachment(
                    "Лог теста",
                    "text/plain",
                    new ByteArrayInputStream(
                            logs.getBytes(StandardCharsets.UTF_8)),
                    ".log"
            );
        }
        LOG_BUFFER.remove(); // Очищаем после прикрепления
    }
}
```

### Прикрепление видео

```java
import io.qameta.allure.Allure;
import java.io.FileInputStream;
import java.nio.file.Path;

public class VideoAttachment {

    /**
     * Прикрепление видео из Selenoid или другого сервиса записи.
     * Видео обычно доступно по URL после завершения сессии.
     */
    public static void attachVideo(String sessionId) {
        // URL видео из Selenoid
        String videoUrl = String.format(
                "http://selenoid-host:4444/video/%s.mp4", sessionId);

        // Вариант 1: Ссылка на видео
        Allure.addAttachment("Video URL", "text/plain", videoUrl);

        // Вариант 2: Скачиваем и прикрепляем файл
        try {
            Path videoFile = downloadVideo(videoUrl);
            Allure.addAttachment(
                    "Видео записи теста",
                    "video/mp4",
                    new FileInputStream(videoFile.toFile()),
                    ".mp4"
            );
        } catch (Exception e) {
            Allure.addAttachment(
                    "Ошибка видео",
                    "text/plain",
                    "Не удалось прикрепить видео: " + e.getMessage()
            );
        }
    }

    private static Path downloadVideo(String url) throws Exception {
        // Логика скачивания видео по HTTP
        var httpClient = java.net.http.HttpClient.newHttpClient();
        var request = java.net.http.HttpRequest.newBuilder()
                .uri(java.net.URI.create(url))
                .GET()
                .build();
        Path tempFile = Path.of("target", "videos", "test-video.mp4");
        httpClient.send(request,
                java.net.http.HttpResponse.BodyHandlers.ofFile(tempFile));
        return tempFile;
    }
}
```

### Прикрепление логов консоли браузера

```java
import io.qameta.allure.Allure;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.logging.LogEntry;
import org.openqa.selenium.logging.LogType;

import java.util.List;
import java.util.stream.Collectors;

public class BrowserConsoleAttachment {

    // Прикрепляем все логи консоли браузера
    public static void attachBrowserLogs(WebDriver driver) {
        try {
            List<LogEntry> logs = driver.manage().logs()
                    .get(LogType.BROWSER).getAll();

            if (!logs.isEmpty()) {
                String logContent = logs.stream()
                        .map(entry -> String.format("[%s][%s] %s",
                                new java.util.Date(entry.getTimestamp()),
                                entry.getLevel(),
                                entry.getMessage()))
                        .collect(Collectors.joining("\n"));

                Allure.addAttachment(
                        "Browser Console Logs",
                        "text/plain",
                        logContent
                );
            }
        } catch (Exception e) {
            // Некоторые браузеры не поддерживают API логов
        }
    }

    // Прикрепляем только ошибки консоли
    public static void attachBrowserErrors(WebDriver driver) {
        try {
            List<LogEntry> errors = driver.manage().logs()
                    .get(LogType.BROWSER).getAll().stream()
                    .filter(entry -> entry.getLevel().intValue() >= 900)
                    .toList();

            if (!errors.isEmpty()) {
                String errorContent = errors.stream()
                        .map(LogEntry::getMessage)
                        .collect(Collectors.joining("\n"));

                Allure.addAttachment(
                        "Browser Errors",
                        "text/plain",
                        errorContent
                );
            }
        } catch (Exception e) {
            // Тихо пропускаем
        }
    }
}
```

---

## Allure Serve vs Allure Generate

| Характеристика | `allure serve` | `allure generate` |
|----------------|---------------|-------------------|
| **Назначение** | Быстрый просмотр | Генерация статического отчёта |
| **Результат** | Открывает отчёт в браузере | Создаёт папку `allure-report/` |
| **Временные файлы** | Создаёт во временной директории | Создаёт в указанной директории |
| **Использование** | Локальная разработка | CI/CD, публикация |
| **Команда** | `allure serve allure-results` | `allure generate allure-results -o allure-report --clean` |
| **Просмотр** | Автоматически открывает браузер | Нужно открыть `allure-report/index.html` |

```bash
# Быстрый просмотр (локально)
allure serve target/allure-results

# Генерация статического отчёта (для CI/CD)
allure generate target/allure-results --clean -o target/allure-report

# Открыть сгенерированный отчёт
allure open target/allure-report
```

---

## Конфигурация Gradle

```groovy
plugins {
    id 'java'
    id 'io.qameta.allure' version '2.11.2'
}

allure {
    version = '2.25.0'
    adapter {
        // Автоматически добавляет allure-junit5 или allure-testng
        autoconfigure = true
        aspectjWeaver = true
    }
}

dependencies {
    testImplementation 'org.junit.jupiter:junit-jupiter:5.10.2'
    testImplementation 'io.qameta.allure:allure-junit5:2.25.0'
    testImplementation 'org.seleniumhq.selenium:selenium-java:4.17.0'
}

tasks.withType(Test) {
    useJUnitPlatform()
    systemProperty 'allure.results.directory', 'build/allure-results'
}
```

```bash
# Запуск тестов с Gradle
./gradlew clean test

# Генерация и просмотр отчёта
./gradlew allureServe

# Только генерация
./gradlew allureReport
```

---

## Полный рабочий пример

### Тест с полной интеграцией Allure

```java
import io.qameta.allure.*;
import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.ExtendWith;
import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;

import static org.junit.jupiter.api.Assertions.*;

@Epic("E-Commerce")
@Feature("Авторизация")
@ExtendWith(AllureScreenshotExtension.class)
public class LoginTests extends BaseTest {

    private LoginPage loginPage;

    @BeforeEach
    void initPages() {
        loginPage = new LoginPage(getDriver());
    }

    @Test
    @Story("Успешная авторизация")
    @Severity(SeverityLevel.BLOCKER)
    @Owner("QA Team")
    @Description("Проверяем, что пользователь с валидными данными "
               + "может войти в систему")
    @Issue("AUTH-100")
    @TmsLink("TC-001")
    void testSuccessfulLogin() {
        // Шаги отображаются в Allure благодаря @Step на методах LoginPage
        loginPage.open();
        loginPage.login("user@example.com", "password123");

        // Проверяем, что попали на главную страницу
        Allure.step("Проверить переход на главную страницу", () -> {
            String currentUrl = getDriver().getCurrentUrl();
            assertTrue(currentUrl.contains("/dashboard"),
                    "Ожидался переход на /dashboard, но текущий URL: " + currentUrl);
        });

        // Проверяем приветственное сообщение
        Allure.step("Проверить приветственное сообщение", () -> {
            String welcomeText = getDriver()
                    .findElement(By.id("welcome"))
                    .getText();
            assertEquals("Добро пожаловать, User!", welcomeText);
        });
    }

    @Test
    @Story("Неуспешная авторизация")
    @Severity(SeverityLevel.CRITICAL)
    @Description("Проверяем отображение ошибки при неверном пароле")
    void testLoginWithInvalidPassword() {
        loginPage.open();
        loginPage.login("user@example.com", "wrong_password");

        Allure.step("Проверить сообщение об ошибке", () -> {
            String errorMessage = loginPage.getErrorMessage();
            assertEquals("Неверный логин или пароль", errorMessage);
        });
    }

    @Test
    @Story("Валидация формы")
    @Severity(SeverityLevel.NORMAL)
    @Description("Проверяем валидацию при пустых полях формы логина")
    void testLoginWithEmptyFields() {
        loginPage.open();
        loginPage.clickLoginButton();

        Allure.step("Проверить валидационные сообщения", () -> {
            assertTrue(loginPage.isUsernameErrorDisplayed(),
                    "Ошибка валидации для поля логин не отображается");
            assertTrue(loginPage.isPasswordErrorDisplayed(),
                    "Ошибка валидации для поля пароль не отображается");
        });
    }
}
```

### Page Object с аннотациями @Step

```java
import io.qameta.allure.Step;
import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.support.ui.ExpectedConditions;
import org.openqa.selenium.support.ui.WebDriverWait;

import java.time.Duration;

public class LoginPage {

    private final WebDriver driver;
    private final WebDriverWait wait;

    // Локаторы
    private final By usernameField = By.id("username");
    private final By passwordField = By.id("password");
    private final By loginButton = By.id("login-btn");
    private final By errorMessage = By.cssSelector(".error-message");
    private final By usernameError = By.cssSelector("#username-error");
    private final By passwordError = By.cssSelector("#password-error");

    public LoginPage(WebDriver driver) {
        this.driver = driver;
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(10));
    }

    @Step("Открыть страницу авторизации")
    public void open() {
        driver.get("https://example.com/login");
        wait.until(ExpectedConditions.visibilityOfElementLocated(usernameField));
    }

    @Step("Ввести логин: {username}")
    public void enterUsername(String username) {
        driver.findElement(usernameField).clear();
        driver.findElement(usernameField).sendKeys(username);
    }

    @Step("Ввести пароль")
    public void enterPassword(String password) {
        // Пароль не выводим в отчёт
        driver.findElement(passwordField).clear();
        driver.findElement(passwordField).sendKeys(password);
    }

    @Step("Нажать кнопку 'Войти'")
    public void clickLoginButton() {
        driver.findElement(loginButton).click();
    }

    @Step("Авторизоваться как {username}")
    public void login(String username, String password) {
        enterUsername(username);
        enterPassword(password);
        clickLoginButton();
    }

    @Step("Получить текст сообщения об ошибке")
    public String getErrorMessage() {
        WebElement error = wait.until(
                ExpectedConditions.visibilityOfElementLocated(errorMessage));
        return error.getText();
    }

    public boolean isUsernameErrorDisplayed() {
        return !driver.findElements(usernameError).isEmpty()
                && driver.findElement(usernameError).isDisplayed();
    }

    public boolean isPasswordErrorDisplayed() {
        return !driver.findElements(passwordError).isEmpty()
                && driver.findElement(passwordError).isDisplayed();
    }
}
```

---

## Связь с тестированием

Интеграция Allure — это не просто «подключить библиотеку». Качественная интеграция
обеспечивает:

- **Быструю диагностику** — скриншоты, логи и page source при падении сокращают время
  анализа с часов до минут
- **Прозрачность для команды** — менеджеры и разработчики видят понятные отчёты
- **Автоматизацию отчётности** — отчёт генерируется на каждый запуск в CI/CD
- **Историю качества** — тренды показывают динамику стабильности тестов
- **Контекст окружения** — `environment.properties` фиксирует версии браузера, ОС, окружения

---

## Типичные ошибки

1. **Забыли AspectJ Weaver** — `@Step` аннотации на Page Object не работают, шаги не попадают в отчёт
2. **Не включили `autodetection.enabled`** — Allure listener не регистрируется автоматически в JUnit 5
3. **Скриншот после `driver.quit()`** — WebDriver уже закрыт, скриншот невозможно сделать
4. **Нет `ThreadLocal` для `WebDriver`** — при параллельном запуске скриншот делается от чужого браузера
5. **Слишком большие вложения** — видео или page source мегабайтного размера раздувают отчёт
6. **Не очищают `allure-results/`** — результаты прошлых запусков смешиваются с текущими
7. **`allure serve` в CI/CD** — `serve` запускает локальный сервер, в CI нужен `generate`
8. **Не прикрепляют контекст при падении** — без скриншота и логов диагностика затруднена
9. **Пароли в `@Step`** — параметр `password` отображается в отчёте в открытом виде
10. **Разные версии Allure** — несовместимость allure-junit5 и allure CLI по версиям

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Какие зависимости нужны для интеграции Allure с JUnit 5?
2. Чем отличается `allure serve` от `allure generate`? Когда что использовать?
3. Как прикрепить скриншот к Allure-отчёту?
4. Зачем нужен AspectJ Weaver при работе с Allure?
5. Как добавить информацию об окружении в Allure-отчёт?

### 🟡 Средний уровень
6. Как реализовать автоматическое создание скриншотов при падении теста в JUnit 5?
7. Что такое `TestWatcher` и как его использовать с Allure?
8. Как прикрепить к отчёту логи консоли браузера?
9. Как настроить Allure для работы с Gradle?
10. Как обеспечить корректную работу Allure при параллельном запуске тестов?

### 🔴 Продвинутый уровень
11. Как организовать прикрепление видеозаписи тестов (Selenoid) в Allure-отчёт?
12. Как реализовать кастомный Allure listener для сбора метрик производительности тестов?
13. Как настроить полный CI/CD pipeline с Allure: генерация отчёта, сохранение истории, публикация?
14. Как реализовать thread-safe логгер для Allure при параллельном запуске?
15. Как интегрировать Allure с REST Assured для автоматического логирования HTTP-запросов?

---

## Практические задания

### Задание 1: Базовая интеграция
Создайте Maven-проект с JUnit 5 и Allure. Настройте все зависимости, AspectJ Weaver
и Allure Maven Plugin. Убедитесь, что `allure serve` показывает отчёт с шагами.

### Задание 2: Скриншоты при падении
Реализуйте `TestWatcher` extension, который автоматически делает скриншот, сохраняет
page source и URL при падении теста. Намеренно уроните несколько тестов и проверьте вложения.

### Задание 3: Логирование
Реализуйте thread-safe логгер с `ThreadLocal<StringBuilder>`, который накапливает
лог-записи в ходе теста и прикрепляет их к Allure-отчёту по завершении теста.

### Задание 4: API-тесты с Allure
Добавьте `allure-rest-assured` фильтр к REST Assured тестам. Убедитесь, что HTTP-запросы
и ответы автоматически логируются в отчёте.

### Задание 5: Полный pipeline
Настройте GitHub Actions (или Jenkins) pipeline: запуск тестов, генерация Allure-отчёта,
сохранение `history/` между запусками, публикация отчёта на GitHub Pages.

---

## Дополнительные ресурсы

- [Allure JUnit 5 Integration Guide](https://docs.qameta.io/allure/#_junit_5)
- [Allure TestNG Integration Guide](https://docs.qameta.io/allure/#_testng)
- [Allure Maven Plugin](https://github.com/allure-framework/allure-maven)
- [Allure Gradle Plugin](https://github.com/allure-framework/allure-gradle)
- [Allure REST Assured Integration](https://docs.qameta.io/allure/#_rest_assured)
- [JUnit 5 Extensions Guide](https://junit.org/junit5/docs/current/user-guide/#extensions)
- [Selenide + Allure Integration](https://github.com/allure-framework/allure-java/tree/master/allure-selenide)
