# Этап 4: Репортинг с Allure

## Обзор

На предыдущих этапах мы написали API-тесты и UI-тесты с базовыми Allure-аннотациями. Теперь
пришло время превратить сырые результаты тестов в профессиональный отчёт, который можно
показать менеджеру, заказчику или на собеседовании. Allure Report предоставляет мощную
визуализацию: графики трендов, иерархию тестов, скриншоты при падении, логи запросов и
классификацию дефектов.

> **Цель:** Настроить полноценный Allure Report с иерархией Epic/Feature/Story, категоризацией
> дефектов, информацией об окружении, скриншотами, логами и историей прогонов.

---

## Предварительные требования

Перед началом убедитесь, что:
- API-тесты и UI-тесты из Этапов 2 и 3 проходят
- В `pom.xml` подключены зависимости `allure-junit5`, `allure-rest-assured`, `allure-selenide`
- Maven Surefire Plugin настроен с AspectJ Weaver
- Тесты генерируют результаты в `target/allure-results/`

---

## Часть 1: Иерархия тестов -- @Epic, @Feature, @Story

### 1.1 Концепция BDD-иерархии в Allure

Allure организует тесты в три уровня:

| Уровень | Аннотация | Назначение | Пример |
|---------|-----------|------------|--------|
| Epic | `@Epic` | Крупный модуль или продукт | "Conduit Blog Platform" |
| Feature | `@Feature` | Функциональная область | "Articles", "Authentication" |
| Story | `@Story` | Конкретный пользовательский сценарий | "Create Article", "Login" |

Эта иерархия отображается в Allure на вкладке **Behaviors** и позволяет быстро понять,
какая функциональность покрыта тестами и где есть проблемы.

### 1.2 Полная схема аннотаций для Conduit

```
@Epic("Conduit Blog Platform")
├── @Feature("Authentication")
│   ├── @Story("Register")
│   ├── @Story("Login")
│   └── @Story("Current User")
├── @Feature("Articles")
│   ├── @Story("Create Article")
│   ├── @Story("Get Article")
│   ├── @Story("Update Article")
│   ├── @Story("Delete Article")
│   └── @Story("List Articles")
├── @Feature("Comments")
│   ├── @Story("Add Comment")
│   ├── @Story("Get Comments")
│   └── @Story("Delete Comment")
├── @Feature("Favorites")
│   ├── @Story("Favorite Article")
│   └── @Story("Unfavorite Article")
├── @Feature("Profiles")
│   ├── @Story("Get Profile")
│   ├── @Story("Follow User")
│   └── @Story("Unfollow User")
├── @Feature("Tags")
│   └── @Story("Get Tags")
└── @Feature("User Settings")
    ├── @Story("Update Profile")
    └── @Story("Logout")
```

### 1.3 Применение аннотаций на практике

Аннотации ставятся на уровне класса и/или метода:

```java
package api.tests;

import io.qameta.allure.*;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

// Epic и Feature — на уровне класса (общие для всех тестов в классе)
@Epic("Conduit Blog Platform")
@Feature("Articles")
@DisplayName("API: Статьи — CRUD операции")
public class ArticleApiTest extends BaseApiTest {

    // Story и Severity — на уровне метода (конкретный сценарий)
    @Test
    @Story("Create Article")
    @Severity(SeverityLevel.BLOCKER)
    @DisplayName("Создание статьи со всеми полями")
    @Description("Проверяем, что авторизованный пользователь может создать статью " +
                 "с заголовком, описанием, телом и тегами. Ответ должен содержать " +
                 "все переданные поля и slug.")
    @Owner("qa-team")
    @Link(name = "API Spec", url = "https://realworld-docs.netlify.app/specifications/backend/endpoints/")
    @Issue("COND-123")
    @TmsLink("TC-ART-001")
    void shouldCreateArticle() {
        // Тело теста
    }

    @Test
    @Story("Delete Article")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("Удаление своей статьи")
    void shouldDeleteArticle() {
        // Тело теста
    }
}
```

### 1.4 Полный список аннотаций Allure

| Аннотация | Назначение | Где ставить |
|-----------|------------|-------------|
| `@Epic` | Крупный модуль | Класс |
| `@Feature` | Функциональная область | Класс |
| `@Story` | Конкретный сценарий | Метод |
| `@Severity` | Критичность теста (BLOCKER, CRITICAL, NORMAL, MINOR, TRIVIAL) | Метод |
| `@DisplayName` | Читаемое имя теста (из JUnit 5, Allure подхватывает) | Метод |
| `@Description` | Подробное описание теста | Метод |
| `@Owner` | Ответственный за тест | Класс или метод |
| `@Link` | Ссылка на документацию | Метод |
| `@Issue` | Ссылка на баг-трекер | Метод |
| `@TmsLink` | Ссылка на тест-кейс в TMS | Метод |
| `@Step` | Шаг внутри теста (для методов) | Вспомогательный метод |

