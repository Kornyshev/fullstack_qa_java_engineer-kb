# Первый проект на Selenide + Allure

## Содержание

1. [Цель практики](#цель-практики)
2. [Selenide vs Selenium: ключевые отличия](#selenide-vs-selenium-ключевые-отличия)
3. [Шаг 1 — Создание Maven-проекта](#шаг-1--создание-maven-проекта)
4. [Шаг 2 — Зависимости в pom.xml](#шаг-2--зависимости-в-pomxml)
5. [Шаг 3 — Структура проекта](#шаг-3--структура-проекта)
6. [Шаг 4 — Конфигурация Selenide](#шаг-4--конфигурация-selenide)
7. [Шаг 5 — BasePage на Selenide](#шаг-5--basepage-на-selenide)
8. [Шаг 6 — LoginPage с $ и $$](#шаг-6--loginpage-с--и-)
9. [Шаг 7 — ProductsPage](#шаг-7--productspage)
10. [Шаг 8 — BaseTest](#шаг-8--basetest)
11. [Шаг 9 — Тесты с shouldHave/shouldBe](#шаг-9--тесты-с-shouldhaveshouldbe)
12. [Шаг 10 — Allure-аннотации](#шаг-10--allure-аннотации)
13. [Шаг 11 — Генерация Allure-отчёта](#шаг-11--генерация-allure-отчёта)
14. [Сравнение кода: Selenium vs Selenide](#сравнение-кода-selenium-vs-selenide)
15. [Задания для самостоятельной работы](#задания-для-самостоятельной-работы)

---

## Цель практики

Создать проект UI-автоматизации на **Selenide** с интеграцией **Allure Reports**. Вы увидите, как Selenide упрощает код по сравнению с чистым Selenium, и научитесь генерировать наглядные отчёты.

**Целевой сайт:** [https://www.saucedemo.com](https://www.saucedemo.com)

---

## Selenide vs Selenium: ключевые отличия

| Аспект | Selenium | Selenide |
|--------|----------|----------|
| Поиск элемента | `driver.findElement(By.id("x"))` | `$("[id='x']")` или `$("#x")` |
| Ожидание | Ручной `WebDriverWait` | Встроенное автоматическое ожидание |
| Assertions | JUnit `assertEquals` | `shouldHave(text("..."))` |
| Скриншоты при падении | Ручная настройка | Автоматически |
| Управление браузером | `WebDriverManager` + конструктор | `Configuration.browser` |
| Коллекции | `List<WebElement>` + Stream | `$$("selector").shouldHave(size(6))` |
| Объём кода | Больше | Значительно меньше |

---

## Шаг 1 — Создание Maven-проекта

```bash
mvn archetype:generate \
  -DgroupId=com.saucedemo \
  -DartifactId=selenide-saucedemo \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DarchetypeVersion=1.4 \
  -DinteractiveMode=false
```

Удалите `App.java` и `AppTest.java`.

---

## Шаг 2 — Зависимости в pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.saucedemo</groupId>
    <artifactId>selenide-saucedemo</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <!-- Версии -->
        <selenide.version>7.5.1</selenide.version>
        <junit.version>5.11.3</junit.version>
        <allure.version>2.29.1</allure.version>
        <allure-selenide.version>2.29.1</allure-selenide.version>
        <aspectj.version>1.9.22.1</aspectj.version>
        <maven-surefire.version>3.5.2</maven-surefire.version>
    </properties>

    <dependencies>
        <!-- Selenide -->
        <dependency>
            <groupId>com.codeborne</groupId>
            <artifactId>selenide</artifactId>
            <version>${selenide.version}</version>
        </dependency>

        <!-- JUnit 5 -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>${junit.version}</version>
            <scope>test</scope>
        </dependency>

        <!-- Allure JUnit 5 -->
        <dependency>
            <groupId>io.qameta.allure</groupId>
            <artifactId>allure-junit5</artifactId>
            <version>${allure.version}</version>
            <scope>test</scope>
        </dependency>

        <!-- Allure Selenide — автоприкрепление скриншотов -->
        <dependency>
            <groupId>io.qameta.allure</groupId>
            <artifactId>allure-selenide</artifactId>
            <version>${allure-selenide.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>${maven-surefire.version}</version>
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
        </plugins>
    </build>
</project>
```

> **AspectJ Weaver** нужен для работы Allure-аннотаций `@Step`. Без него аннотации будут проигнорированы, и шаги не появятся в отчёте.

---

## Шаг 3 — Структура проекта

```
selenide-saucedemo/
├── pom.xml
└── src/
    ├── main/
    │   └── java/
    │       └── com/saucedemo/
    │           └── pages/
    │               ├── BasePage.java
    │               ├── LoginPage.java
    │               └── ProductsPage.java
    └── test/
        ├── java/
        │   └── com/saucedemo/
        │       └── tests/
        │           ├── BaseTest.java
        │           └── LoginTest.java
        └── resources/
            └── selenide.properties
```

---

## Шаг 4 — Конфигурация Selenide

### Вариант 1: selenide.properties (рекомендуемый)

Файл: `src/test/resources/selenide.properties`

```properties
# Браузер
selenide.browser=chrome
# Базовый URL
selenide.baseUrl=https://www.saucedemo.com
# Таймаут ожидания (мс)
selenide.timeout=10000
# Размер окна
selenide.browserSize=1920x1080
# Скриншоты при падении
selenide.screenshots=true
# Сохранять HTML страницы при падении
selenide.savePageSource=false
```

Selenide автоматически подхватывает этот файл — никакого дополнительного кода не требуется.

### Вариант 2: программная конфигурация в BaseTest

```java
import com.codeborne.selenide.Configuration;

Configuration.browser = "chrome";
Configuration.baseUrl = "https://www.saucedemo.com";
Configuration.timeout = 10000;
Configuration.browserSize = "1920x1080";
```

> **Когда какой вариант?** Используйте `selenide.properties` для значений по умолчанию. Программную конфигурацию — когда нужно менять параметры динамически (например, `Configuration.headless = true` для CI).

---

## Шаг 5 — BasePage на Selenide

Файл: `src/main/java/com/saucedemo/pages/BasePage.java`

```java
package com.saucedemo.pages;

import com.codeborne.selenide.Condition;
import com.codeborne.selenide.SelenideElement;
import io.qameta.allure.Step;

import static com.codeborne.selenide.Selenide.page;

/**
 * Базовый класс для Page Object на Selenide.
 * В отличие от Selenium-версии, здесь значительно меньше кода —
 * Selenide берёт на себя ожидания, поиск элементов и скриншоты.
 */
public abstract class BasePage {

    /**
     * Проверить, что элемент виден на странице.
     * Selenide автоматически ждёт появления элемента в пределах timeout.
     */
    protected void shouldBeVisible(SelenideElement element) {
        element.shouldBe(Condition.visible);
    }

    /**
     * Получить текст элемента с ожиданием видимости.
     */
    protected String getTextOf(SelenideElement element) {
        return element.shouldBe(Condition.visible).getText();
    }

    /**
     * Создать экземпляр Page Object через Selenide.
     * Selenide автоматически инициализирует элементы с @FindBy.
     */
    protected <T extends BasePage> T navigateTo(Class<T> pageClass) {
        return page(pageClass);
    }
}
```

> **Обратите внимание**, как мало кода по сравнению с `BasePage` на чистом Selenium. В Selenide не нужен `WebDriver` в конструкторе, не нужен `WebDriverWait`, не нужны обёртки вокруг `findElement`.

---

## Шаг 6 — LoginPage с $ и $$

Файл: `src/main/java/com/saucedemo/pages/LoginPage.java`

```java
package com.saucedemo.pages;

import com.codeborne.selenide.SelenideElement;
import io.qameta.allure.Step;

import static com.codeborne.selenide.Selenide.$;
import static com.codeborne.selenide.Selenide.open;

/**
 * Page Object для страницы авторизации SauceDemo.
 * Использует Selenide-сокращения: $ для поиска одного элемента.
 */
public class LoginPage extends BasePage {

    // === Элементы ===
    // $(селектор) — аналог driver.findElement(), но с автоматическим ожиданием
    private final SelenideElement usernameField = $("#user-name");
    private final SelenideElement passwordField = $("#password");
    private final SelenideElement loginButton = $("#login-button");
    private final SelenideElement errorMessage = $("[data-test='error']");

    @Step("Открыть страницу логина")
    public LoginPage openPage() {
        // Selenide использует baseUrl из конфигурации
        open("/");
        return this;
    }

    @Step("Ввести имя пользователя: {username}")
    public LoginPage enterUsername(String username) {
        usernameField.setValue(username);
        return this;
    }

    @Step("Ввести пароль")
    public LoginPage enterPassword(String password) {
        // Не логируем пароль в @Step из соображений безопасности
        passwordField.setValue(password);
        return this;
    }

    @Step("Нажать кнопку Login")
    public LoginPage clickLogin() {
        loginButton.click();
        return this;
    }

    @Step("Выполнить логин: {username}")
    public ProductsPage loginAs(String username, String password) {
        enterUsername(username);
        enterPassword(password);
        clickLogin();
        return navigateTo(ProductsPage.class);
    }

    /** Получить элемент сообщения об ошибке (для assertions в тесте) */
    public SelenideElement getErrorMessage() {
        return errorMessage;
    }
}
```

### Основные Selenide-методы для элементов

| Метод | Назначение | Аналог в Selenium |
|-------|------------|-------------------|
| `$(selector)` | Найти один элемент | `driver.findElement(By.cssSelector(...))` |
| `$$(selector)` | Найти коллекцию | `driver.findElements(By.cssSelector(...))` |
| `.setValue(text)` | Очистить и ввести | `element.clear(); element.sendKeys(text)` |
| `.click()` | Кликнуть | `element.click()` |
| `.getText()` | Получить текст | `element.getText()` |
| `.shouldBe(visible)` | Проверить видимость | `wait.until(visibilityOf(...))` |
| `.shouldHave(text("x"))` | Проверить текст | `assertEquals("x", element.getText())` |

---

## Шаг 7 — ProductsPage

Файл: `src/main/java/com/saucedemo/pages/ProductsPage.java`

```java
package com.saucedemo.pages;

import com.codeborne.selenide.ElementsCollection;
import com.codeborne.selenide.SelenideElement;
import io.qameta.allure.Step;

import static com.codeborne.selenide.Selenide.$;
import static com.codeborne.selenide.Selenide.$$;

/**
 * Page Object для страницы каталога товаров.
 */
public class ProductsPage extends BasePage {

    // === Элементы ===
    private final SelenideElement pageTitle = $(".title");
    private final SelenideElement cartBadge = $(".shopping_cart_badge");
    private final SelenideElement cartLink = $(".shopping_cart_link");
    private final SelenideElement sortDropdown = $(".product_sort_container");

    // $$ — коллекция элементов
    private final ElementsCollection inventoryItems = $$(".inventory_item");
    private final ElementsCollection inventoryItemNames = $$(".inventory_item_name");

    /** Элемент заголовка — для assertions в тесте */
    public SelenideElement getPageTitle() {
        return pageTitle;
    }

    /** Коллекция товаров — для assertions в тесте */
    public ElementsCollection getInventoryItems() {
        return inventoryItems;
    }

    /** Коллекция названий товаров */
    public ElementsCollection getInventoryItemNames() {
        return inventoryItemNames;
    }

    /** Элемент значка корзины */
    public SelenideElement getCartBadge() {
        return cartBadge;
    }

    @Step("Добавить в корзину: {productName}")
    public ProductsPage addToCart(String productName) {
        String buttonId = "add-to-cart-" + productName
                .toLowerCase()
                .replace(" ", "-");
        $("#" + buttonId).click();
        return this;
    }

    @Step("Удалить из корзины: {productName}")
    public ProductsPage removeFromCart(String productName) {
        String buttonId = "remove-" + productName
                .toLowerCase()
                .replace(" ", "-");
        $("#" + buttonId).click();
        return this;
    }

    @Step("Перейти в корзину")
    public void goToCart() {
        cartLink.click();
    }

    @Step("Выбрать сортировку: {option}")
    public ProductsPage sortBy(String option) {
        sortDropdown.selectOption(option);
        return this;
    }
}
```

> **`ElementsCollection`** — Selenide-обёртка над `List<WebElement>`. Поддерживает `shouldHave(size(6))`, `filter(visible)`, `first()`, `last()` и другие удобные методы.

---

## Шаг 8 — BaseTest

Файл: `src/test/java/com/saucedemo/tests/BaseTest.java`

```java
package com.saucedemo.tests;

import com.codeborne.selenide.Configuration;
import com.codeborne.selenide.logevents.SelenideLogger;
import io.qameta.allure.selenide.AllureSelenide;
import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.BeforeEach;

import static com.codeborne.selenide.Selenide.clearBrowserCookies;
import static com.codeborne.selenide.Selenide.clearBrowserLocalStorage;

/**
 * Базовый класс для всех тестов.
 * Selenide управляет браузером автоматически — не нужен явный quit().
 */
public abstract class BaseTest {

    @BeforeAll
    static void setUpAllure() {
        // Подключить Allure listener — логирует все Selenide-действия как шаги
        SelenideLogger.addListener("allure", new AllureSelenide()
                .screenshots(true)      // Скриншот при падении
                .savePageSource(false)   // Не сохранять HTML
        );
    }

    @BeforeEach
    void cleanUp() {
        // Очистить состояние браузера между тестами
        clearBrowserCookies();
        clearBrowserLocalStorage();
    }

    @AfterAll
    static void tearDownAllure() {
        SelenideLogger.removeListener("allure");
    }
}
```

> **Selenide автоматически закрывает браузер** после прогона. Не нужен `@AfterEach` с `driver.quit()`. Selenide также автоматически делает скриншот при падении теста.

---

## Шаг 9 — Тесты с shouldHave/shouldBe

Файл: `src/test/java/com/saucedemo/tests/LoginTest.java`

```java
package com.saucedemo.tests;

import com.saucedemo.pages.LoginPage;
import com.saucedemo.pages.ProductsPage;
import io.qameta.allure.Epic;
import io.qameta.allure.Feature;
import io.qameta.allure.Severity;
import io.qameta.allure.SeverityLevel;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import static com.codeborne.selenide.CollectionCondition.size;
import static com.codeborne.selenide.Condition.*;

/**
 * Тесты авторизации SauceDemo на Selenide.
 */
@Epic("SauceDemo")
@Feature("Авторизация")
class LoginTest extends BaseTest {

    private LoginPage loginPage;

    @BeforeEach
    void openLoginPage() {
        loginPage = new LoginPage();
        loginPage.openPage();
    }

    @Test
    @DisplayName("Успешный логин со стандартным пользователем")
    @Severity(SeverityLevel.BLOCKER)
    void testSuccessfulLogin() {
        // Act
        ProductsPage productsPage = loginPage.loginAs("standard_user", "secret_sauce");

        // Assert — Selenide-стиль: читаемый, декларативный
        productsPage.getPageTitle()
                .shouldBe(visible)
                .shouldHave(text("Products"));

        productsPage.getInventoryItems()
                .shouldHave(size(6));
    }

    @Test
    @DisplayName("Логин с заблокированным пользователем")
    @Severity(SeverityLevel.CRITICAL)
    void testLockedOutUserLogin() {
        // Act
        loginPage.enterUsername("locked_out_user");
        loginPage.enterPassword("secret_sauce");
        loginPage.clickLogin();

        // Assert
        loginPage.getErrorMessage()
                .shouldBe(visible)
                .shouldHave(text("Sorry, this user has been locked out."));
    }

    @Test
    @DisplayName("Логин с пустым именем пользователя")
    @Severity(SeverityLevel.NORMAL)
    void testEmptyUsername() {
        loginPage.enterPassword("secret_sauce");
        loginPage.clickLogin();

        loginPage.getErrorMessage()
                .shouldBe(visible)
                .shouldHave(text("Username is required"));
    }

    @Test
    @DisplayName("Логин с пустым паролем")
    @Severity(SeverityLevel.NORMAL)
    void testEmptyPassword() {
        loginPage.enterUsername("standard_user");
        loginPage.clickLogin();

        loginPage.getErrorMessage()
                .shouldBe(visible)
                .shouldHave(text("Password is required"));
    }

    @Test
    @DisplayName("Логин с неверными учётными данными")
    @Severity(SeverityLevel.CRITICAL)
    void testInvalidCredentials() {
        loginPage.enterUsername("wrong_user");
        loginPage.enterPassword("wrong_pass");
        loginPage.clickLogin();

        loginPage.getErrorMessage()
                .shouldBe(visible)
                .shouldHave(text("do not match"));
    }

    @Test
    @DisplayName("Добавление товара в корзину после логина")
    @Severity(SeverityLevel.CRITICAL)
    void testAddToCartAfterLogin() {
        // Arrange + Act
        ProductsPage productsPage = loginPage.loginAs("standard_user", "secret_sauce");
        productsPage.addToCart("sauce-labs-backpack");

        // Assert
        productsPage.getCartBadge()
                .shouldBe(visible)
                .shouldHave(text("1"));
    }

    @Test
    @DisplayName("Названия товаров не пустые")
    @Severity(SeverityLevel.NORMAL)
    void testProductNamesNotEmpty() {
        ProductsPage productsPage = loginPage.loginAs("standard_user", "secret_sauce");

        productsPage.getInventoryItemNames()
                .shouldHave(size(6));

        // Проверить, что каждое название не пустое
        productsPage.getInventoryItemNames()
                .forEach(name -> name.shouldNotBe(empty));
    }
}
```

### Selenide Conditions — справочник

| Condition | Пример | Что проверяет |
|-----------|--------|---------------|
| `visible` | `.shouldBe(visible)` | Элемент видим на странице |
| `hidden` | `.shouldBe(hidden)` | Элемент скрыт |
| `enabled` | `.shouldBe(enabled)` | Элемент активен (не disabled) |
| `text("X")` | `.shouldHave(text("X"))` | Содержит текст (без учёта регистра) |
| `exactText("X")` | `.shouldHave(exactText("X"))` | Точное совпадение текста |
| `attribute("a","v")` | `.shouldHave(attribute("a","v"))` | Атрибут имеет значение |
| `cssClass("c")` | `.shouldHave(cssClass("c"))` | Элемент имеет CSS-класс |
| `empty` | `.shouldBe(empty)` | Поле ввода пустое |
| `value("X")` | `.shouldHave(value("X"))` | Значение input-поля |

### CollectionCondition — для коллекций

| Condition | Пример |
|-----------|--------|
| `size(n)` | `$$(".item").shouldHave(size(6))` |
| `sizeGreaterThan(n)` | `$$(".item").shouldHave(sizeGreaterThan(0))` |
| `texts("A","B")` | `$$(".item").shouldHave(texts("A","B"))` |
| `exactTexts(...)` | Точный порядок и совпадение |

---

## Шаг 10 — Allure-аннотации

### Доступные аннотации

| Аннотация | Уровень | Назначение |
|-----------|---------|------------|
| `@Epic` | Класс | Эпик (крупная функциональность) |
| `@Feature` | Класс | Фича внутри эпика |
| `@Story` | Метод | Конкретный сценарий |
| `@Step` | Метод Page Object | Шаг теста (отображается в отчёте) |
| `@Severity` | Метод | Критичность: BLOCKER, CRITICAL, NORMAL, MINOR, TRIVIAL |
| `@Description` | Метод | Подробное описание теста |
| `@Link` | Метод | Ссылка на задачу в Jira или wiki |

### Пример с полным набором аннотаций

```java
@Epic("SauceDemo")
@Feature("Корзина")
@Story("Добавление товара в корзину")
@Severity(SeverityLevel.CRITICAL)
@Description("Проверяет, что товар добавляется в корзину и badge обновляется")
@Link(name = "JIRA-123", url = "https://jira.example.com/browse/JIRA-123")
@DisplayName("Добавление одного товара в корзину")
@Test
void testAddSingleItemToCart() {
    // ...
}
```

---

## Шаг 11 — Генерация Allure-отчёта

### Установка Allure CLI

```bash
# macOS
brew install allure

# Linux (через snap)
sudo snap install allure

# Windows (через scoop)
scoop install allure

# Проверка установки
allure --version
```

### Генерация и просмотр отчёта

```bash
# 1. Запустить тесты (генерируются allure-results)
mvn clean test

# 2. Открыть отчёт в браузере (генерирует HTML и запускает локальный сервер)
allure serve target/allure-results
```

Команда `allure serve` откроет браузер с интерактивным отчётом.

### Что увидите в отчёте

1. **Overview** — общая статистика: пройдено/упало/сломано/пропущено
2. **Suites** — группировка по тестовым классам
3. **Behaviors** — группировка по Epic → Feature → Story
4. **Graphs** — диаграммы: severity, duration, trend
5. **Timeline** — хронология выполнения тестов
6. **Каждый тест** — шаги (@Step), скриншот при падении, время выполнения

### Генерация статического отчёта

```bash
# Сгенерировать HTML-отчёт в папку allure-report
allure generate target/allure-results -o target/allure-report --clean

# Открыть сгенерированный отчёт
allure open target/allure-report
```

> **Совет.** В CI/CD используйте `allure generate` и публикуйте папку `allure-report` как артефакт сборки.

---

## Сравнение кода: Selenium vs Selenide

### Поиск элемента и клик

```java
// Selenium — 3 строки
WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));
WebElement button = wait.until(ExpectedConditions.elementToBeClickable(By.id("login-button")));
button.click();

// Selenide — 1 строка
$("#login-button").click();
```

### Проверка текста

```java
// Selenium
String actual = driver.findElement(By.className("title")).getText();
assertEquals("Products", actual);

// Selenide
$(".title").shouldHave(text("Products"));
```

### Проверка коллекции

```java
// Selenium — 4 строки
List<WebElement> items = driver.findElements(By.className("inventory_item"));
assertEquals(6, items.size());
for (WebElement item : items) {
    assertFalse(item.getText().isEmpty());
}

// Selenide — 2 строки
$$(".inventory_item").shouldHave(size(6));
$$(".inventory_item").forEach(item -> item.shouldNotBe(empty));
```

---

## Задания для самостоятельной работы

### Задание 1 — CartPage

Создайте `CartPage` с Selenide. Методы:
- `getCartItems()` — возвращает `ElementsCollection`
- `removeItem(String name)` — удаляет товар из корзины
- `clickCheckout()` — переход к оформлению

### Задание 2 — Тест полного checkout-потока

Напишите E2E-тест: логин → добавить 2 товара → перейти в корзину → checkout → заполнить форму → finish.

### Задание 3 — Параметризованный тест на Selenide

```java
@ParameterizedTest(name = "Сортировка: {0}")
@ValueSource(strings = {
    "Name (A to Z)",
    "Name (Z to A)",
    "Price (low to high)",
    "Price (high to low)"
})
void testSortingOptions(String sortOption) {
    // Реализуйте: выбрать сортировку и проверить порядок товаров
}
```

### Задание 4 — Custom Allure attachment

Добавьте метод для прикрепления скриншота вручную:

```java
@Attachment(value = "Скриншот", type = "image/png")
public byte[] takeScreenshot() {
    return Selenide.screenshot(OutputType.BYTES);
}
```

---

**Результат практики:** проект на Selenide с Allure-отчётностью. Тесты значительно короче и читабельнее, чем на чистом Selenium. Отчёты содержат шаги, скриншоты и группировку по Epic/Feature. Запуск: `mvn clean test && allure serve target/allure-results`.
