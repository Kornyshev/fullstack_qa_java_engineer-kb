# Первый проект на Playwright Java

## Содержание

1. [Цель практики](#цель-практики)
2. [Playwright: ключевые особенности](#playwright-ключевые-особенности)
3. [Шаг 1 — Создание Maven-проекта](#шаг-1--создание-maven-проекта)
4. [Шаг 2 — Зависимости в pom.xml](#шаг-2--зависимости-в-pomxml)
5. [Шаг 3 — Установка браузеров](#шаг-3--установка-браузеров)
6. [Шаг 4 — Структура проекта](#шаг-4--структура-проекта)
7. [Шаг 5 — Архитектура Playwright: Browser → Context → Page](#шаг-5--архитектура-playwright-browser--context--page)
8. [Шаг 6 — BaseTest: lifecycle браузера](#шаг-6--basetest-lifecycle-браузера)
9. [Шаг 7 — Locators: getByRole, getByText, getByTestId](#шаг-7--locators-getbyrole-getbytext-getbytestid)
10. [Шаг 8 — Assertions: expect](#шаг-8--assertions-expect)
11. [Шаг 9 — LoginPage с Playwright-локаторами](#шаг-9--loginpage-с-playwright-локаторами)
12. [Шаг 10 — ProductsPage](#шаг-10--productspage)
13. [Шаг 11 — Тесты авторизации](#шаг-11--тесты-авторизации)
14. [Шаг 12 — Скриншоты и трассировка](#шаг-12--скриншоты-и-трассировка)
15. [Шаг 13 — Запуск и отладка](#шаг-13--запуск-и-отладка)
16. [Playwright vs Selenium: сравнение](#playwright-vs-selenium-сравнение)
17. [Задания для самостоятельной работы](#задания-для-самостоятельной-работы)

---

## Цель практики

Создать проект UI-автоматизации на **Playwright for Java**. Вы освоите современный подход к локаторам (`getByRole`, `getByTestId`), встроенные assertions и возможности трассировки.

**Целевой сайт:** [https://www.saucedemo.com](https://www.saucedemo.com)

---

## Playwright: ключевые особенности

| Особенность | Описание |
|-------------|----------|
| Auto-waiting | Автоматическое ожидание элементов перед любым действием |
| Web-first assertions | `expect(locator).toHaveText("X")` — ждёт выполнения условия |
| Семантические локаторы | `getByRole`, `getByText`, `getByLabel`, `getByTestId` |
| Trace viewer | Запись полного trace: скриншоты, DOM, сеть, действия |
| Изоляция через BrowserContext | Каждый тест получает чистый контекст (cookies, localStorage) |
| Поддержка всех браузеров | Chromium, Firefox, WebKit (Safari) из коробки |

---

## Шаг 1 — Создание Maven-проекта

```bash
mvn archetype:generate \
  -DgroupId=com.saucedemo \
  -DartifactId=playwright-saucedemo \
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
    <artifactId>playwright-saucedemo</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <playwright.version>1.49.0</playwright.version>
        <junit.version>5.11.3</junit.version>
        <maven-surefire.version>3.5.2</maven-surefire.version>
    </properties>

    <dependencies>
        <!-- Playwright -->
        <dependency>
            <groupId>com.microsoft.playwright</groupId>
            <artifactId>playwright</artifactId>
            <version>${playwright.version}</version>
        </dependency>

        <!-- JUnit 5 -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>${junit.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>${maven-surefire.version}</version>
            </plugin>
        </plugins>
    </build>
</project>
```

> **Playwright не требует WebDriverManager.** Браузеры скачиваются через CLI Playwright и управляются напрямую через CDP/протоколы, без промежуточного WebDriver.

---

## Шаг 3 — Установка браузеров

После добавления зависимости необходимо скачать браузеры:

```bash
# Установить все браузеры (Chromium, Firefox, WebKit)
mvn exec:java -e -D exec.mainClass=com.microsoft.playwright.CLI -D exec.args="install"

# Или только Chromium (быстрее)
mvn exec:java -e -D exec.mainClass=com.microsoft.playwright.CLI -D exec.args="install chromium"
```

Эту команду нужно выполнить один раз. Браузеры сохраняются в системный кэш.

> **Что скачивается?** Не обычный Chrome, а специальные сборки браузеров, оптимизированные для автоматизации. Chromium, Firefox и WebKit (движок Safari).

---

## Шаг 4 — Структура проекта

```
playwright-saucedemo/
├── pom.xml
└── src/
    └── test/
        └── java/
            └── com/saucedemo/
                ├── base/
                │   └── BaseTest.java
                ├── pages/
                │   ├── LoginPage.java
                │   └── ProductsPage.java
                └── tests/
                    └── LoginTest.java
```

> **В Playwright-проектах** Page Object часто размещают в `src/test/java`, а не в `src/main/java`, так как они используются только тестами. Оба подхода допустимы.

---

## Шаг 5 — Архитектура Playwright: Browser → Context → Page

Playwright имеет трёхуровневую архитектуру:

```
Playwright (точка входа)
└── Browser (экземпляр браузера, один на все тесты)
    └── BrowserContext (изолированная сессия, один на тест)
        └── Page (вкладка, один на тест)
```

| Уровень | Аналог | Жизненный цикл |
|---------|--------|-----------------|
| `Playwright` | — | Весь тестовый запуск |
| `Browser` | Процесс Chrome | Весь тестовый класс (`@BeforeAll/@AfterAll`) |
| `BrowserContext` | Incognito-окно | Один тест (`@BeforeEach/@AfterEach`) |
| `Page` | Вкладка браузера | Один тест |

### Почему такая архитектура?

- **Browser** запускается долго, поэтому один экземпляр на весь класс
- **BrowserContext** обеспечивает изоляцию: чистые cookies, localStorage, кэш
- **Page** — рабочая единица, с которой взаимодействует тест

---

## Шаг 6 — BaseTest: lifecycle браузера

Файл: `src/test/java/com/saucedemo/base/BaseTest.java`

```java
package com.saucedemo.base;

import com.microsoft.playwright.Browser;
import com.microsoft.playwright.BrowserContext;
import com.microsoft.playwright.BrowserType;
import com.microsoft.playwright.Page;
import com.microsoft.playwright.Playwright;
import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.BeforeEach;

/**
 * Базовый класс для тестов на Playwright.
 * Управляет жизненным циклом: Playwright → Browser → Context → Page.
 */
public abstract class BaseTest {

    // === Общие на весь класс ===
    protected static Playwright playwright;
    protected static Browser browser;

    // === Для каждого теста ===
    protected BrowserContext context;
    protected Page page;

    // Базовый URL тестируемого приложения
    protected static final String BASE_URL = "https://www.saucedemo.com";

    @BeforeAll
    static void launchBrowser() {
        // Создать экземпляр Playwright
        playwright = Playwright.create();

        // Запустить Chromium
        browser = playwright.chromium().launch(
                new BrowserType.LaunchOptions()
                        .setHeadless(false)   // true — без GUI (для CI)
                        .setSlowMo(0)         // Замедление между действиями (мс), полезно для отладки
        );
    }

    @BeforeEach
    void createContextAndPage() {
        // Новый изолированный контекст для каждого теста
        context = browser.newContext(
                new Browser.NewContextOptions()
                        .setViewportSize(1920, 1080)
        );

        // Включить трассировку (опционально)
        context.tracing().start(
                new com.microsoft.playwright.Tracing.StartOptions()
                        .setScreenshots(true)
                        .setSnapshots(true)
                        .setSources(false)
        );

        // Новая вкладка
        page = context.newPage();
    }

    @AfterEach
    void closeContext() {
        // Сохранить trace при необходимости (раскомментируйте)
        // context.tracing().stop(
        //     new com.microsoft.playwright.Tracing.StopOptions()
        //         .setPath(Paths.get("target/traces/trace-" +
        //             System.currentTimeMillis() + ".zip"))
        // );

        // Закрыть контекст (все страницы внутри закроются автоматически)
        context.close();
    }

    @AfterAll
    static void closeBrowser() {
        browser.close();
        playwright.close();
    }
}
```

### Диаграмма жизненного цикла

```
@BeforeAll:  Playwright.create() → browser = chromium.launch()
     ↓
@BeforeEach: context = browser.newContext() → page = context.newPage()
     ↓
@Test:       page.navigate(...) → locator.click() → expect(...)
     ↓
@AfterEach:  context.close()
     ↓
@AfterAll:   browser.close() → playwright.close()
```

---

## Шаг 7 — Locators: getByRole, getByText, getByTestId

Playwright продвигает семантические локаторы, устойчивые к изменениям вёрстки.

### Типы локаторов (по приоритету)

| Приоритет | Метод | Когда использовать |
|:---------:|-------|-------------------|
| 1 | `getByRole()` | Всегда, когда элемент имеет ARIA-роль |
| 2 | `getByLabel()` | Для input-полей с `<label>` |
| 3 | `getByPlaceholder()` | Для input-полей с placeholder |
| 4 | `getByText()` | Для элементов с уникальным текстом |
| 5 | `getByTestId()` | Для элементов с `data-test` атрибутом |
| 6 | `locator()` | CSS/XPath — крайний случай |

### Примеры для SauceDemo

```java
// Кнопка Login по роли и тексту
page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Login"));

// Поле ввода по placeholder
page.getByPlaceholder("Username");
page.getByPlaceholder("Password");

// Элемент по data-test атрибуту
page.getByTestId("error");

// Текст на странице
page.getByText("Products");

// CSS-селектор (fallback)
page.locator("#login-button");
page.locator(".inventory_item");
```

> **Настройка testId-атрибута.** По умолчанию `getByTestId` ищет по `data-testid`. Для SauceDemo нужно настроить `data-test`:
>
> ```java
> playwright.selectors().setTestIdAttribute("data-test");
> ```

---

## Шаг 8 — Assertions: expect

Playwright assertions автоматически ждут выполнения условия (по умолчанию 5 секунд).

### Основные assertions

```java
import static com.microsoft.playwright.assertions.PlaywrightAssertions.assertThat;

// Элемент видим
assertThat(page.locator(".title")).isVisible();

// Элемент содержит текст
assertThat(page.locator(".title")).hasText("Products");

// Элемент содержит текст (частичное совпадение)
assertThat(page.locator(".title")).containsText("Prod");

// Количество элементов
assertThat(page.locator(".inventory_item")).hasCount(6);

// URL страницы
assertThat(page).hasURL("https://www.saucedemo.com/inventory.html");

// Заголовок страницы
assertThat(page).hasTitle("Swag Labs");

// Элемент имеет атрибут
assertThat(page.locator("#login-button")).hasAttribute("type", "submit");

// Элемент включён / выключен
assertThat(page.locator("#login-button")).isEnabled();
assertThat(page.locator("#submit")).isDisabled();

// Input имеет значение
assertThat(page.locator("#user-name")).hasValue("standard_user");

// Элемент не видим
assertThat(page.locator(".error")).not().isVisible();
```

### Отличие от JUnit assertions

```java
// JUnit — проверяет СЕЙЧАС, не ждёт
assertEquals("Products", element.textContent()); // может упасть, если страница загружается

// Playwright — ждёт до timeout
assertThat(page.locator(".title")).hasText("Products"); // подождёт появления текста
```

---

## Шаг 9 — LoginPage с Playwright-локаторами

Файл: `src/test/java/com/saucedemo/pages/LoginPage.java`

```java
package com.saucedemo.pages;

import com.microsoft.playwright.Locator;
import com.microsoft.playwright.Page;
import com.microsoft.playwright.options.AriaRole;

/**
 * Page Object для страницы логина SauceDemo.
 * Использует Playwright-локаторы: getByPlaceholder, getByRole, getByTestId.
 */
public class LoginPage {

    private final Page page;

    // === Локаторы ===
    private final Locator usernameField;
    private final Locator passwordField;
    private final Locator loginButton;
    private final Locator errorMessage;

    public LoginPage(Page page) {
        this.page = page;

        // Семантические локаторы — устойчивы к изменениям вёрстки
        this.usernameField = page.getByPlaceholder("Username");
        this.passwordField = page.getByPlaceholder("Password");
        this.loginButton = page.getByRole(
                AriaRole.BUTTON,
                new Page.GetByRoleOptions().setName("Login")
        );
        this.errorMessage = page.locator("[data-test='error']");
    }

    /** Открыть страницу логина */
    public LoginPage navigate(String baseUrl) {
        page.navigate(baseUrl);
        return this;
    }

    /** Ввести имя пользователя */
    public LoginPage enterUsername(String username) {
        usernameField.fill(username);
        return this;
    }

    /** Ввести пароль */
    public LoginPage enterPassword(String password) {
        passwordField.fill(password);
        return this;
    }

    /** Нажать кнопку Login */
    public void clickLogin() {
        loginButton.click();
    }

    /** Выполнить полный логин */
    public ProductsPage loginAs(String username, String password) {
        enterUsername(username);
        enterPassword(password);
        clickLogin();
        return new ProductsPage(page);
    }

    /** Получить локатор ошибки для assertions */
    public Locator getErrorMessage() {
        return errorMessage;
    }

    /** Получить объект Page (для assertions в тестах) */
    public Page getPage() {
        return page;
    }
}
```

> **`fill()` vs `type()`.** Метод `fill()` очищает поле и вставляет текст мгновенно. Метод `type()` (в Playwright — `pressSequentially()`) вводит символы по одному, имитируя нажатия клавиш. Для обычных форм используйте `fill()`.

---

## Шаг 10 — ProductsPage

Файл: `src/test/java/com/saucedemo/pages/ProductsPage.java`

```java
package com.saucedemo.pages;

import com.microsoft.playwright.Locator;
import com.microsoft.playwright.Page;

import java.util.List;

/**
 * Page Object для страницы каталога товаров SauceDemo.
 */
public class ProductsPage {

    private final Page page;

    // === Локаторы ===
    private final Locator pageTitle;
    private final Locator inventoryItems;
    private final Locator inventoryItemNames;
    private final Locator cartBadge;
    private final Locator cartLink;
    private final Locator sortDropdown;

    public ProductsPage(Page page) {
        this.page = page;
        this.pageTitle = page.locator(".title");
        this.inventoryItems = page.locator(".inventory_item");
        this.inventoryItemNames = page.locator(".inventory_item_name");
        this.cartBadge = page.locator(".shopping_cart_badge");
        this.cartLink = page.locator(".shopping_cart_link");
        this.sortDropdown = page.locator(".product_sort_container");
    }

    /** Локатор заголовка (для assertThat) */
    public Locator getPageTitle() {
        return pageTitle;
    }

    /** Локатор всех товаров (для assertThat(...).hasCount()) */
    public Locator getInventoryItems() {
        return inventoryItems;
    }

    /** Локатор названий товаров */
    public Locator getInventoryItemNames() {
        return inventoryItemNames;
    }

    /** Локатор значка корзины */
    public Locator getCartBadge() {
        return cartBadge;
    }

    /** Получить список названий всех товаров */
    public List<String> getProductNamesList() {
        return inventoryItemNames.allTextContents();
    }

    /** Добавить товар в корзину по названию */
    public ProductsPage addToCart(String productName) {
        String buttonId = "add-to-cart-" + productName
                .toLowerCase()
                .replace(" ", "-");
        page.locator("#" + buttonId).click();
        return this;
    }

    /** Удалить товар из корзины */
    public ProductsPage removeFromCart(String productName) {
        String buttonId = "remove-" + productName
                .toLowerCase()
                .replace(" ", "-");
        page.locator("#" + buttonId).click();
        return this;
    }

    /** Перейти в корзину */
    public void goToCart() {
        cartLink.click();
    }

    /** Выбрать сортировку */
    public ProductsPage sortBy(String value) {
        sortDropdown.selectOption(value);
        return this;
    }

    /** Получить объект Page */
    public Page getPage() {
        return page;
    }
}
```

---

## Шаг 11 — Тесты авторизации

Файл: `src/test/java/com/saucedemo/tests/LoginTest.java`

```java
package com.saucedemo.tests;

import com.saucedemo.base.BaseTest;
import com.saucedemo.pages.LoginPage;
import com.saucedemo.pages.ProductsPage;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import java.util.List;

import static com.microsoft.playwright.assertions.PlaywrightAssertions.assertThat;
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;

/**
 * Тесты авторизации SauceDemo на Playwright.
 */
class LoginTest extends BaseTest {

    private LoginPage loginPage;

    @BeforeEach
    void openLoginPage() {
        loginPage = new LoginPage(page);
        loginPage.navigate(BASE_URL);
    }

    @Test
    @DisplayName("Успешный логин со стандартным пользователем")
    void testSuccessfulLogin() {
        // Act
        ProductsPage productsPage = loginPage.loginAs("standard_user", "secret_sauce");

        // Assert — Playwright web-first assertions
        assertThat(productsPage.getPageTitle()).isVisible();
        assertThat(productsPage.getPageTitle()).hasText("Products");
        assertThat(page).hasURL(BASE_URL + "/inventory.html");
    }

    @Test
    @DisplayName("На странице отображаются 6 товаров")
    void testProductsCount() {
        ProductsPage productsPage = loginPage.loginAs("standard_user", "secret_sauce");

        assertThat(productsPage.getInventoryItems()).hasCount(6);
    }

    @Test
    @DisplayName("Логин заблокированного пользователя")
    void testLockedOutUser() {
        loginPage.enterUsername("locked_out_user");
        loginPage.enterPassword("secret_sauce");
        loginPage.clickLogin();

        assertThat(loginPage.getErrorMessage()).isVisible();
        assertThat(loginPage.getErrorMessage())
                .containsText("Sorry, this user has been locked out.");
    }

    @Test
    @DisplayName("Логин с пустым именем пользователя")
    void testEmptyUsername() {
        loginPage.enterPassword("secret_sauce");
        loginPage.clickLogin();

        assertThat(loginPage.getErrorMessage()).isVisible();
        assertThat(loginPage.getErrorMessage())
                .containsText("Username is required");
    }

    @Test
    @DisplayName("Логин с пустым паролем")
    void testEmptyPassword() {
        loginPage.enterUsername("standard_user");
        loginPage.clickLogin();

        assertThat(loginPage.getErrorMessage())
                .containsText("Password is required");
    }

    @Test
    @DisplayName("Логин с неверными учётными данными")
    void testInvalidCredentials() {
        loginPage.enterUsername("invalid");
        loginPage.enterPassword("invalid");
        loginPage.clickLogin();

        assertThat(loginPage.getErrorMessage())
                .containsText("do not match");
    }

    @Test
    @DisplayName("Добавление товара в корзину")
    void testAddToCart() {
        ProductsPage productsPage = loginPage.loginAs("standard_user", "secret_sauce");
        productsPage.addToCart("sauce-labs-backpack");

        assertThat(productsPage.getCartBadge()).isVisible();
        assertThat(productsPage.getCartBadge()).hasText("1");
    }

    @Test
    @DisplayName("Проверка списка названий товаров")
    void testProductNames() {
        ProductsPage productsPage = loginPage.loginAs("standard_user", "secret_sauce");
        List<String> names = productsPage.getProductNamesList();

        assertEquals(6, names.size(), "Должно быть 6 товаров");
        assertTrue(names.contains("Sauce Labs Backpack"),
                "Список должен содержать 'Sauce Labs Backpack'");
    }
}
```

---

## Шаг 12 — Скриншоты и трассировка

### Скриншот страницы

```java
// Скриншот всей страницы
page.screenshot(new Page.ScreenshotOptions()
        .setPath(Paths.get("target/screenshots/page.png"))
        .setFullPage(true));

// Скриншот конкретного элемента
page.locator(".inventory_list").screenshot(
        new Locator.ScreenshotOptions()
                .setPath(Paths.get("target/screenshots/products.png")));
```

### Скриншот при падении теста

Добавьте в `BaseTest` метод с JUnit 5 `TestWatcher`:

```java
import org.junit.jupiter.api.extension.ExtensionContext;
import org.junit.jupiter.api.extension.RegisterExtension;
import org.junit.jupiter.api.extension.TestWatcher;

// Внутри BaseTest:
@RegisterExtension
TestWatcher screenshotOnFailure = new TestWatcher() {
    @Override
    public void testFailed(ExtensionContext context, Throwable cause) {
        String testName = context.getDisplayName().replaceAll("[^a-zA-Z0-9]", "_");
        page.screenshot(new Page.ScreenshotOptions()
                .setPath(Paths.get("target/screenshots/" + testName + ".png"))
                .setFullPage(true));
    }
};
```

### Трассировка (Trace)

Трассировка уже включена в `BaseTest`. Чтобы сохранять trace-файлы, раскомментируйте блок в `@AfterEach`.

Просмотр trace:

```bash
# Открыть trace viewer в браузере
mvn exec:java -e \
  -D exec.mainClass=com.microsoft.playwright.CLI \
  -D exec.args="show-trace target/traces/trace-12345.zip"
```

Trace viewer показывает:
- Все действия пошагово (клики, ввод, навигация)
- Скриншот до и после каждого действия
- Снимок DOM
- Сетевые запросы
- Логи консоли

> **Trace viewer — убийственная фича Playwright.** Один zip-файл содержит всё для воспроизведения и отладки проблемы. Его можно передать разработчику, прикрепить к багу в Jira.

---

## Шаг 13 — Запуск и отладка

### Запуск тестов

```bash
# Все тесты
mvn clean test

# Конкретный класс
mvn clean test -Dtest=LoginTest

# Конкретный метод
mvn clean test -Dtest=LoginTest#testSuccessfulLogin
```

### Headless-режим (для CI/CD)

В `BaseTest` измените:

```java
browser = playwright.chromium().launch(
        new BrowserType.LaunchOptions()
                .setHeadless(true)   // Без GUI
);
```

### Замедление для отладки

```java
browser = playwright.chromium().launch(
        new BrowserType.LaunchOptions()
                .setHeadless(false)
                .setSlowMo(500)   // 500мс пауза между действиями
);
```

### Debug-режим с паузой

Добавьте в тест:

```java
page.pause(); // Откроет Playwright Inspector — интерактивную отладку
```

Playwright Inspector позволяет:
- Пошагово выполнять действия
- Выбирать элементы и генерировать локаторы
- Просматривать DOM

---

## Playwright vs Selenium: сравнение

| Аспект | Selenium | Playwright |
|--------|----------|------------|
| Архитектура | WebDriver-протокол | CDP/собственный протокол |
| Ожидания | Ручные (explicit/implicit wait) | Автоматические (auto-wait) |
| Локаторы | `By.id`, `By.css`, `By.xpath` | `getByRole`, `getByText`, `getByTestId` |
| Assertions | JUnit/TestNG | Встроенные web-first assertions |
| Отладка | Логи, скриншоты | Trace viewer, Inspector, `page.pause()` |
| Браузеры | Chrome, Firefox, Edge, Safari | Chromium, Firefox, WebKit |
| Скорость | Через HTTP-протокол (медленнее) | Прямое подключение (быстрее) |
| Изоляция тестов | Ручная (cookies, cache) | BrowserContext из коробки |
| Драйвер | Нужен (ChromeDriver) | Не нужен |
| Экосистема | Огромная, зрелая | Растущая, современная |

---

## Задания для самостоятельной работы

### Задание 1 — CartPage

Создайте `CartPage` с Playwright-локаторами. Используйте `getByRole` для кнопок, `getByText` для названий.

### Задание 2 — Полный checkout-поток

Напишите E2E-тест: логин → добавить товар → корзина → checkout → заполнить форму → finish. Проверьте текст "Thank you for your order!".

### Задание 3 — Запуск в разных браузерах

Параметризуйте тесты для запуска в Chromium, Firefox и WebKit:

```java
@ParameterizedTest
@ValueSource(strings = {"chromium", "firefox", "webkit"})
void testLoginInDifferentBrowsers(String browserName) {
    // Создайте browser по имени и выполните тест логина
}
```

### Задание 4 — Trace для упавшего теста

Настройте сохранение trace только для упавших тестов (используйте `TestWatcher`). Откройте trace в viewer и изучите содержимое.

### Задание 5 — Network interception

Перехватите API-запросы при логине:

```java
page.route("**/api/**", route -> {
    System.out.println("Перехвачен запрос: " + route.request().url());
    route.resume();
});
```

---

**Результат практики:** рабочий проект на Playwright for Java с современными локаторами, web-first assertions и возможностью трассировки. Запуск: `mvn clean test`. Отладка: добавить `page.pause()` и использовать Playwright Inspector.