---

## Часть 2: Шаги (@Step) -- детализация выполнения теста

### 2.1 Зачем нужны @Step

Без `@Step` Allure-отчёт показывает только название теста и результат (passed/failed).
С `@Step` отчёт показывает каждое действие: какой запрос отправлен, какие данные введены,
что проверено. Это критически важно для анализа падений.

### 2.2 @Step в API-клиентах

```java
package api.client;

import io.qameta.allure.Step;
import io.restassured.response.Response;

public class ArticleApi {

    /**
     * Параметры в фигурных скобках автоматически подставляются в отчёт.
     * {request.article.title} -> "My Article Title"
     */
    @Step("POST /articles — Создание статьи: {request.article.title}")
    public static Response createArticle(String token, ArticleRequest request) {
        return given()
                .spec(RequestSpecs.authenticated(token))
                .body(request)
                .when()
                .post("/articles");
    }
}
```

### 2.3 @Step в Page Objects

```java
package ui.pages;

import io.qameta.allure.Step;

public class LoginPage extends BasePage {

    /**
     * Шаги в Page Object создают читаемый лог в Allure:
     * 1. "Открытие страницы логина"
     * 2. "Ввод email: user@test.com"
     * 3. "Ввод пароля"
     * 4. "Нажатие кнопки Sign in"
     */
    @Step("Открытие страницы логина")
    public LoginPage openPage() {
        open("/#/login");
        return this;
    }

    @Step("Ввод email: {email}")
    public LoginPage typeEmail(String email) {
        emailInput.setValue(email);
        return this;
    }

    // Пароль не выводим в лог из соображений безопасности
    @Step("Ввод пароля")
    public LoginPage typePassword(String password) {
        passwordInput.setValue(password);
        return this;
    }
}
```

### 2.4 Программные шаги (без аннотаций)

Иногда `@Step` неудобен (например, в лямбдах). Используйте программный API:

```java
import io.qameta.allure.Allure;

@Test
void shouldCreateAndVerifyArticle() {
    String title = DataGenerator.randomTitle();

    Allure.step("Подготовка тестовых данных", () -> {
        // Подготовка
    });

    Allure.step("Создание статьи через API", () -> {
        // Создание
    });

    Allure.step("Проверка статьи в списке", () -> {
        // Проверка
    });
}
```

---

## Часть 3: Категоризация дефектов -- categories.json

### 3.1 Зачем нужна категоризация

Когда тест падает, важно понять: это баг в продукте, проблема в тесте или сбой инфраструктуры.
Allure позволяет классифицировать падения по категориям через файл `categories.json`.

### 3.2 Создание файла categories.json

Поместите в `src/test/resources/categories.json`:

```json
[
  {
    "name": "Product Defects",
    "description": "Дефекты продукта — баги в приложении Conduit",
    "descriptionHtml": "<b>Дефекты продукта</b> — реальные баги в приложении, требующие исправления разработчиками.",
    "matchedStatuses": ["failed"],
    "messageRegex": ".*expected.*but.*was.*|.*Expected status code.*|.*status code: 5\\d{2}.*"
  },
  {
    "name": "Test Defects",
    "description": "Дефекты тестов — ошибки в тестовом коде",
    "descriptionHtml": "<b>Дефекты тестов</b> — проблемы в тестовом коде: NullPointerException, ClassCastException и т.д.",
    "matchedStatuses": ["broken"],
    "messageRegex": ".*NullPointerException.*|.*ClassCastException.*|.*NoSuchElementException.*|.*StaleElementReferenceException.*"
  },
  {
    "name": "Infrastructure Issues",
    "description": "Проблемы инфраструктуры — недоступность сервисов, таймауты",
    "descriptionHtml": "<b>Проблемы инфраструктуры</b> — сервер недоступен, таймауты сети, проблемы Docker.",
    "matchedStatuses": ["broken", "failed"],
    "messageRegex": ".*Connection refused.*|.*timed out.*|.*TimeoutException.*|.*UnreachableBrowserException.*|.*ERR_CONNECTION_REFUSED.*"
  },
  {
    "name": "Known Issues",
    "description": "Известные проблемы — задокументированные дефекты, ожидающие исправления",
    "matchedStatuses": ["failed"],
    "messageRegex": ".*KNOWN_ISSUE.*"
  },
  {
    "name": "Flaky Tests",
    "description": "Нестабильные тесты — проходят не каждый раз",
    "descriptionHtml": "<b>Нестабильные тесты</b> — требуют стабилизации. Проверьте ожидания и тестовые данные.",
    "matchedStatuses": ["failed", "broken"],
    "traceRegex": ".*flaky.*|.*intermittent.*"
  }
]
```

