# Фреймворк с нуля — пошаговый гайд

## Обзор

Этот раздел — практическое руководство по созданию тестового фреймворка с нуля. Мы пройдём все этапы:
от создания Maven-проекта до генерации Allure-отчёта. В результате у вас будет рабочий фреймворк
с Selenide (UI), REST Assured (API), JUnit 5, Allure, Lombok и Owner — стандартный стек
для Java QA Automation в 2024-2026 годах. Каждый шаг содержит код, который можно копировать и запускать.

---

## Шаг 1: Создание Maven-проекта

### Структура проекта

```
test-framework/
├── pom.xml
├── .gitignore
├── src/
│   └── test/
│       ├── java/
│       │   └── com/
│       │       └── example/
│       │           └── framework/
│       │               ├── config/
│       │               ├── pages/
│       │               ├── api/
│       │               ├── tests/
│       │               │   ├── ui/
│       │               │   └── api/
│       │               ├── models/
│       │               ├── data/
│       │               ├── base/
│       │               └── utils/
│       └── resources/
│           ├── config/
│           │   ├── application.properties
│           │   ├── dev.properties
│           │   └── staging.properties
│           ├── junit-platform.properties
│           └── allure.properties
```

### .gitignore

```gitignore
# IDE
.idea/
*.iml
.vscode/
.project
.classpath

# Build
target/
build/
out/

# Allure
allure-results/
allure-report/

# OS
.DS_Store
Thumbs.db

# Секреты
.env
**/secrets.properties

# Логи
*.log
```

---

## Шаг 2: Настройка pom.xml

