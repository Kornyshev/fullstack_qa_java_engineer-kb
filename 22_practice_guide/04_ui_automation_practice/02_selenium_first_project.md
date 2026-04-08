# Первый проект на Selenium + JUnit 5 + Page Object Model

## Содержание

1. [Цель практики](#цель-практики)
2. [Что мы создадим](#что-мы-создадим)
3. [Шаг 1 — Создание Maven-проекта](#шаг-1--создание-maven-проекта)
4. [Шаг 2 — Зависимости в pom.xml](#шаг-2--зависимости-в-pomxml)
5. [Шаг 3 — Структура проекта](#шаг-3--структура-проекта)
6. [Шаг 4 — Конфигурация из properties](#шаг-4--конфигурация-из-properties)
7. [Шаг 5 — BasePage: общие методы](#шаг-5--basepage-общие-методы)
8. [Шаг 6 — LoginPage](#шаг-6--loginpage)
9. [Шаг 7 — ProductsPage](#шаг-7--productspage)
10. [Шаг 8 — BaseTest с setup и teardown](#шаг-8--basetest-с-setup-и-teardown)
11. [Шаг 9 — Первый тест: логин](#шаг-9--первый-тест-логин)
12. [Шаг 10 — Запуск и анализ результатов](#шаг-10--запуск-и-анализ-результатов)
13. [Типичные ошибки](#типичные-ошибки)
14. [Задания для самостоятельной работы](#задания-для-самостоятельной-работы)

---

## Цель практики

Создать с нуля рабочий проект UI-автоматизации с использованием **Selenium WebDriver**, **JUnit 5** и паттерна **Page Object Model**. К концу практики у вас будет готовый проект, который можно запустить одной командой.

**Целевой сайт:** [https://www.saucedemo.com](https://www.saucedemo.com)

---

## Что мы создадим

| Компонент | Назначение |
|-----------|------------|
| `pom.xml` | Зависимости и настройка сборки |
| `ConfigReader` | Чтение параметров из `config.properties` |
| `BasePage` | Общие методы для всех Page Object |
| `LoginPage` | Page Object страницы логина |
| `ProductsPage` | Page Object страницы каталога товаров |
| `BaseTest` | Setup/teardown для WebDriver |
| `LoginTest` | Тесты авторизации |

---

## Шаг 1 — Создание Maven-проекта

Откройте терминал и выполните:

```bash
mvn archetype:generate \
  -DgroupId=com.saucedemo \
  -DartifactId=selenium-saucedemo \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DarchetypeVersion=1.4 \
  -DinteractiveMode=false
```

Или создайте проект через IntelliJ IDEA: **File → New → Project → Maven Archetype → quickstart**.

После создания удалите файлы `App.java` и `AppTest.java` — они нам не понадобятся.

---

## Шаг 2 — Зависимости в pom.xml

Замените содержимое `pom.xml` целиком:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.saucedemo</groupId>
    <artifactId>selenium-saucedemo</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <!-- Версии зависимостей -->
        <selenium.version>4.27.0</selenium.version>
        <webdrivermanager.version>5.9.2</webdrivermanager.version>
        <junit.version>5.11.3</junit.version>
        <maven-surefire.version>3.5.2</maven-surefire.version>
    </properties>

    <dependencies>
        <!-- Selenium WebDriver -->
        <dependency>
            <groupId>org.seleniumhq.selenium</groupId>
            <artifactId>selenium-java</artifactId>
            <version>${selenium.version}</version>
        </dependency>

        <!-- WebDriverManager — автоматическое управление драйверами -->
        <dependency>
            <groupId>io.github.bonigarcia</groupId>
            <artifactId>webdrivermanager</artifactId>
            <version>${webdrivermanager.version}</version>
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
            <!-- Плагин для запуска тестов -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>${maven-surefire.version}</version>
            </plugin>
        </plugins>
    </build>
</project>
```

После сохранения выполните:

```bash
mvn clean install -DskipTests
```

> **Зачем WebDriverManager?** Он автоматически скачивает нужную версию драйвера для вашего браузера. Без него пришлось бы вручную скачивать `chromedriver` и следить за совместимостью версий.

---

## Шаг 3 — Структура проекта

Создайте следующую структуру каталогов:

```
selenium-saucedemo/
├── pom.xml
└── src/
    ├── main/
    │   └── java/
    │       └── com/saucedemo/
    │           ├── config/
    │           │   └── ConfigReader.java
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
            └── config.properties
```

---

## Шаг 4 — Конфигурация из properties

### config.properties

Файл: `src/test/resources/config.properties`

```properties
# Браузер: chrome, firefox, edge
browser=chrome
# Базовый URL тестируемого приложения
base.url=https://www.saucedemo.com
# Таймаут ожидания элементов (секунды)
timeout=10
# Учётные данные
username=standard_user
password=secret_sauce
```

### ConfigReader.java

Файл: `src/main/java/com/saucedemo/config/ConfigReader.java`

```java
package com.saucedemo.config;

import java.io.IOException;
import java.io.InputStream;
import java.util.Properties;

/**
 * Читает настройки из config.properties.
 * Используем Singleton — файл загружается один раз.
 */
public class ConfigReader {

    private static final Properties properties = new Properties();

    static {
        try (InputStream input = ConfigReader.class.getClassLoader()
                .getResourceAsStream("config.properties")) {
            if (input == null) {
                throw new RuntimeException("Файл config.properties не найден в classpath");
            }
            properties.load(input);
        } catch (IOException e) {
            throw new RuntimeException("Ошибка при чтении config.properties", e);
        }
    }

    /** Получить значение по ключу */
    public static String get(String key) {
        String value = properties.getProperty(key);
        if (value == null) {
            throw new RuntimeException("Ключ '" + key + "' не найден в config.properties");
        }
        return value;
    }

    /** Получить значение как int */
    public static int getInt(String key) {
        return Integer.parseInt(get(key));
    }
}
```

> **Почему properties, а не хардкод?** Чтобы менять браузер, URL или таймаут без перекомпиляции. В CI/CD можно подставлять разные файлы конфигурации для разных окружений.

---

## Шаг 5 — BasePage: общие методы

Файл: `src/main/java/com/saucedemo/pages/BasePage.java`

```java
package com.saucedemo.pages;

import com.saucedemo.config.ConfigReader;
import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.support.ui.ExpectedConditions;
import org.openqa.selenium.support.ui.WebDriverWait;

import java.time.Duration;

/**
 * Базовый класс для всех Page Object.
 * Содержит общие методы работы с элементами.
 */
public abstract class BasePage {

    protected WebDriver driver;
    protected WebDriverWait wait;

    public BasePage(WebDriver driver) {
        this.driver = driver;
        int timeout = ConfigReader.getInt("timeout");
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(timeout));
    }

    /** Найти элемент с ожиданием видимости */
    protected WebElement findElement(By locator) {
        return wait.until(ExpectedConditions.visibilityOfElementLocated(locator));
    }

    /** Кликнуть по элементу с ожиданием кликабельности */
    protected void click(By locator) {
        wait.until(ExpectedConditions.elementToBeClickable(locator)).click();
    }

    /** Очистить поле и ввести текст */
    protected void type(By locator, String text) {
        WebElement element = findElement(locator);
        element.clear();
        element.sendKeys(text);
    }

    /** Получить текст элемента */
    protected String getText(By locator) {
        return findElement(locator).getText();
    }

    /** Проверить, отображается ли элемент */
    protected boolean isDisplayed(By locator) {
        try {
            return findElement(locator).isDisplayed();
        } catch (Exception e) {
            return false;
        }
    }

    /** Получить текущий URL */
    protected String getCurrentUrl() {
        return driver.getCurrentUrl();
    }
}
```

### Что делает каждый метод

| Метод | Назначение | Почему с ожиданием |
|-------|------------|--------------------|
| `findElement` | Найти элемент | Страница может ещё загружаться |
| `click` | Клик | Элемент может быть перекрыт или не прогружен |
| `type` | Ввод текста | Поле ввода может появиться с задержкой |
| `getText` | Получить текст | Текст может подгружаться динамически |
| `isDisplayed` | Проверить видимость | Безопасная проверка без исключений |

---

## Шаг 6 — LoginPage

Файл: `src/main/java/com/saucedemo/pages/LoginPage.java`

```java
package com.saucedemo.pages;

import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;

/**
 * Page Object для страницы авторизации SauceDemo.
 * URL: https://www.saucedemo.com
 */
public class LoginPage extends BasePage {

    // === Локаторы ===
    private final By usernameField = By.id("user-name");
    private final By passwordField = By.id("password");
    private final By loginButton = By.id("login-button");
    private final By errorMessage = By.cssSelector("[data-test='error']");

    public LoginPage(WebDriver driver) {
        super(driver);
    }

    /** Открыть страницу логина */
    public LoginPage open(String baseUrl) {
        driver.get(baseUrl);
        return this;
    }

    /** Ввести имя пользователя */
    public LoginPage enterUsername(String username) {
        type(usernameField, username);
        return this;
    }

    /** Ввести пароль */
    public LoginPage enterPassword(String password) {
        type(passwordField, password);
        return this;
    }

    /** Нажать кнопку Login */
    public LoginPage clickLogin() {
        click(loginButton);
        return this;
    }

    /**
     * Выполнить полный логин (заполнить поля и нажать Login).
     * Возвращает ProductsPage, т.к. при успешном логине
     * происходит переход на страницу товаров.
     */
    public ProductsPage loginAs(String username, String password) {
        enterUsername(username);
        enterPassword(password);
        clickLogin();
        return new ProductsPage(driver);
    }

    /** Получить текст сообщения об ошибке */
    public String getErrorMessage() {
        return getText(errorMessage);
    }

    /** Проверить, отображается ли ошибка */
    public boolean isErrorDisplayed() {
        return isDisplayed(errorMessage);
    }
}
```

> **Паттерн Fluent Interface.** Методы возвращают `this` (или следующую страницу), что позволяет писать цепочки вызовов: `loginPage.enterUsername("user").enterPassword("pass").clickLogin()`.

---

## Шаг 7 — ProductsPage

Файл: `src/main/java/com/saucedemo/pages/ProductsPage.java`

```java
package com.saucedemo.pages;

import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;

import java.util.List;
import java.util.stream.Collectors;

/**
 * Page Object для страницы каталога товаров.
 * URL: https://www.saucedemo.com/inventory.html
 */
public class ProductsPage extends BasePage {

    // === Локаторы ===
    private final By pageTitle = By.className("title");
    private final By inventoryItems = By.className("inventory_item");
    private final By inventoryItemNames = By.className("inventory_item_name");
    private final By shoppingCartBadge = By.className("shopping_cart_badge");
    private final By shoppingCartLink = By.className("shopping_cart_link");
    private final By sortDropdown = By.className("product_sort_container");

    public ProductsPage(WebDriver driver) {
        super(driver);
    }

    /** Получить заголовок страницы */
    public String getTitle() {
        return getText(pageTitle);
    }

    /** Проверить, что страница Products загружена */
    public boolean isPageOpened() {
        return isDisplayed(pageTitle) && getTitle().equals("Products");
    }

    /** Получить количество товаров на странице */
    public int getProductCount() {
        return driver.findElements(inventoryItems).size();
    }

    /** Получить список названий всех товаров */
    public List<String> getProductNames() {
        return driver.findElements(inventoryItemNames)
                .stream()
                .map(WebElement::getText)
                .collect(Collectors.toList());
    }

    /**
     * Добавить товар в корзину по названию.
     * Кнопка формируется по шаблону: add-to-cart-{название-в-нижнем-регистре-через-дефис}
     */
    public ProductsPage addToCart(String productName) {
        String buttonId = "add-to-cart-" + productName
                .toLowerCase()
                .replace(" ", "-");
        click(By.id(buttonId));
        return this;
    }

    /** Получить число товаров в значке корзины */
    public int getCartBadgeCount() {
        try {
            return Integer.parseInt(getText(shoppingCartBadge));
        } catch (Exception e) {
            // Если значок не отображается — корзина пуста
            return 0;
        }
    }
}
```

---

## Шаг 8 — BaseTest с setup и teardown

Файл: `src/test/java/com/saucedemo/tests/BaseTest.java`

```java
package com.saucedemo.tests;

import com.saucedemo.config.ConfigReader;
import io.github.bonigarcia.wdm.WebDriverManager;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;
import org.openqa.selenium.edge.EdgeDriver;
import org.openqa.selenium.firefox.FirefoxDriver;

import java.time.Duration;

/**
 * Базовый класс для всех тестов.
 * Управляет жизненным циклом WebDriver.
 */
public abstract class BaseTest {

    protected WebDriver driver;

    @BeforeEach
    void setUp() {
        String browser = ConfigReader.get("browser").toLowerCase();

        // Инициализация WebDriver в зависимости от настройки
        switch (browser) {
            case "chrome" -> {
                WebDriverManager.chromedriver().setup();
                ChromeOptions options = new ChromeOptions();
                // Раскомментируйте для headless-режима (CI/CD)
                // options.addArguments("--headless=new");
                options.addArguments("--no-sandbox");
                options.addArguments("--disable-dev-shm-usage");
                driver = new ChromeDriver(options);
            }
            case "firefox" -> {
                WebDriverManager.firefoxdriver().setup();
                driver = new FirefoxDriver();
            }
            case "edge" -> {
                WebDriverManager.edgedriver().setup();
                driver = new EdgeDriver();
            }
            default -> throw new IllegalArgumentException(
                    "Неизвестный браузер: " + browser + ". Допустимые: chrome, firefox, edge"
            );
        }

        // Развернуть окно на весь экран
        driver.manage().window().maximize();

        // Неявное ожидание (implicit wait) — страховка
        int timeout = ConfigReader.getInt("timeout");
        driver.manage().timeouts().implicitlyWait(Duration.ofSeconds(timeout));
    }

    @AfterEach
    void tearDown() {
        if (driver != null) {
            driver.quit();
        }
    }
}
```

### Жизненный цикл теста

```
@BeforeEach: создать WebDriver → настроить → развернуть окно
     ↓
@Test: выполнить тестовый сценарий
     ↓
@AfterEach: закрыть браузер (driver.quit)
```

> **`driver.quit()` vs `driver.close()`.** Метод `quit()` закрывает все окна и завершает процесс драйвера. Метод `close()` закрывает только текущее окно. В `tearDown` всегда используйте `quit()`.

---

## Шаг 9 — Первый тест: логин

Файл: `src/test/java/com/saucedemo/tests/LoginTest.java`

```java
package com.saucedemo.tests;

import com.saucedemo.config.ConfigReader;
import com.saucedemo.pages.LoginPage;
import com.saucedemo.pages.ProductsPage;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Тесты авторизации на SauceDemo.
 */
class LoginTest extends BaseTest {

    private LoginPage loginPage;
    private final String baseUrl = ConfigReader.get("base.url");

    @BeforeEach
    void openLoginPage() {
        // Вызываем super.setUp() автоматически через @BeforeEach в BaseTest
        loginPage = new LoginPage(driver);
        loginPage.open(baseUrl);
    }

    @Test
    @DisplayName("Успешный логин со стандартным пользователем")
    void testSuccessfulLogin() {
        // Arrange: данные из конфига
        String username = ConfigReader.get("username");
        String password = ConfigReader.get("password");

        // Act: выполнить логин
        ProductsPage productsPage = loginPage.loginAs(username, password);

        // Assert: проверить переход на страницу Products
        assertTrue(productsPage.isPageOpened(),
                "Страница Products должна быть открыта после успешного логина");
        assertEquals("Products", productsPage.getTitle(),
                "Заголовок страницы должен быть 'Products'");
    }

    @Test
    @DisplayName("Логин с заблокированным пользователем")
    void testLockedOutUserLogin() {
        // Act
        loginPage.enterUsername("locked_out_user");
        loginPage.enterPassword("secret_sauce");
        loginPage.clickLogin();

        // Assert
        assertTrue(loginPage.isErrorDisplayed(),
                "Должно отображаться сообщение об ошибке");
        assertEquals(
                "Epic sadface: Sorry, this user has been locked out.",
                loginPage.getErrorMessage(),
                "Текст ошибки для заблокированного пользователя"
        );
    }

    @Test
    @DisplayName("Логин с пустым именем пользователя")
    void testEmptyUsername() {
        // Act
        loginPage.enterPassword("secret_sauce");
        loginPage.clickLogin();

        // Assert
        assertTrue(loginPage.isErrorDisplayed());
        assertEquals(
                "Epic sadface: Username is required",
                loginPage.getErrorMessage()
        );
    }

    @Test
    @DisplayName("Логин с пустым паролем")
    void testEmptyPassword() {
        // Act
        loginPage.enterUsername("standard_user");
        loginPage.clickLogin();

        // Assert
        assertTrue(loginPage.isErrorDisplayed());
        assertEquals(
                "Epic sadface: Password is required",
                loginPage.getErrorMessage()
        );
    }

    @Test
    @DisplayName("Логин с неверными учётными данными")
    void testInvalidCredentials() {
        // Act
        loginPage.enterUsername("invalid_user");
        loginPage.enterPassword("wrong_password");
        loginPage.clickLogin();

        // Assert
        assertTrue(loginPage.isErrorDisplayed());
        assertTrue(
                loginPage.getErrorMessage().contains("do not match"),
                "Сообщение должно содержать 'do not match'"
        );
    }

    @Test
    @DisplayName("После логина отображаются товары")
    void testProductsDisplayedAfterLogin() {
        // Act
        ProductsPage productsPage = loginPage.loginAs(
                ConfigReader.get("username"),
                ConfigReader.get("password")
        );

        // Assert
        assertTrue(productsPage.getProductCount() > 0,
                "На странице должен быть хотя бы один товар");
        assertEquals(6, productsPage.getProductCount(),
                "На SauceDemo должно быть 6 товаров");
    }
}
```

### Структура каждого теста: AAA

```
Arrange  — подготовка данных и начального состояния
Act      — выполнение тестируемого действия
Assert   — проверка результата
```

---

## Шаг 10 — Запуск и анализ результатов

### Запуск из терминала

```bash
# Запустить все тесты
mvn clean test

# Запустить конкретный тестовый класс
mvn clean test -Dtest=LoginTest

# Запустить конкретный тестовый метод
mvn clean test -Dtest=LoginTest#testSuccessfulLogin
```

### Запуск из IntelliJ IDEA

1. Откройте файл `LoginTest.java`
2. Нажмите зелёный треугольник рядом с классом (запуск всех тестов) или рядом с конкретным методом
3. Результаты отобразятся во вкладке **Run**

### Чтение результатов

Успешный прогон в терминале выглядит так:

```
[INFO] Tests run: 6, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

Если тест упал:

```
[ERROR] testLockedOutUserLogin  Time elapsed: 2.3 s  <<< FAILURE!
org.opentest4j.AssertionFailedError:
Expected: "Epic sadface: Sorry, this user has been locked out."
Actual:   "Epic sadface: Username is required"
```

> **Совет.** Всегда добавляйте осмысленное сообщение в assert: `assertTrue(condition, "Что именно пошло не так")`. Это экономит время при отладке.

---

## Типичные ошибки

| Проблема | Причина | Решение |
|----------|---------|---------|
| `SessionNotCreatedException` | Несовпадение версий Chrome и ChromeDriver | Обновите Chrome или используйте `WebDriverManager` |
| `NoSuchElementException` | Элемент ещё не загружен | Используйте explicit wait в BasePage |
| `StaleElementReferenceException` | Элемент пересоздан в DOM после действия | Найдите элемент заново перед обращением |
| `ElementClickInterceptedException` | Элемент перекрыт другим элементом | Прокрутите к элементу или подождите исчезновения перекрытия |
| Тесты проходят локально, падают в CI | В CI нет GUI-экрана | Включите `--headless=new` в ChromeOptions |

---

## Задания для самостоятельной работы

### Задание 1 — CartPage

Создайте `CartPage` (Page Object для страницы корзины):
- Локаторы: список товаров, кнопка Continue Shopping, кнопка Checkout
- Методы: `getCartItemNames()`, `removeItem(String name)`, `clickCheckout()`

### Задание 2 — Тест добавления в корзину

Напишите тест: логин → добавить товар → проверить badge → перейти в корзину → проверить товар в корзине.

### Задание 3 — Параметризованный тест

Используйте `@ParameterizedTest` и `@CsvSource` для проверки логина с разными пользователями:

```java
@ParameterizedTest(name = "Логин: {0}")
@CsvSource({
    "standard_user, secret_sauce, true",
    "locked_out_user, secret_sauce, false",
    "invalid_user, wrong_pass, false"
})
void testLoginVariants(String user, String pass, boolean shouldSucceed) {
    // Реализуйте самостоятельно
}
```

### Задание 4 — Headless-режим

Добавьте в `config.properties` параметр `headless=true` и обработайте его в `BaseTest`.

---

**Результат практики:** готовый Maven-проект с Selenium + JUnit 5, Page Object Model, конфигурацией из properties-файла и набором тестов логина для SauceDemo. Проект запускается командой `mvn clean test`.