### 3.3 Как Allure использует categories.json

Allure сопоставляет `messageRegex` и `traceRegex` с сообщением об ошибке и stack trace
упавшего теста. Если regex совпадает -- тест попадает в соответствующую категорию.
Результаты видны на вкладке **Categories** в отчёте.

### 3.4 Копирование categories.json в allure-results

Файл `categories.json` должен оказаться в папке `target/allure-results/` при запуске.
Настройте копирование в `pom.xml`:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-resources-plugin</artifactId>
    <version>3.3.1</version>
    <executions>
        <!-- Копируем categories.json в allure-results -->
        <execution>
            <id>copy-allure-categories</id>
            <phase>test-compile</phase>
            <goals>
                <goal>copy-resources</goal>
            </goals>
            <configuration>
                <outputDirectory>${project.build.directory}/allure-results</outputDirectory>
                <resources>
                    <resource>
                        <directory>src/test/resources</directory>
                        <includes>
                            <include>categories.json</include>
                        </includes>
                    </resource>
                </resources>
            </configuration>
        </execution>
    </executions>
</plugin>
```

---

## Часть 4: Информация об окружении -- environment.properties

### 4.1 Файл environment.properties

Allure отображает параметры окружения на главной странице отчёта. Это критически важно
для понимания контекста: на каком окружении запускались тесты, с каким браузером,
какая версия приложения.

Создайте файл `src/test/resources/environment.properties`:

```properties
Browser=Chrome 125
Browser.Version=125.0.6422.60
OS=macOS 14.5 / Ubuntu 22.04 (CI)
Java.Version=17.0.10
Base.URL=http://localhost:3000/api
App.URL=http://localhost:4100
App.Version=1.0.0
Test.Framework=JUnit 5.10.2
Build.Tool=Maven 3.9.6
Selenide.Version=7.2.3
REST.Assured.Version=5.4.0
Parallel.Threads=4
```

### 4.2 Динамическая генерация environment.properties

Статический файл быстро устаревает. Лучше генерировать его программно при каждом запуске:

```java
package utils;

import config.ConfigReader;
import io.qameta.allure.Step;

import java.io.FileOutputStream;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Properties;

/**
 * Генерирует environment.properties для Allure Report.
 * Вызывается один раз перед запуском тестов.
 */
public class AllureEnvironmentWriter {

    private static final String ALLURE_RESULTS_DIR = "target/allure-results";

    @Step("Генерация environment.properties для Allure")
    public static void writeEnvironment() {
        Properties props = new Properties();

        // Информация о системе
        props.setProperty("OS", System.getProperty("os.name") + " " +
                System.getProperty("os.version"));
        props.setProperty("Java.Version", System.getProperty("java.version"));

        // Информация о тестовом окружении
        props.setProperty("Base.URL", ConfigReader.getBaseUrl());
        String appUrl = ConfigReader.get("app.base.url");
        if (appUrl != null) {
            props.setProperty("App.URL", appUrl);
        }

        // Информация о браузере
        String browser = ConfigReader.get("browser");
        if (browser != null) {
            props.setProperty("Browser", browser);
        }
        String headless = ConfigReader.get("headless");
        props.setProperty("Headless.Mode", headless != null ? headless : "false");

        // Версии зависимостей (можно считать из pom.xml через Maven)
        props.setProperty("Test.Framework", "JUnit 5");
        props.setProperty("REST.Assured", "5.4.0");
        props.setProperty("Selenide", "7.2.3");
        props.setProperty("Allure", "2.25.0");

        // CI-информация (если запуск из GitHub Actions)
        String githubRun = System.getenv("GITHUB_RUN_NUMBER");
        if (githubRun != null) {
            props.setProperty("CI.Run.Number", githubRun);
            props.setProperty("CI.Branch", System.getenv("GITHUB_REF_NAME"));
            props.setProperty("CI.Commit", System.getenv("GITHUB_SHA"));
        }

        // Записываем в файл
        try {
            Path dir = Path.of(ALLURE_RESULTS_DIR);
            Files.createDirectories(dir);
            try (FileOutputStream out = new FileOutputStream(
                    dir.resolve("environment.properties").toFile())) {
                props.store(out, "Allure Environment Properties");
            }
        } catch (IOException e) {
            System.err.println("Не удалось записать environment.properties: " + e.getMessage());
        }
    }
}
```

### 4.3 Вызов генерации перед тестами

Используйте JUnit 5 Extension для автоматического вызова:

```java
package utils;