Полный `pom.xml` с версиями, зависимостями и плагинами:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>test-framework</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <name>Test Automation Framework</name>
    <description>UI + API тестовый фреймворк на Java 17</description>

    <properties>
        <!-- Java -->
        <java.version>17</java.version>
        <maven.compiler.source>${java.version}</maven.compiler.source>
        <maven.compiler.target>${java.version}</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>

        <!-- Версии зависимостей -->
        <selenide.version>7.2.3</selenide.version>
        <rest-assured.version>5.4.0</rest-assured.version>
        <junit.version>5.10.2</junit.version>
        <allure.version>2.27.0</allure.version>
        <allure-maven.version>2.12.0</allure-maven.version>
        <aspectj.version>1.9.22</aspectj.version>
        <lombok.version>1.18.32</lombok.version>
        <owner.version>1.0.12</owner.version>
        <slf4j.version>2.0.13</slf4j.version>
        <logback.version>1.5.6</logback.version>
        <jackson.version>2.17.1</jackson.version>
        <javafaker.version>1.0.2</javafaker.version>
        <surefire.version>3.2.5</surefire.version>

        <!-- Параметры запуска -->
        <env>dev</env>
        <threads>4</threads>
    </properties>

    <dependencies>
        <!-- ========== Тестовый фреймворк ========== -->

        <!-- JUnit 5 -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>${junit.version}</version>
            <scope>test</scope>
        </dependency>

        <!-- ========== UI-тестирование ========== -->

        <!-- Selenide — обёртка над Selenium с удобным API -->
        <dependency>
            <groupId>com.codeborne</groupId>
            <artifactId>selenide</artifactId>
            <version>${selenide.version}</version>
            <scope>test</scope>
        </dependency>

        <!-- ========== API-тестирование ========== -->

        <!-- REST Assured — тестирование REST API -->
        <dependency>
            <groupId>io.rest-assured</groupId>
            <artifactId>rest-assured</artifactId>
            <version>${rest-assured.version}</version>
            <scope>test</scope>
        </dependency>

        <!-- Jackson — сериализация/десериализация JSON -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>${jackson.version}</version>
            <scope>test</scope>
        </dependency>

        <!-- ========== Отчётность ========== -->

        <!-- Allure JUnit 5 интеграция -->
        <dependency>
            <groupId>io.qameta.allure</groupId>
            <artifactId>allure-junit5</artifactId>
            <version>${allure.version}</version>
            <scope>test</scope>
        </dependency>

        <!-- Allure Selenide — автоматические скриншоты и шаги -->
        <dependency>
            <groupId>io.qameta.allure</groupId>
            <artifactId>allure-selenide</artifactId>
            <version>${allure.version}</version>
            <scope>test</scope>
        </dependency>

        <!-- Allure REST Assured — логирование HTTP запросов/ответов -->
        <dependency>
            <groupId>io.qameta.allure</groupId>
            <artifactId>allure-rest-assured</artifactId>
            <version>${allure.version}</version>
            <scope>test</scope>
        </dependency>

        <!-- ========== Утилиты ========== -->

        <!-- Lombok — генерация boilerplate-кода -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lombok.version}</version>
            <scope>provided</scope>
        </dependency>

        <!-- Owner — управление конфигурацией -->
        <dependency>
            <groupId>org.aeonbits.owner</groupId>
            <artifactId>owner</artifactId>
            <version>${owner.version}</version>
            <scope>test</scope>
        </dependency>

        <!-- JavaFaker — генерация тестовых данных -->
        <dependency>
            <groupId>com.github.javafaker</groupId>
            <artifactId>javafaker</artifactId>
            <version>${javafaker.version}</version>
            <scope>test</scope>
        </dependency>

        <!-- ========== Логирование ========== -->

        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>${slf4j.version}</version>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>${logback.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- Компиляция -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.13.0</version>
                <configuration>
                    <source>${java.version}</source>
                    <target>${java.version}</target>
                    <annotationProcessorPaths>
                        <path>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                            <version>${lombok.version}</version>
                        </path>
                    </annotationProcessorPaths>
                </configuration>
            </plugin>

            <!-- Запуск тестов -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>${surefire.version}</version>
                <configuration>
                    <argLine>
                        -javaagent:"${settings.localRepository}/org/aspectj/aspectjweaver/${aspectj.version}/aspectjweaver-${aspectj.version}.jar"
                    </argLine>
                    <systemPropertyVariables>
                        <env>${env}</env>
                    </systemPropertyVariables>
                    <testFailureIgnore>false</testFailureIgnore>
                </configuration>
                <dependencies>
                    <dependency>
                        <groupId>org.aspectj</groupId>
                        <artifactId>aspectjweaver</artifactId>
                        <version>${aspectj.version}</version>
                    </dependency>
                </dependencies>
            </plugin>

            <!-- Allure отчёт -->
            <plugin>
                <groupId>io.qameta.allure</groupId>
                <artifactId>allure-maven</artifactId>
                <version>${allure-maven.version}</version>
                <configuration>
                    <reportVersion>${allure.version}</reportVersion>
                    <resultsDirectory>allure-results</resultsDirectory>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## Шаг 3: Создание конфигурации

### config/application.properties

```properties
# Браузер
browser=chrome
browser.size=1920x1080
browser.headless=false

# Таймауты (мс)
timeout=10000
page.load.timeout=30000

# Отчётность
screenshots.on.failure=true
```

### config/dev.properties

```properties
# Dev-окружение
base.url=https://demoqa.com
api.base.url=https://demoqa.com
```

### config/staging.properties

```properties
# Staging-окружение
base.url=https://staging.demoqa.com
api.base.url=https://staging.demoqa.com
```

### ProjectConfig.java

```java
package com.example.framework.config;

import org.aeonbits.owner.Config;
import org.aeonbits.owner.Config.LoadPolicy;
import org.aeonbits.owner.Config.LoadType;
import org.aeonbits.owner.Config.Sources;

@LoadPolicy(LoadType.MERGE)
@Sources({
    "system:properties",
    "system:env",
    "classpath:config/${env}.properties",
    "classpath:config/application.properties"
})
public interface ProjectConfig extends Config {

    @Key("base.url")
    @DefaultValue("https://demoqa.com")
    String baseUrl();

    @Key("api.base.url")
    @DefaultValue("https://demoqa.com")
    String apiBaseUrl();

    @Key("browser")
    @DefaultValue("chrome")
    String browser();

    @Key("browser.size")
    @DefaultValue("1920x1080")
    String browserSize();

    @Key("browser.headless")
    @DefaultValue("false")
    boolean headless();

    @Key("timeout")
    @DefaultValue("10000")
    int timeout();

    @Key("env")
    @DefaultValue("dev")
    String env();
}
```

### ConfigProvider.java

```java
package com.example.framework.config;

import org.aeonbits.owner.ConfigFactory;

/**
 * Единая точка доступа к конфигурации проекта (Singleton).
 * Использование: ConfigProvider.getConfig().baseUrl()
 */
public class ConfigProvider {

    private static volatile ProjectConfig config;

    public static ProjectConfig getConfig() {
        if (config == null) {
            synchronized (ConfigProvider.class) {
                if (config == null) {
                    config = ConfigFactory.create(
                        ProjectConfig.class,
                        System.getProperties(),
                        System.getenv()
                    );
                }
            }
        }
        return config;
    }

    private ConfigProvider() {
    }
}
```

---

## Шаг 4: Создание BasePage

```java
package com.example.framework.base;

import com.codeborne.selenide.Condition;
import com.codeborne.selenide.SelenideElement;

import static com.codeborne.selenide.Selenide.$;

/**
 * Базовый класс для всех Page Objects.
 * Содержит общие методы, используемые на всех страницах.
 */
public abstract class BasePage {

    // Общий элемент загрузки (spinner/loader)
    private final SelenideElement loader = $(".loading-spinner");

    /**
     * Ожидание исчезновения индикатора загрузки.
     * Вызывается после навигации или действий, вызывающих загрузку.
     */
    protected void waitForLoaderDisappear() {
        loader.shouldNotBe(Condition.visible);
    }

    /**
     * Проверка, что страница загружена.
     * Каждый наследник определяет свой критерий загрузки.
     */
    public abstract BasePage waitForPageLoaded();
}
```

---

## Шаг 5: Создание первого Page Object

Используем сайт [demoqa.com](https://demoqa.com) как пример:

### TextBoxPage.java

```java
package com.example.framework.pages;

import com.codeborne.selenide.Condition;
import com.codeborne.selenide.SelenideElement;
import com.example.framework.base.BasePage;
import io.qameta.allure.Step;

import static com.codeborne.selenide.Selenide.$;
import static com.codeborne.selenide.Selenide.open;

/**
 * Page Object для страницы Text Box (https://demoqa.com/text-box).
 * Содержит форму ввода пользовательских данных.
 */
public class TextBoxPage extends BasePage {

    // Локаторы полей ввода
    private final SelenideElement fullNameInput = $("#userName");
    private final SelenideElement emailInput = $("#userEmail");
    private final SelenideElement currentAddressInput = $("#currentAddress");
    private final SelenideElement permanentAddressInput = $("#permanentAddress");
    private final SelenideElement submitButton = $("#submit");

    // Локаторы результата
    private final SelenideElement outputSection = $("#output");
    private final SelenideElement outputName = $("#name");
    private final SelenideElement outputEmail = $("#email");

    @Step("Открыть страницу Text Box")
    public TextBoxPage openPage() {
        open("/text-box");
        return this;
    }

    @Override
    public TextBoxPage waitForPageLoaded() {
        fullNameInput.shouldBe(Condition.visible);
        return this;
    }

    @Step("Ввести полное имя: '{name}'")
    public TextBoxPage enterFullName(String name) {
        fullNameInput.setValue(name);
        return this;
    }

    @Step("Ввести email: '{email}'")
    public TextBoxPage enterEmail(String email) {
        emailInput.setValue(email);
        return this;
    }

    @Step("Ввести текущий адрес: '{address}'")
    public TextBoxPage enterCurrentAddress(String address) {
        currentAddressInput.setValue(address);
        return this;
    }

    @Step("Ввести постоянный адрес: '{address}'")
    public TextBoxPage enterPermanentAddress(String address) {
        permanentAddressInput.setValue(address);
        return this;
    }

    @Step("Нажать кнопку Submit")
    public TextBoxPage clickSubmit() {
        submitButton.scrollTo().click();
        return this;
    }

    @Step("Проверить, что секция результата отображается")
    public TextBoxPage verifyOutputVisible() {
        outputSection.shouldBe(Condition.visible);
        return this;
    }

    @Step("Проверить, что имя в результате содержит '{expectedName}'")
    public TextBoxPage verifyOutputName(String expectedName) {
        outputName.shouldHave(Condition.text(expectedName));
        return this;
    }

    @Step("Проверить, что email в результате содержит '{expectedEmail}'")
    public TextBoxPage verifyOutputEmail(String expectedEmail) {
        outputEmail.shouldHave(Condition.text(expectedEmail));
        return this;
    }
}
```

---

## Шаг 6: Создание базовых тестовых классов и первого теста

### BaseUiTest.java

```java
package com.example.framework.base;

import com.codeborne.selenide.Configuration;
import com.codeborne.selenide.logevents.SelenideLogger;
import com.example.framework.config.ConfigProvider;
import com.example.framework.config.ProjectConfig;
import io.qameta.allure.selenide.AllureSelenide;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.BeforeEach;

import static com.codeborne.selenide.Selenide.closeWebDriver;

/**
 * Базовый класс для всех UI-тестов.
 * Настраивает браузер, Selenide и Allure.
 */
public abstract class BaseUiTest {

    protected static final ProjectConfig CONFIG = ConfigProvider.getConfig();

    @BeforeAll
    static void setupAllure() {
        // Подключение Allure listener для автоматического логирования шагов Selenide
        SelenideLogger.addListener("allure", new AllureSelenide()
                .screenshots(true)    // Скриншоты при падении
                .savePageSource(true) // Сохранение HTML при падении
        );
    }

    @BeforeEach
    void setupBrowser() {
        // Настройка Selenide из конфигурации
        Configuration.baseUrl = CONFIG.baseUrl();
        Configuration.browser = CONFIG.browser();
        Configuration.browserSize = CONFIG.browserSize();
        Configuration.timeout = CONFIG.timeout();
        Configuration.headless = CONFIG.headless();
        Configuration.pageLoadTimeout = 30000;
    }

    @AfterEach
    void teardown() {
        // Закрытие браузера после каждого теста
        closeWebDriver();
    }
}
```

### BaseApiTest.java

```java
package com.example.framework.base;

import com.example.framework.config.ConfigProvider;
import com.example.framework.config.ProjectConfig;
import io.qameta.allure.restassured.AllureRestAssured;
import io.restassured.RestAssured;
import io.restassured.filter.log.RequestLoggingFilter;
import io.restassured.filter.log.ResponseLoggingFilter;
import org.junit.jupiter.api.BeforeAll;

/**
 * Базовый класс для всех API-тестов.
 * Настраивает REST Assured и Allure.
 */
public abstract class BaseApiTest {

    protected static final ProjectConfig CONFIG = ConfigProvider.getConfig();

    @BeforeAll
    static void setupRestAssured() {
        // Базовый URI для всех запросов
        RestAssured.baseURI = CONFIG.apiBaseUrl();

        // Allure фильтр — логирование запросов/ответов в отчёт
        RestAssured.filters(
                new AllureRestAssured(),
                new RequestLoggingFilter(),
                new ResponseLoggingFilter()
        );
    }
}
```

### Модель данных: UserFormData.java

```java
package com.example.framework.models;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

/**
 * Модель данных для формы Text Box.
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UserFormData {
    private String fullName;
    private String email;
    private String currentAddress;
    private String permanentAddress;
}
```

### Фабрика данных: UserDataFactory.java

```java
package com.example.framework.data;

import com.example.framework.models.UserFormData;
import com.github.javafaker.Faker;

/**
 * Фабрика для создания тестовых данных пользователей.
 */
public class UserDataFactory {

    private static final Faker FAKER = new Faker();

    /**
     * Генерация случайных пользовательских данных.
     */
    public static UserFormData randomUserFormData() {
        return UserFormData.builder()
                .fullName(FAKER.name().fullName())
                .email(FAKER.internet().emailAddress())
                .currentAddress(FAKER.address().fullAddress())
                .permanentAddress(FAKER.address().fullAddress())
                .build();
    }

    /**
     * Фиксированные данные для smoke-тестов.
     */
    public static UserFormData defaultUserFormData() {
        return UserFormData.builder()
                .fullName("John Doe")
                .email("john.doe@test.com")
                .currentAddress("123 Main St, New York")
                .permanentAddress("456 Oak Ave, Los Angeles")
                .build();
    }
}
```

### Первый UI-тест: TextBoxTest.java

```java
package com.example.framework.tests.ui;

import com.example.framework.base.BaseUiTest;
import com.example.framework.data.UserDataFactory;
import com.example.framework.models.UserFormData;
import com.example.framework.pages.TextBoxPage;
import io.qameta.allure.Epic;
import io.qameta.allure.Feature;
import io.qameta.allure.Severity;
import io.qameta.allure.SeverityLevel;
import io.qameta.allure.Story;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

@Epic("DemoQA")
@Feature("Text Box")
@Tag("ui")
public class TextBoxTest extends BaseUiTest {

    private final TextBoxPage textBoxPage = new TextBoxPage();

    @Test
    @Story("Заполнение формы")
    @Severity(SeverityLevel.BLOCKER)
    @DisplayName("Успешная отправка формы с валидными данными")
    void successfulFormSubmission() {
        // Подготовка: генерируем случайные данные
        UserFormData userData = UserDataFactory.randomUserFormData();

        // Действие: заполняем и отправляем форму
        textBoxPage
                .openPage()
                .waitForPageLoaded()
                .enterFullName(userData.getFullName())
                .enterEmail(userData.getEmail())
                .enterCurrentAddress(userData.getCurrentAddress())
                .enterPermanentAddress(userData.getPermanentAddress())
                .clickSubmit();

        // Проверка: данные отображаются в результате
        textBoxPage
                .verifyOutputVisible()
                .verifyOutputName(userData.getFullName())
                .verifyOutputEmail(userData.getEmail());
    }

    @Test
    @Story("Заполнение формы")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("Отправка формы с фиксированными данными")
    void formSubmissionWithDefaultData() {
        UserFormData userData = UserDataFactory.defaultUserFormData();

        textBoxPage
                .openPage()
                .waitForPageLoaded()
                .enterFullName(userData.getFullName())
                .enterEmail(userData.getEmail())
                .clickSubmit();

        textBoxPage
                .verifyOutputVisible()
                .verifyOutputName(userData.getFullName())
                .verifyOutputEmail(userData.getEmail());
    }
}
```

### Первый API-тест: BookStoreApiTest.java

```java
package com.example.framework.tests.api;

import com.example.framework.base.BaseApiTest;
import io.qameta.allure.Epic;
import io.qameta.allure.Feature;
import io.qameta.allure.Severity;
import io.qameta.allure.SeverityLevel;
import io.qameta.allure.Story;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@Epic("DemoQA")
@Feature("Book Store API")
@Tag("api")
public class BookStoreApiTest extends BaseApiTest {

    @Test
    @Story("Получение списка книг")
    @Severity(SeverityLevel.BLOCKER)
    @DisplayName("GET /BookStore/v1/Books возвращает список книг")
    void getBookListReturnsBooks() {
        given()
        .when()
            .get("/BookStore/v1/Books")
        .then()
            .statusCode(200)
            .body("books", notNullValue())
            .body("books.size()", greaterThan(0))
            .body("books[0].isbn", notNullValue())
            .body("books[0].title", notNullValue());
    }

    @Test
    @Story("Получение книги по ISBN")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("GET /BookStore/v1/Book возвращает книгу по ISBN")
    void getBookByIsbnReturnsBook() {
        // Используем известный ISBN из DemoQA
        String isbn = "9781449325862";

        given()
            .queryParam("ISBN", isbn)
        .when()
            .get("/BookStore/v1/Book")
        .then()
            .statusCode(200)
            .body("isbn", equalTo(isbn))
            .body("title", notNullValue())
            .body("author", notNullValue());
    }

    @Test
    @Story("Обработка ошибок")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("GET /BookStore/v1/Book с несуществующим ISBN возвращает ошибку")
    void getBookByInvalidIsbnReturnsError() {
        given()
            .queryParam("ISBN", "0000000000000")
        .when()
            .get("/BookStore/v1/Book")
        .then()
            .statusCode(400)
            .body("code", notNullValue())
            .body("message", notNullValue());
    }
}
```

---

## Шаг 7: Настройка Allure-отчётности

### allure.properties

```properties
allure.results.directory=allure-results
allure.link.tms.pattern=https://jira.company.com/browse/{}
allure.link.issue.pattern=https://jira.company.com/browse/{}
```

### junit-platform.properties

```properties
# Параллельный запуск (опционально, можно включить позже)
# junit.jupiter.execution.parallel.enabled=true
# junit.jupiter.execution.parallel.mode.default=concurrent
# junit.jupiter.execution.parallel.config.strategy=fixed
# junit.jupiter.execution.parallel.config.fixed.parallelism=4
```

### Утилита для прикрепления файлов к Allure

```java
package com.example.framework.utils;

import io.qameta.allure.Allure;
import io.qameta.allure.Attachment;
import org.openqa.selenium.OutputType;
import org.openqa.selenium.TakesScreenshot;

import java.io.ByteArrayInputStream;
import java.nio.charset.StandardCharsets;

import static com.codeborne.selenide.WebDriverRunner.getWebDriver;

/**
 * Утилиты для расширения Allure-отчётов.
 */
public class AllureUtils {

    /**
     * Прикрепление скриншота к отчёту.
     */
    @Attachment(value = "Скриншот", type = "image/png")
    public static byte[] attachScreenshot() {
        return ((TakesScreenshot) getWebDriver())
                .getScreenshotAs(OutputType.BYTES);
    }

    /**
     * Прикрепление текста (например, лог или тело ответа).
     */
    public static void attachText(String name, String content) {
        Allure.addAttachment(
                name,
                "text/plain",
                new ByteArrayInputStream(
                        content.getBytes(StandardCharsets.UTF_8)
                ),
                ".txt"
        );
    }

    /**
     * Прикрепление JSON-ответа.
     */
    public static void attachJson(String name, String json) {
        Allure.addAttachment(
                name,
                "application/json",
                new ByteArrayInputStream(
                        json.getBytes(StandardCharsets.UTF_8)
                ),
                ".json"
        );
    }
}
```

---

## Шаг 8: Запуск и генерация отчёта

### Запуск тестов

```bash
# Запуск всех тестов на dev-окружении
mvn clean test

# Запуск на staging
mvn clean test -Denv=staging

# Запуск только UI-тестов
mvn clean test -Dgroups=ui

# Запуск только API-тестов
mvn clean test -Dgroups=api

# Запуск в headless-режиме (для CI/CD)
mvn clean test -Dbrowser.headless=true

# Запуск конкретного теста
mvn clean test -Dtest=TextBoxTest

# Запуск с другим браузером
mvn clean test -Dbrowser=firefox
```

### Генерация Allure-отчёта

```bash
# Генерация и открытие отчёта в браузере
mvn allure:serve

# Только генерация отчёта (без открытия)
mvn allure:report
# Отчёт будет в target/site/allure-maven-plugin/
```

### Что будет в Allure-отчёте

- **Overview** — общая статистика: сколько прошло, упало, пропущено.
- **Suites** — группировка по тестовым классам.
- **Behaviors** — группировка по Epic → Feature → Story.
- **Timeline** — хронология выполнения (видно параллелизм).
- **Для каждого теста:**
  - Шаги выполнения (из `@Step` аннотаций).
  - Скриншот при падении (автоматически от AllureSelenide).
  - HTML-страницы при падении.
  - HTTP-запросы и ответы для API-тестов.

---

## Итоговая структура проекта

```
test-framework/
├── pom.xml
├── .gitignore
└── src/test/
    ├── java/com/example/framework/
    │   ├── config/
    │   │   ├── ProjectConfig.java       ← Owner-интерфейс
    │   │   └── ConfigProvider.java       ← Singleton-провайдер
    │   ├── base/
    │   │   ├── BasePage.java            ← Базовый Page Object
    │   │   ├── BaseUiTest.java          ← Базовый UI-тест
    │   │   └── BaseApiTest.java         ← Базовый API-тест
    │   ├── pages/
    │   │   └── TextBoxPage.java         ← Page Object
    │   ├── models/
    │   │   └── UserFormData.java        ← Модель данных
    │   ├── data/
    │   │   └── UserDataFactory.java     ← Фабрика данных
    │   ├── utils/
    │   │   └── AllureUtils.java         ← Утилиты отчётности
    │   └── tests/
    │       ├── ui/
    │       │   └── TextBoxTest.java     ← UI-тесты
    │       └── api/
    │           └── BookStoreApiTest.java ← API-тесты
    └── resources/
        ├── config/
        │   ├── application.properties
        │   ├── dev.properties
        │   └── staging.properties
        ├── allure.properties
        └── junit-platform.properties
```

---

## Связь с тестированием

Этот фреймворк — отправная точка для реального проекта автоматизации:

- **Selenide** — для стабильных UI-тестов с автоматическими ожиданиями.
- **REST Assured** — для быстрых и надёжных API-тестов.
- **JUnit 5** — современный фреймворк с параллелизмом, параметризацией, расширениями.
- **Allure** — красивые отчёты, понятные всей команде (включая менеджеров).
- **Owner** — гибкое переключение окружений.
- **Lombok** — минимум boilerplate-кода.

Фреймворк легко расширяется: добавление DB-слоя, mobile-тестов, новых Page Objects.

---

## Типичные ошибки

1. **Начинать с «идеальной» архитектуры** — начните с минимума, развивайте итеративно.
2. **Забыть AspectJ weaver** — без него Allure `@Step` не работает.
3. **Не закрывать браузер** — утечка ресурсов, тесты тормозят.
4. **Хардкод URL в тестах** — используйте `Configuration.baseUrl` из Selenide.
5. **Не использовать `@DisplayName`** — в отчёте непонятные имена тестов.
6. **Игнорирование `.gitignore`** — `allure-results/` и `target/` попадают в Git.
7. **Слишком много зависимостей** — добавляйте только то, что реально используете.
8. **Нет Allure-аннотаций** — отчёт существует, но бесполезен без `@Epic`, `@Feature`, `@Story`.

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Какие зависимости необходимы для базового тестового фреймворка на Java?
2. Зачем нужен `maven-surefire-plugin`?
3. Что такое Selenide и чем он отличается от чистого Selenium?
4. Как запустить тесты через Maven?
5. Что делает аннотация `@BeforeAll`?

### 🟡 Средний уровень
6. Как организовать конфигурацию для разных окружений?
7. Зачем нужен AspectJ weaver для Allure?
8. Как настроить Allure-отчётность в Maven-проекте?
9. Для чего нужен базовый класс `BaseUiTest`? Что в нём должно быть?
10. Как запустить только определённую группу тестов (`@Tag`)?

### 🔴 Продвинутый уровень
11. Как бы вы расширили этот фреймворк для тестирования мобильных приложений?
12. Как организовать запуск тестов в Docker?
13. Как добавить кастомный Allure listener для автоматического прикрепления логов?
14. Как интегрировать этот фреймворк с GitHub Actions?
15. Как организовать multi-module проект, если UI и API тесты должны запускаться независимо?

---

## Практические задания

### Задание 1: Воспроизведение
Создайте проект по этому гайду с нуля. Убедитесь, что:
- Проект компилируется (`mvn compile`).
- UI-тесты проходят (`mvn test -Dgroups=ui`).
- API-тесты проходят (`mvn test -Dgroups=api`).
- Allure-отчёт генерируется (`mvn allure:serve`).

### Задание 2: Расширение
Добавьте в проект:
- Новый Page Object для страницы [demoqa.com/buttons](https://demoqa.com/buttons).
- Тест на двойной клик и правый клик.
- Прикрепление скриншотов через `AllureUtils`.

### Задание 3: CI/CD
Напишите GitHub Actions workflow (`ci.yml`), который:
- Запускает тесты при push и pull request.
- Генерирует Allure-отчёт.
- Публикует отчёт на GitHub Pages.

### Задание 4: Dockerизация
Создайте `Dockerfile` и `docker-compose.yml` для запуска тестов:
- Chrome в Docker (через Selenoid или Selenium Grid).
- Тестовый фреймворк в отдельном контейнере.
- Allure-отчёт доступен через HTTP.

---

## Дополнительные ресурсы

- [Selenide — Official Documentation](https://selenide.org/documentation.html)
- [REST Assured — Getting Started](https://rest-assured.io/)
- [JUnit 5 User Guide](https://junit.org/junit5/docs/current/user-guide/)
- [Allure Report — Documentation](https://docs.qameta.io/allure-report/)
- [Lombok — Features](https://projectlombok.org/features/)
- [Owner Library](http://owner.aeonbits.org/)
- [DemoQA — Practice Site](https://demoqa.com/)
- [Maven Surefire Plugin](https://maven.apache.org/surefire/maven-surefire-plugin/)