import org.junit.jupiter.api.extension.BeforeAllCallback;
import org.junit.jupiter.api.extension.ExtensionContext;

/**
 * JUnit 5 Extension: генерирует environment.properties перед первым тестом.
 * Используется через @ExtendWith(AllureEnvironmentExtension.class) на базовом классе.
 */
public class AllureEnvironmentExtension implements BeforeAllCallback {

    // Гарантируем однократный вызов даже при параллельном запуске
    private static boolean written = false;

    @Override
    public void beforeAll(ExtensionContext context) {
        if (!written) {
            synchronized (AllureEnvironmentExtension.class) {
                if (!written) {
                    AllureEnvironmentWriter.writeEnvironment();
                    written = true;
                }
            }
        }
    }
}
```

Подключите extension к базовому тестовому классу:

```java
import org.junit.jupiter.api.extension.ExtendWith;
import utils.AllureEnvironmentExtension;

@ExtendWith(AllureEnvironmentExtension.class)
public abstract class BaseApiTest {
    // ...
}

@ExtendWith(AllureEnvironmentExtension.class)
public abstract class BaseUiTest {
    // ...
}
```

---

## Часть 5: Скриншоты при падении через TestWatcher

### 5.1 Зачем нужен TestWatcher

AllureSelenide автоматически делает скриншоты для UI-тестов. Но для API-тестов или
для дополнительной информации (логи, JSON-ответы) нужен кастомный TestWatcher.

### 5.2 Реализация AllureTestWatcher

```java
package utils;

import com.codeborne.selenide.Selenide;
import com.codeborne.selenide.WebDriverRunner;
import io.qameta.allure.Allure;
import org.junit.jupiter.api.extension.ExtensionContext;
import org.junit.jupiter.api.extension.TestWatcher;
import org.openqa.selenium.OutputType;

import java.io.ByteArrayInputStream;
import java.util.Optional;

/**
 * JUnit 5 TestWatcher: автоматически прикрепляет информацию при падении теста.
 * Для UI-тестов: скриншот + page source.
 * Для всех тестов: лог сообщений, stack trace.
 */
public class AllureTestWatcher implements TestWatcher {

    @Override
    public void testFailed(ExtensionContext context, Throwable cause) {
        // Прикрепляем сообщение об ошибке
        Allure.addAttachment("Error message", "text/plain", cause.getMessage());

        // Прикрепляем stack trace
        String stackTrace = getStackTrace(cause);
        Allure.addAttachment("Stack trace", "text/plain", stackTrace);

        // Если WebDriver активен — делаем скриншот и сохраняем page source
        if (isWebDriverActive()) {
            attachScreenshot("Скриншот при падении");
            attachPageSource("HTML-код страницы при падении");
        }
    }

    @Override
    public void testAborted(ExtensionContext context, Throwable cause) {
        Allure.addAttachment("Aborted", "text/plain",
                "Тест был прерван: " + cause.getMessage());
    }

    @Override
    public void testDisabled(ExtensionContext context, Optional<String> reason) {
        reason.ifPresent(r ->
                Allure.addAttachment("Disabled reason", "text/plain", r));
    }

    @Override
    public void testSuccessful(ExtensionContext context) {
        // Ничего не делаем при успехе (можно логировать время выполнения)
    }

    // ========== Вспомогательные методы ==========

    private void attachScreenshot(String name) {
        try {
            byte[] screenshot = Selenide.screenshot(OutputType.BYTES);
            if (screenshot != null) {
                Allure.addAttachment(name, "image/png",
                        new ByteArrayInputStream(screenshot), ".png");
            }
        } catch (Exception e) {
            // Если скриншот не удалось сделать — не падаем
            Allure.addAttachment("Screenshot error", "text/plain",
                    "Не удалось сделать скриншот: " + e.getMessage());
        }
    }

    private void attachPageSource(String name) {
        try {
            String source = Selenide.webdriver().driver().source();
            if (source != null) {
                Allure.addAttachment(name, "text/html", source, ".html");
            }
        } catch (Exception e) {
            // Игнорируем ошибку получения page source
        }
    }

    private boolean isWebDriverActive() {
        try {
            return WebDriverRunner.hasWebDriverStarted();
        } catch (Exception e) {
            return false;
        }
    }

    private String getStackTrace(Throwable cause) {
        StringBuilder sb = new StringBuilder();
        sb.append(cause.toString()).append("\n");
        for (StackTraceElement element : cause.getStackTrace()) {
            sb.append("\tat ").append(element).append("\n");
        }
        if (cause.getCause() != null) {
            sb.append("Caused by: ").append(cause.getCause().toString()).append("\n");
        }
        return sb.toString();
    }
}
```

### 5.3 Подключение TestWatcher к тестам

```java
import org.junit.jupiter.api.extension.ExtendWith;
import utils.AllureTestWatcher;
import utils.AllureEnvironmentExtension;

@ExtendWith({AllureEnvironmentExtension.class, AllureTestWatcher.class})
@Tag("ui")
public abstract class BaseUiTest {
    // ...
}

@ExtendWith({AllureEnvironmentExtension.class, AllureTestWatcher.class})
@Tag("api")
public abstract class BaseApiTest {
    // ...
}
```

---

## Часть 6: Прикрепление логов и дополнительных данных

### 6.1 Прикрепление API response body

Для API-тестов полезно прикрепить тело ответа (помимо AllureRestAssured):

```java
package utils;

import io.qameta.allure.Allure;
import io.restassured.response.Response;

/**
 * Утилиты для Allure: прикрепление различных артефактов к отчёту.
 */
public class AllureUtils {

    /**
     * Прикрепляет JSON-ответ API к текущему шагу Allure.
     */
    public static void attachResponse(String name, Response response) {
        String body = response.getBody().asPrettyString();
        Allure.addAttachment(name,
                "application/json",
                body,
                ".json");
    }

    /**
     * Прикрепляет произвольный текст к отчёту.
     */
    public static void attachText(String name, String content) {
        Allure.addAttachment(name, "text/plain", content);
    }

    /**
     * Прикрепляет JSON-строку к отчёту.
     */
    public static void attachJson(String name, String json) {
        Allure.addAttachment(name, "application/json", json, ".json");
    }

    /**
     * Прикрепляет CSV-данные к отчёту.
     */
    public static void attachCsv(String name, String csv) {
        Allure.addAttachment(name, "text/csv", csv, ".csv");
    }

    /**
     * Прикрепляет скриншот из массива байтов.
     */
    public static void attachScreenshot(String name, byte[] screenshot) {
        Allure.addAttachment(name, "image/png",
                new java.io.ByteArrayInputStream(screenshot), ".png");
    }
}
```

### 6.2 Использование в тестах

```java
@Test
@Story("Create Article")
@DisplayName("Создание статьи — прикрепляем полный ответ API")
void shouldCreateArticleWithResponseAttachment() {
    ArticleRequest request = new ArticleRequest(
            DataGenerator.randomTitle(), DataGenerator.randomDescription(),
            DataGenerator.randomBody(), List.of("java", "testing"));

    Response response = ArticleApi.createArticle(token, request);

    // Прикрепляем ответ для анализа в отчёте
    AllureUtils.attachResponse("API Response: Create Article", response);

    response.then()
            .statusCode(200)
            .body("article.title", notNullValue());
}
```

### 6.3 Прикрепление логов браузера (для UI-тестов)

```java
package utils;

import com.codeborne.selenide.WebDriverRunner;
import io.qameta.allure.Allure;
import org.openqa.selenium.logging.LogEntries;
import org.openqa.selenium.logging.LogEntry;
import org.openqa.selenium.logging.LogType;

import java.util.stream.Collectors;

/**
 * Прикрепляет логи браузерной консоли к Allure-отчёту.
 * Полезно для отладки JavaScript-ошибок.
 */
public class BrowserLogAttacher {

    public static void attachBrowserLogs() {
        try {
            LogEntries logs = WebDriverRunner.getWebDriver()
                    .manage()
                    .logs()
                    .get(LogType.BROWSER);

            if (logs != null && !logs.getAll().isEmpty()) {
                String logText = logs.getAll().stream()
                        .map(LogEntry::toString)
                        .collect(Collectors.joining("\n"));

                Allure.addAttachment("Browser Console Logs", "text/plain", logText);
            }
        } catch (Exception e) {
            // Не все браузеры и драйверы поддерживают логи
        }
    }
}
```

---

## Часть 7: Настройка allure.properties

### 7.1 Файл allure.properties

Создайте `src/test/resources/allure.properties`:

```properties
# Директория для результатов Allure
allure.results.directory=target/allure-results

# Ссылки на баг-трекер и TMS (Test Management System)
# Allure подставит ID из @Issue и @TmsLink в эти шаблоны
allure.link.issue.pattern=https://github.com/your-org/conduit/issues/{}
allure.link.tms.pattern=https://your-tms.com/testcase/{}
allure.link.custom.pattern=https://your-docs.com/{}
```

### 7.2 Настройка maven-surefire-plugin

Полная конфигурация плагина с AspectJ Weaver (необходим для работы `@Step`):

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>3.2.5</version>
            <configuration>
                <!-- AspectJ Weaver: необходим для @Step аннотаций Allure -->
                <argLine>
                    -javaagent:"${settings.localRepository}/org/aspectj/aspectjweaver/${aspectj.version}/aspectjweaver-${aspectj.version}.jar"
                </argLine>

                <!-- JUnit 5 Tag для фильтрации тестов -->
                <groups>${groups}</groups>

                <!-- Системные свойства для тестов -->
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

        <!-- Allure Maven Plugin: генерация отчёта -->
        <plugin>
            <groupId>io.qameta.allure</groupId>
            <artifactId>allure-maven</artifactId>
            <version>2.12.0</version>
            <configuration>
                <reportVersion>2.25.0</reportVersion>
                <resultsDirectory>${project.build.directory}/allure-results</resultsDirectory>
            </configuration>
        </plugin>
    </plugins>
</build>
```

---

## Часть 8: История и тренды

### 8.1 Как работает Allure History

Allure умеет показывать историю прогонов: графики trend, статус каждого теста в динамике,
изменение времени выполнения. Для этого нужна папка `history/` из предыдущего отчёта.

Принцип работы:
1. Запускаем тесты -- результаты в `target/allure-results/`
2. Копируем папку `history/` из предыдущего отчёта в `target/allure-results/history/`
3. Генерируем отчёт -- Allure подхватывает историю
4. В новом отчёте появляются графики трендов

### 8.2 Локальная настройка истории

Скрипт для сохранения и восстановления истории:

```bash
#!/bin/bash
# run-tests-with-history.sh
# Скрипт для запуска тестов с сохранением истории Allure

ALLURE_RESULTS="target/allure-results"
ALLURE_REPORT="target/allure-report"
HISTORY_DIR="allure-history"

# Шаг 1: Очищаем предыдущие результаты
rm -rf "$ALLURE_RESULTS"
mkdir -p "$ALLURE_RESULTS"

# Шаг 2: Копируем историю из предыдущего отчёта (если есть)
if [ -d "$ALLURE_REPORT/history" ]; then
    echo "Копируем историю из предыдущего отчёта..."
    cp -r "$ALLURE_REPORT/history" "$ALLURE_RESULTS/history"
fi

# Шаг 3: Запускаем тесты
echo "Запускаем тесты..."
mvn clean test "$@"

# Шаг 4: Генерируем отчёт
echo "Генерируем Allure-отчёт..."
mvn allure:report

# Шаг 5: Открываем отчёт в браузере
echo "Открываем отчёт..."
mvn allure:serve
```

### 8.3 История в CI/CD

В GitHub Actions история сохраняется на ветке `gh-pages` (подробно описано в 06_stage_ci_cd.md).
Ключевые шаги:
1. Скачать ветку `gh-pages`
2. Скопировать `history/` в `allure-results/`
3. Сгенерировать отчёт
4. Опубликовать обратно на `gh-pages`

---

## Часть 9: Пошаговый процесс -- от настройки до анализа

### Шаг 1: Подготовка конфигурации

1. Создайте `src/test/resources/allure.properties` с путями к результатам
2. Создайте `src/test/resources/categories.json` с 5 категориями
3. Реализуйте `AllureEnvironmentWriter` для динамической генерации environment
4. Настройте `maven-surefire-plugin` с AspectJ Weaver в `pom.xml`
5. Добавьте копирование `categories.json` через `maven-resources-plugin`

### Шаг 2: Аннотирование тестов

1. Добавьте `@Epic("Conduit Blog Platform")` на все тестовые классы
2. Добавьте `@Feature` на каждый тестовый класс (по модулю)
3. Добавьте `@Story` на каждый тестовый метод
4. Добавьте `@Severity` на каждый тестовый метод
5. Убедитесь, что `@DisplayName` задан для всех тестов
6. Добавьте `@Description` для сложных тестов
7. Добавьте `@Step` ко всем методам API-клиентов и Page Objects

### Шаг 3: Подключение Extensions

1. Реализуйте `AllureEnvironmentExtension`
2. Реализуйте `AllureTestWatcher`
3. Подключите оба extension к `BaseApiTest` и `BaseUiTest` через `@ExtendWith`

### Шаг 4: Запуск тестов и генерация отчёта

```bash
# Запуск всех тестов
mvn clean test

# Генерация и открытие отчёта в браузере
mvn allure:serve

# Или генерация статического отчёта (без веб-сервера)
mvn allure:report
# Отчёт в target/site/allure-maven-plugin/index.html
```

### Шаг 5: Анализ отчёта

На что обращать внимание в Allure Report:

| Вкладка | Что проверить |
|---------|---------------|
| **Overview** | Общая статистика: passed/failed/broken/skipped, environment info |
| **Suites** | Тесты сгруппированы по классам, видны шаги и вложения |
| **Behaviors** | Иерархия Epic -> Feature -> Story, покрытие функциональности |
| **Categories** | Классификация падений: Product Defects, Test Defects, Infrastructure |
| **Graphs** | Графики: Status, Severity, Duration, Trend (при наличии истории) |
| **Timeline** | Хронология запуска: параллелизм, длительность каждого теста |
| **Packages** | Тесты сгруппированы по Java-пакетам |

---

## Часть 10: Пример аннотированного теста (полная версия)

Ниже показан тест со всеми Allure-аннотациями и прикреплениями:

```java
package api.tests;

import api.client.ArticleApi;
import api.client.TokenManager;
import api.models.request.ArticleRequest;
import api.models.response.ArticleResponse;
import io.qameta.allure.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import io.restassured.response.Response;
import utils.AllureUtils;
import utils.DataGenerator;

import java.util.List;

import static io.restassured.module.jsv.JsonSchemaValidator.matchesJsonSchemaInClasspath;
import static org.assertj.core.api.Assertions.assertThat;

@Epic("Conduit Blog Platform")
@Feature("Articles")
@Owner("qa-team")
@DisplayName("API: Статьи — полный CRUD с Allure-аннотациями")
public class ArticleApiFullTest extends BaseApiTest {

    private String token;

    @BeforeEach
    void setUp() {
        token = TokenManager.getToken();
    }

    @Test
    @Story("Create Article")
    @Severity(SeverityLevel.BLOCKER)
    @DisplayName("Создание статьи со всеми полями и валидацией JSON Schema")
    @Description("Тест проверяет создание статьи с title, description, body и tagList. "
            + "Валидирует структуру ответа через JSON Schema, проверяет соответствие "
            + "полей отправленным данным и наличие slug и author.")
    @Link(name = "API Docs: Create Article",
          url = "https://realworld-docs.netlify.app/specifications/backend/endpoints/#create-article")
    @TmsLink("TC-ART-001")
    void shouldCreateArticleWithAllFields() {
        // Подготовка данных
        String title = DataGenerator.randomTitle();
        String description = DataGenerator.randomDescription();
        String body = DataGenerator.randomBody();
        List<String> tags = List.of("java", "automation");

        ArticleRequest request = new ArticleRequest(title, description, body, tags);

        // Выполнение запроса
        Response rawResponse = ArticleApi.createArticle(token, request);

        // Прикрепляем ответ к Allure-отчёту
        AllureUtils.attachResponse("Response: POST /articles", rawResponse);

        // Валидация JSON Schema
        rawResponse.then()
                .statusCode(200)
                .body(matchesJsonSchemaInClasspath("json-schemas/article-response.json"));

        // Десериализация и проверка полей
        ArticleResponse response = rawResponse.as(ArticleResponse.class);

        Allure.step("Проверка заголовка статьи", () ->
                assertThat(response.getArticle().getTitle()).isEqualTo(title));

        Allure.step("Проверка описания статьи", () ->
                assertThat(response.getArticle().getDescription()).isEqualTo(description));

        Allure.step("Проверка тела статьи", () ->
                assertThat(response.getArticle().getBody()).isEqualTo(body));

        Allure.step("Проверка тегов", () ->
                assertThat(response.getArticle().getTagList())
                        .containsExactlyInAnyOrder("java", "automation"));

        Allure.step("Проверка slug не пуст", () ->
                assertThat(response.getArticle().getSlug()).isNotBlank());

        Allure.step("Проверка автора статьи", () ->
                assertThat(response.getArticle().getAuthor().getUsername())
                        .isEqualTo(TokenManager.getUsername()));
    }
}
```

---

## Часть 11: Частые ошибки при настройке Allure

### Ошибка 1: @Step не отображаются в отчёте

**Причина:** отсутствует AspectJ Weaver в `maven-surefire-plugin`.

**Решение:** добавьте `-javaagent` в `<argLine>`:

```xml
<argLine>
    -javaagent:"${settings.localRepository}/org/aspectj/aspectjweaver/${aspectj.version}/aspectjweaver-${aspectj.version}.jar"
</argLine>
```

### Ошибка 2: environment.properties не отображается

**Причина:** файл не попал в `target/allure-results/`.

**Решение:** используйте `AllureEnvironmentWriter` или вручную скопируйте файл перед
генерацией отчёта.

### Ошибка 3: categories.json не работает

**Причина:** файл должен находиться именно в `target/allure-results/`, а не только
в `src/test/resources/`.

**Решение:** настройте `maven-resources-plugin` для копирования (см. Часть 3.4).

### Ошибка 4: История не отображается

**Причина:** папка `history/` из предыдущего отчёта не скопирована в `allure-results/`.

**Решение:** используйте скрипт из Части 8.2 или настройте CI/CD pipeline (06_stage_ci_cd.md).

### Ошибка 5: Скриншоты не прикрепляются к UI-тестам

**Причина:** AllureSelenide listener не добавлен.

**Решение:** добавьте в `@BeforeAll`:

```java
SelenideLogger.addListener("AllureSelenide",
        new AllureSelenide().screenshots(true).savePageSource(true));
```

---

## Практическое задание

### Задание 1: Настройка конфигурации Allure

1. Создайте `allure.properties` с путями и ссылками на баг-трекер
2. Создайте `categories.json` с 5 категориями дефектов
3. Настройте `maven-resources-plugin` для копирования `categories.json`
4. Настройте `maven-surefire-plugin` с AspectJ Weaver
5. Запустите `mvn clean test` и убедитесь, что `target/allure-results/` содержит данные

### Задание 2: Аннотирование тестов

1. Добавьте `@Epic`, `@Feature`, `@Story` ко всем тестам по схеме из Части 1.2
2. Добавьте `@Severity` ко всем тестовым методам
3. Добавьте `@DisplayName` ко всем тестовым методам
4. Добавьте `@Description` к 5 наиболее сложным тестам
5. Запустите `mvn allure:serve` и проверьте вкладку Behaviors

### Задание 3: Environment и TestWatcher

1. Реализуйте `AllureEnvironmentWriter` с динамической генерацией
2. Реализуйте `AllureEnvironmentExtension` и подключите к базовым классам
3. Реализуйте `AllureTestWatcher` со скриншотами при падении
4. Специально "сломайте" один тест и проверьте, что скриншот прикрепился
5. Проверьте, что environment отображается на вкладке Overview

### Задание 4: Категории и история

1. Запустите тесты минимум 3 раза с сохранением истории
2. Проверьте, что графики трендов отображаются (вкладка Graphs -> Trend)
3. Специально создайте тест, который падает с `Connection refused`
4. Проверьте, что тест попал в категорию "Infrastructure Issues"
5. Проверьте категорию "Product Defects" для assertion failure

### Задание 5: Полировка отчёта

1. Добавьте `@Owner("ваше_имя")` ко всем тестовым классам
2. Добавьте `@Link` с ссылкой на API-документацию к API-тестам
3. Добавьте `@TmsLink` к 10 ключевым тестам
4. Используйте `AllureUtils.attachResponse()` в 5 API-тестах
5. Сделайте финальный прогон и проверьте каждую вкладку Allure

---

## Чек-лист самопроверки

- [ ] `allure.properties` настроен с путями и шаблонами ссылок
- [ ] `categories.json` содержит 5 категорий: Product Defects, Test Defects, Infrastructure, Known Issues, Flaky Tests
- [ ] `categories.json` автоматически копируется в `target/allure-results/`
- [ ] `environment.properties` генерируется динамически (OS, Java, URL, браузер, CI)
- [ ] `maven-surefire-plugin` настроен с AspectJ Weaver
- [ ] Все тестовые классы имеют `@Epic` и `@Feature`
- [ ] Все тестовые методы имеют `@Story`, `@Severity` и `@DisplayName`
- [ ] API-клиенты и Page Objects используют `@Step` с параметрами
- [ ] `AllureTestWatcher` прикрепляет скриншоты и stack trace при падении
- [ ] `AllureEnvironmentExtension` генерирует environment перед тестами
- [ ] AllureSelenide listener настроен с `screenshots(true)` и `savePageSource(true)`
- [ ] На вкладке Behaviors видна иерархия Epic -> Feature -> Story
- [ ] На вкладке Categories корректно классифицируются падения
- [ ] На вкладке Overview отображается информация об окружении
- [ ] Графики трендов появляются после 2+ прогонов
- [ ] Отчёт выглядит профессионально и информативно
- [ ] Готов перейти к этапу CI/CD
