# Полный мини-проект: автоматизация SauceDemo

## Содержание

1. [Описание проекта](#описание-проекта)
2. [Структура проекта](#структура-проекта)
3. [pom.xml — полная конфигурация](#pomxml--полная-конфигурация)
4. [Конфигурация проекта](#конфигурация-проекта)
5. [Утилиты](#утилиты)
6. [BasePage — фундамент Page Object](#basepage--фундамент-page-object)
7. [LoginPage](#loginpage)
8. [ProductsPage](#productspage)
9. [CartPage](#cartpage)
10. [CheckoutStepOnePage](#checkoutsteponepage)
11. [CheckoutStepTwoPage](#checkoutsteptwopage)
12. [CheckoutCompletePage](#checkoutcompletepage)
13. [BaseTest — setup и teardown](#basetest--setup-и-teardown)
14. [LoginTest — авторизация](#logintest--авторизация)
15. [CartTest — корзина](#carttest--корзина)
16. [SortingTest — сортировка](#sortingtest--сортировка)
17. [CheckoutTest — полный E2E-поток](#checkouttest--полный-e2e-поток)
18. [Allure-интеграция](#allure-интеграция)
19. [Запуск и отчёт](#запуск-и-отчёт)
20. [README-шаблон для проекта](#readme-шаблон-для-проекта)

---

## Описание проекта

Этот раздел содержит полный, готовый к запуску проект автоматизации тестирования сайта [SauceDemo](https://www.saucedemo.com). Весь код можно скопировать, создать Maven-проект и запустить.

### Покрытие тестами

| Область | Сценарии |
|---------|----------|
| Авторизация | Успешный логин, заблокированный пользователь, пустые поля, неверные данные |
| Корзина | Добавление товара, удаление товара, проверка badge |
| Сортировка | По имени A-Z/Z-A, по цене low-high/high-low |
| Checkout | Полный E2E-поток от логина до подтверждения заказа |

### Технологический стек

| Компонент | Технология |
|-----------|------------|
| Язык | Java 17 |
| UI-фреймворк | Selenide 7.x |
| Тест-фреймворк | JUnit 5 |
| Отчётность | Allure 2.x |
| Сборка | Maven |

---

## Структура проекта

```
saucedemo-automation/
├── pom.xml
├── README.md
└── src/
    ├── main/
    │   └── java/
    │       └── com/saucedemo/
    │           ├── config/
    │           │   └── ProjectConfig.java
    │           ├── pages/
    │           │   ├── BasePage.java
    │           │   ├── LoginPage.java
    │           │   ├── ProductsPage.java
    │           │   ├── CartPage.java
    │           │   ├── CheckoutStepOnePage.java
    │           │   ├── CheckoutStepTwoPage.java
    │           │   └── CheckoutCompletePage.java
    │           └── utils/
    │               └── AllureUtils.java
    └── test/
        ├── java/
        │   └── com/saucedemo/
        │       └── tests/
        │           ├── BaseTest.java
        │           ├── LoginTest.java
        │           ├── CartTest.java
        │           ├── SortingTest.java
        │           └── CheckoutTest.java
        └── resources/
            ├── config.properties
            └── selenide.properties
```

---

## pom.xml — полная конфигурация

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.saucedemo</groupId>
    <artifactId>saucedemo-automation</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <!-- Версии зависимостей -->
        <selenide.version>7.5.1</selenide.version>
        <junit.version>5.11.3</junit.version>
        <allure.version>2.29.1</allure.version>
        <aspectj.version>1.9.22.1</aspectj.version>
        <maven-surefire.version>3.5.2</maven-surefire.version>
    </properties>

    <dependencies>
        <!-- Selenide — обёртка над Selenium с автоматическими ожиданиями -->
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

        <!-- Allure JUnit 5 — интеграция с тест-фреймворком -->
        <dependency>
            <groupId>io.qameta.allure</groupId>
            <artifactId>allure-junit5</artifactId>
            <version>${allure.version}</version>
            <scope>test</scope>
        </dependency>

        <!-- Allure Selenide — автоматические шаги и скриншоты -->
        <dependency>
            <groupId>io.qameta.allure</groupId>
            <artifactId>allure-selenide</artifactId>
            <version>${allure.version}</version>
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

---

## Конфигурация проекта

### config.properties

Файл: `src/test/resources/config.properties`

```properties
# === Учётные данные ===
standard.username=standard_user
standard.password=secret_sauce
locked.username=locked_out_user
locked.password=secret_sauce

# === Checkout: данные покупателя ===
checkout.firstName=John
checkout.lastName=Doe
checkout.postalCode=12345
```

### selenide.properties

Файл: `src/test/resources/selenide.properties`

```properties
selenide.browser=chrome
selenide.baseUrl=https://www.saucedemo.com
selenide.timeout=10000
selenide.browserSize=1920x1080
selenide.screenshots=true
selenide.savePageSource=false
selenide.headless=false
```

### ProjectConfig.java

Файл: `src/main/java/com/saucedemo/config/ProjectConfig.java`

```java
package com.saucedemo.config;

import java.io.IOException;
import java.io.InputStream;
import java.util.Properties;

/**
 * Чтение конфигурации проекта из config.properties.
 * Содержит учётные данные и параметры для тестов.
 */
public class ProjectConfig {

    private static final Properties props = new Properties();

    static {
        try (InputStream input = ProjectConfig.class.getClassLoader()
                .getResourceAsStream("config.properties")) {
            if (input == null) {
                throw new RuntimeException("config.properties не найден в classpath");
            }
            props.load(input);
        } catch (IOException e) {
            throw new RuntimeException("Ошибка чтения config.properties", e);
        }
    }

    // === Учётные данные ===
    public static String standardUsername() {
        return props.getProperty("standard.username");
    }

    public static String standardPassword() {
        return props.getProperty("standard.password");
    }

    public static String lockedUsername() {
        return props.getProperty("locked.username");
    }

    public static String lockedPassword() {
        return props.getProperty("locked.password");
    }

    // === Checkout ===
    public static String checkoutFirstName() {
        return props.getProperty("checkout.firstName");
    }

    public static String checkoutLastName() {
        return props.getProperty("checkout.lastName");
    }

    public static String checkoutPostalCode() {
        return props.getProperty("checkout.postalCode");
    }
}
```

---

## Утилиты

### AllureUtils.java

Файл: `src/main/java/com/saucedemo/utils/AllureUtils.java`

```java
package com.saucedemo.utils;

import com.codeborne.selenide.Selenide;
import io.qameta.allure.Attachment;
import org.openqa.selenium.OutputType;

/**
 * Вспомогательные методы для Allure-отчётов.
 */
public class AllureUtils {

    private AllureUtils() {
        // Утилитный класс — запрет создания экземпляра
    }

    /** Прикрепить скриншот к текущему шагу отчёта */
    @Attachment(value = "Скриншот", type = "image/png")
    public static byte[] takeScreenshot() {
        return Selenide.screenshot(OutputType.BYTES);
    }

    /** Прикрепить текст к текущему шагу отчёта */
    @Attachment(value = "{name}", type = "text/plain")
    public static String attachText(String name, String content) {
        return content;
    }

    /** Прикрепить HTML-код страницы */
    @Attachment(value = "Исходный код страницы", type = "text/html")
    public static String attachPageSource() {
        return Selenide.webdriver().driver().source();
    }
}
```

---

## BasePage — фундамент Page Object

Файл: `src/main/java/com/saucedemo/pages/BasePage.java`

```java
package com.saucedemo.pages;

import com.codeborne.selenide.Condition;
import com.codeborne.selenide.SelenideElement;

import static com.codeborne.selenide.Selenide.$;

/**
 * Базовый класс для всех Page Object.
 * Содержит общие элементы и методы навигации.
 */
public abstract class BasePage {

    // === Общие элементы (присутствуют на нескольких страницах) ===
    protected final SelenideElement pageTitle = $(".title");
    protected final SelenideElement cartLink = $(".shopping_cart_link");
    protected final SelenideElement cartBadge = $(".shopping_cart_badge");
    protected final SelenideElement burgerMenu = $("#react-burger-menu-btn");

    /** Проверить заголовок страницы */
    public SelenideElement getPageTitle() {
        return pageTitle;
    }

    /** Получить элемент badge корзины */
    public SelenideElement getCartBadge() {
        return cartBadge;
    }

    /** Перейти в корзину через иконку в шапке */
    public CartPage goToCart() {
        cartLink.click();
        return new CartPage();
    }

    /** Открыть бургер-меню */
    public void openMenu() {
        burgerMenu.click();
    }
}
```

---

## LoginPage

Файл: `src/main/java/com/saucedemo/pages/LoginPage.java`

```java
package com.saucedemo.pages;

import com.codeborne.selenide.SelenideElement;
import io.qameta.allure.Step;

import static com.codeborne.selenide.Selenide.$;
import static com.codeborne.selenide.Selenide.open;

/**
 * Page Object: страница авторизации.
 * URL: https://www.saucedemo.com
 */
public class LoginPage {

    private final SelenideElement usernameField = $("#user-name");
    private final SelenideElement passwordField = $("#password");
    private final SelenideElement loginButton = $("#login-button");
    private final SelenideElement errorMessage = $("[data-test='error']");

    @Step("Открыть страницу логина")
    public LoginPage openPage() {
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
        passwordField.setValue(password);
        return this;
    }

    @Step("Нажать кнопку Login")
    public void clickLogin() {
        loginButton.click();
    }

    @Step("Выполнить логин: {username}")
    public ProductsPage loginAs(String username, String password) {
        enterUsername(username);
        enterPassword(password);
        clickLogin();
        return new ProductsPage();
    }

    /** Сообщение об ошибке — для assertions */
    public SelenideElement getErrorMessage() {
        return errorMessage;
    }
}
```

---

## ProductsPage

Файл: `src/main/java/com/saucedemo/pages/ProductsPage.java`

```java
package com.saucedemo.pages;

import com.codeborne.selenide.ElementsCollection;
import com.codeborne.selenide.SelenideElement;
import io.qameta.allure.Step;

import java.util.List;

import static com.codeborne.selenide.Selenide.$;
import static com.codeborne.selenide.Selenide.$$;

/**
 * Page Object: страница каталога товаров.
 * URL: https://www.saucedemo.com/inventory.html
 */
public class ProductsPage extends BasePage {

    private final ElementsCollection inventoryItems = $$(".inventory_item");
    private final ElementsCollection itemNames = $$(".inventory_item_name");
    private final ElementsCollection itemPrices = $$(".inventory_item_price");
    private final SelenideElement sortDropdown = $(".product_sort_container");

    /** Коллекция всех карточек товаров */
    public ElementsCollection getInventoryItems() {
        return inventoryItems;
    }

    /** Коллекция названий товаров */
    public ElementsCollection getItemNames() {
        return itemNames;
    }

    /** Коллекция цен товаров */
    public ElementsCollection getItemPrices() {
        return itemPrices;
    }

    /** Получить список названий как строки */
    public List<String> getProductNamesList() {
        return itemNames.texts();
    }

    /** Получить список цен как числа */
    public List<Double> getProductPricesList() {
        return itemPrices.texts().stream()
                .map(price -> price.replace("$", ""))
                .map(Double::parseDouble)
                .toList();
    }

    @Step("Добавить в корзину: {productName}")
    public ProductsPage addToCart(String productName) {
        String buttonId = "add-to-cart-" + formatProductId(productName);
        $("#" + buttonId).click();
        return this;
    }

    @Step("Удалить из корзины: {productName}")
    public ProductsPage removeFromCart(String productName) {
        String buttonId = "remove-" + formatProductId(productName);
        $("#" + buttonId).click();
        return this;
    }

    @Step("Выбрать сортировку: {option}")
    public ProductsPage sortBy(String option) {
        sortDropdown.selectOption(option);
        return this;
    }

    /** Преобразовать название товара в формат id кнопки */
    private String formatProductId(String productName) {
        return productName.toLowerCase().replace(" ", "-");
    }
}
```

---

## CartPage

Файл: `src/main/java/com/saucedemo/pages/CartPage.java`

```java
package com.saucedemo.pages;

import com.codeborne.selenide.ElementsCollection;
import com.codeborne.selenide.SelenideElement;
import io.qameta.allure.Step;

import java.util.List;

import static com.codeborne.selenide.Selenide.$;
import static com.codeborne.selenide.Selenide.$$;

/**
 * Page Object: страница корзины.
 * URL: https://www.saucedemo.com/cart.html
 */
public class CartPage extends BasePage {

    private final ElementsCollection cartItems = $$(".cart_item");
    private final ElementsCollection cartItemNames = $$(".inventory_item_name");
    private final SelenideElement continueShoppingButton = $("#continue-shopping");
    private final SelenideElement checkoutButton = $("#checkout");

    /** Коллекция товаров в корзине */
    public ElementsCollection getCartItems() {
        return cartItems;
    }

    /** Коллекция названий товаров в корзине */
    public ElementsCollection getCartItemNames() {
        return cartItemNames;
    }

    /** Список названий товаров как строки */
    public List<String> getCartItemNamesList() {
        return cartItemNames.texts();
    }

    @Step("Удалить товар из корзины: {productName}")
    public CartPage removeItem(String productName) {
        String buttonId = "remove-" + productName
                .toLowerCase()
                .replace(" ", "-");
        $("#" + buttonId).click();
        return this;
    }

    @Step("Нажать Continue Shopping")
    public ProductsPage continueShopping() {
        continueShoppingButton.click();
        return new ProductsPage();
    }

    @Step("Нажать Checkout")
    public CheckoutStepOnePage checkout() {
        checkoutButton.click();
        return new CheckoutStepOnePage();
    }
}
```

---

## CheckoutStepOnePage

Файл: `src/main/java/com/saucedemo/pages/CheckoutStepOnePage.java`

```java
package com.saucedemo.pages;

import com.codeborne.selenide.SelenideElement;
import io.qameta.allure.Step;

import static com.codeborne.selenide.Selenide.$;

/**
 * Page Object: первый шаг checkout — ввод данных покупателя.
 * URL: https://www.saucedemo.com/checkout-step-one.html
 */
public class CheckoutStepOnePage extends BasePage {

    private final SelenideElement firstNameField = $("#first-name");
    private final SelenideElement lastNameField = $("#last-name");
    private final SelenideElement postalCodeField = $("#postal-code");
    private final SelenideElement continueButton = $("#continue");
    private final SelenideElement cancelButton = $("#cancel");
    private final SelenideElement errorMessage = $("[data-test='error']");

    @Step("Ввести имя: {firstName}")
    public CheckoutStepOnePage enterFirstName(String firstName) {
        firstNameField.setValue(firstName);
        return this;
    }

    @Step("Ввести фамилию: {lastName}")
    public CheckoutStepOnePage enterLastName(String lastName) {
        lastNameField.setValue(lastName);
        return this;
    }

    @Step("Ввести почтовый индекс: {postalCode}")
    public CheckoutStepOnePage enterPostalCode(String postalCode) {
        postalCodeField.setValue(postalCode);
        return this;
    }

    @Step("Заполнить форму покупателя: {firstName} {lastName}, {postalCode}")
    public CheckoutStepOnePage fillCustomerInfo(
            String firstName, String lastName, String postalCode) {
        enterFirstName(firstName);
        enterLastName(lastName);
        enterPostalCode(postalCode);
        return this;
    }

    @Step("Нажать Continue")
    public CheckoutStepTwoPage clickContinue() {
        continueButton.click();
        return new CheckoutStepTwoPage();
    }

    @Step("Нажать Cancel")
    public CartPage clickCancel() {
        cancelButton.click();
        return new CartPage();
    }

    /** Сообщение об ошибке валидации */
    public SelenideElement getErrorMessage() {
        return errorMessage;
    }
}
```

---

## CheckoutStepTwoPage

Файл: `src/main/java/com/saucedemo/pages/CheckoutStepTwoPage.java`

```java
package com.saucedemo.pages;

import com.codeborne.selenide.ElementsCollection;
import com.codeborne.selenide.SelenideElement;
import io.qameta.allure.Step;

import static com.codeborne.selenide.Selenide.$;
import static com.codeborne.selenide.Selenide.$$;

/**
 * Page Object: второй шаг checkout — обзор заказа.
 * URL: https://www.saucedemo.com/checkout-step-two.html
 */
public class CheckoutStepTwoPage extends BasePage {

    private final ElementsCollection cartItems = $$(".cart_item");
    private final SelenideElement subtotalLabel = $(".summary_subtotal_label");
    private final SelenideElement taxLabel = $(".summary_tax_label");
    private final SelenideElement totalLabel = $(".summary_info_label.summary_total_label");
    private final SelenideElement finishButton = $("#finish");
    private final SelenideElement cancelButton = $("#cancel");

    /** Коллекция товаров в обзоре заказа */
    public ElementsCollection getCartItems() {
        return cartItems;
    }

    /** Получить текст subtotal (например, "Item total: $29.99") */
    public SelenideElement getSubtotalLabel() {
        return subtotalLabel;
    }

    /** Получить текст налога */
    public SelenideElement getTaxLabel() {
        return taxLabel;
    }

    /** Получить текст итоговой суммы */
    public SelenideElement getTotalLabel() {
        return totalLabel;
    }

    /** Извлечь числовое значение subtotal */
    public double getSubtotalValue() {
        String text = subtotalLabel.getText();
        // Формат: "Item total: $29.99"
        return Double.parseDouble(text.replaceAll("[^0-9.]", ""));
    }

    @Step("Нажать Finish — подтвердить заказ")
    public CheckoutCompletePage clickFinish() {
        finishButton.click();
        return new CheckoutCompletePage();
    }

    @Step("Нажать Cancel")
    public ProductsPage clickCancel() {
        cancelButton.click();
        return new ProductsPage();
    }
}
```

---

## CheckoutCompletePage

Файл: `src/main/java/com/saucedemo/pages/CheckoutCompletePage.java`

```java
package com.saucedemo.pages;

import com.codeborne.selenide.SelenideElement;
import io.qameta.allure.Step;

import static com.codeborne.selenide.Selenide.$;

/**
 * Page Object: страница успешного оформления заказа.
 * URL: https://www.saucedemo.com/checkout-complete.html
 */
public class CheckoutCompletePage extends BasePage {

    private final SelenideElement completeHeader = $(".complete-header");
    private final SelenideElement completeText = $(".complete-text");
    private final SelenideElement backHomeButton = $("#back-to-products");
    private final SelenideElement ponyExpressImage = $(".pony_express");

    /** Заголовок "Thank you for your order!" */
    public SelenideElement getCompleteHeader() {
        return completeHeader;
    }

    /** Текст подтверждения */
    public SelenideElement getCompleteText() {
        return completeText;
    }

    /** Изображение pony express (визуальное подтверждение) */
    public SelenideElement getPonyExpressImage() {
        return ponyExpressImage;
    }

    @Step("Нажать Back Home — вернуться на главную")
    public ProductsPage clickBackHome() {
        backHomeButton.click();
        return new ProductsPage();
    }
}
```

---

## BaseTest — setup и teardown

Файл: `src/test/java/com/saucedemo/tests/BaseTest.java`

```java
package com.saucedemo.tests;

import com.codeborne.selenide.logevents.SelenideLogger;
import com.saucedemo.config.ProjectConfig;
import com.saucedemo.pages.LoginPage;
import com.saucedemo.pages.ProductsPage;
import com.saucedemo.utils.AllureUtils;
import io.qameta.allure.selenide.AllureSelenide;
import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.BeforeEach;

import static com.codeborne.selenide.Selenide.clearBrowserCookies;
import static com.codeborne.selenide.Selenide.clearBrowserLocalStorage;

/**
 * Базовый класс для всех тестовых классов.
 * Управляет Allure-интеграцией и очисткой состояния браузера.
 */
public abstract class BaseTest {

    @BeforeAll
    static void setupAllure() {
        // Подключить Allure listener для автоматического логирования
        SelenideLogger.addListener("allure", new AllureSelenide()
                .screenshots(true)
                .savePageSource(false)
        );
    }

    @BeforeEach
    void cleanBrowserState() {
        clearBrowserCookies();
        clearBrowserLocalStorage();
    }

    @AfterEach
    void screenshotIfNeeded() {
        // Selenide автоматически делает скриншот при падении,
        // но можно добавить дополнительную логику здесь
    }

    @AfterAll
    static void teardownAllure() {
        SelenideLogger.removeListener("allure");
    }

    /**
     * Вспомогательный метод: выполнить логин со стандартным пользователем.
     * Используется в тестах, где авторизация — предусловие, а не объект тестирования.
     */
    protected ProductsPage loginAsStandardUser() {
        LoginPage loginPage = new LoginPage();
        loginPage.openPage();
        return loginPage.loginAs(
                ProjectConfig.standardUsername(),
                ProjectConfig.standardPassword()
        );
    }
}
```

---

## LoginTest — авторизация

Файл: `src/test/java/com/saucedemo/tests/LoginTest.java`

```java
package com.saucedemo.tests;

import com.saucedemo.config.ProjectConfig;
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

@Epic("SauceDemo")
@Feature("Авторизация")
class LoginTest extends BaseTest {

    private LoginPage loginPage;

    @BeforeEach
    void openLogin() {
        loginPage = new LoginPage();
        loginPage.openPage();
    }

    @Test
    @DisplayName("Успешный логин со стандартным пользователем")
    @Severity(SeverityLevel.BLOCKER)
    void testSuccessfulLogin() {
        ProductsPage productsPage = loginPage.loginAs(
                ProjectConfig.standardUsername(),
                ProjectConfig.standardPassword()
        );

        productsPage.getPageTitle()
                .shouldBe(visible)
                .shouldHave(text("Products"));

        productsPage.getInventoryItems()
                .shouldHave(size(6));
    }

    @Test
    @DisplayName("Логин заблокированного пользователя")
    @Severity(SeverityLevel.CRITICAL)
    void testLockedOutUser() {
        loginPage.enterUsername(ProjectConfig.lockedUsername());
        loginPage.enterPassword(ProjectConfig.lockedPassword());
        loginPage.clickLogin();

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
}
```

---

## CartTest — корзина

Файл: `src/test/java/com/saucedemo/tests/CartTest.java`

```java
package com.saucedemo.tests;

import com.saucedemo.pages.CartPage;
import com.saucedemo.pages.ProductsPage;
import io.qameta.allure.Epic;
import io.qameta.allure.Feature;
import io.qameta.allure.Severity;
import io.qameta.allure.SeverityLevel;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import static com.codeborne.selenide.CollectionCondition.size;
import static com.codeborne.selenide.CollectionCondition.texts;
import static com.codeborne.selenide.Condition.*;

@Epic("SauceDemo")
@Feature("Корзина")
class CartTest extends BaseTest {

    private ProductsPage productsPage;

    @BeforeEach
    void login() {
        productsPage = loginAsStandardUser();
    }

    @Test
    @DisplayName("Добавление одного товара в корзину")
    @Severity(SeverityLevel.CRITICAL)
    void testAddSingleItem() {
        productsPage.addToCart("sauce-labs-backpack");

        productsPage.getCartBadge()
                .shouldBe(visible)
                .shouldHave(text("1"));
    }

    @Test
    @DisplayName("Добавление нескольких товаров в корзину")
    @Severity(SeverityLevel.CRITICAL)
    void testAddMultipleItems() {
        productsPage
                .addToCart("sauce-labs-backpack")
                .addToCart("sauce-labs-bike-light");

        productsPage.getCartBadge()
                .shouldHave(text("2"));
    }

    @Test
    @DisplayName("Удаление товара из корзины на странице каталога")
    @Severity(SeverityLevel.CRITICAL)
    void testRemoveItemFromProducts() {
        productsPage.addToCart("sauce-labs-backpack");
        productsPage.getCartBadge().shouldHave(text("1"));

        productsPage.removeFromCart("sauce-labs-backpack");

        // Badge должен исчезнуть — корзина пуста
        productsPage.getCartBadge().shouldNotBe(visible);
    }

    @Test
    @DisplayName("Товар отображается в корзине после добавления")
    @Severity(SeverityLevel.CRITICAL)
    void testItemAppearsInCart() {
        productsPage.addToCart("sauce-labs-backpack");
        CartPage cartPage = productsPage.goToCart();

        cartPage.getCartItems().shouldHave(size(1));
        cartPage.getCartItemNames().shouldHave(texts("Sauce Labs Backpack"));
    }

    @Test
    @DisplayName("Удаление товара на странице корзины")
    @Severity(SeverityLevel.NORMAL)
    void testRemoveItemFromCart() {
        productsPage.addToCart("sauce-labs-backpack");
        CartPage cartPage = productsPage.goToCart();

        cartPage.removeItem("sauce-labs-backpack");
        cartPage.getCartItems().shouldHave(size(0));
    }

    @Test
    @DisplayName("Continue Shopping возвращает на страницу каталога")
    @Severity(SeverityLevel.MINOR)
    void testContinueShopping() {
        CartPage cartPage = productsPage.goToCart();
        ProductsPage returned = cartPage.continueShopping();

        returned.getPageTitle()
                .shouldBe(visible)
                .shouldHave(text("Products"));
    }
}
```

---

## SortingTest — сортировка

Файл: `src/test/java/com/saucedemo/tests/SortingTest.java`

```java
package com.saucedemo.tests;

import com.saucedemo.pages.ProductsPage;
import io.qameta.allure.Epic;
import io.qameta.allure.Feature;
import io.qameta.allure.Severity;
import io.qameta.allure.SeverityLevel;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.Comparator;
import java.util.List;

import static org.junit.jupiter.api.Assertions.assertEquals;

@Epic("SauceDemo")
@Feature("Сортировка товаров")
class SortingTest extends BaseTest {

    private ProductsPage productsPage;

    @BeforeEach
    void login() {
        productsPage = loginAsStandardUser();
    }

    @Test
    @DisplayName("Сортировка по имени A-Z")
    @Severity(SeverityLevel.NORMAL)
    void testSortByNameAZ() {
        productsPage.sortBy("Name (A to Z)");

        List<String> actual = productsPage.getProductNamesList();
        List<String> expected = new ArrayList<>(actual);
        expected.sort(String::compareToIgnoreCase);

        assertEquals(expected, actual,
                "Товары должны быть отсортированы по имени A-Z");
    }

    @Test
    @DisplayName("Сортировка по имени Z-A")
    @Severity(SeverityLevel.NORMAL)
    void testSortByNameZA() {
        productsPage.sortBy("Name (Z to A)");

        List<String> actual = productsPage.getProductNamesList();
        List<String> expected = new ArrayList<>(actual);
        expected.sort(Comparator.comparing(String::toLowerCase).reversed());

        assertEquals(expected, actual,
                "Товары должны быть отсортированы по имени Z-A");
    }

    @Test
    @DisplayName("Сортировка по цене: от дешёвых к дорогим")
    @Severity(SeverityLevel.NORMAL)
    void testSortByPriceLowToHigh() {
        productsPage.sortBy("Price (low to high)");

        List<Double> actual = productsPage.getProductPricesList();
        List<Double> expected = new ArrayList<>(actual);
        expected.sort(Comparator.naturalOrder());

        assertEquals(expected, actual,
                "Товары должны быть отсортированы по цене (возрастание)");
    }

    @Test
    @DisplayName("Сортировка по цене: от дорогих к дешёвым")
    @Severity(SeverityLevel.NORMAL)
    void testSortByPriceHighToLow() {
        productsPage.sortBy("Price (high to low)");

        List<Double> actual = productsPage.getProductPricesList();
        List<Double> expected = new ArrayList<>(actual);
        expected.sort(Comparator.reverseOrder());

        assertEquals(expected, actual,
                "Товары должны быть отсортированы по цене (убывание)");
    }
}
```

---

## CheckoutTest — полный E2E-поток

Файл: `src/test/java/com/saucedemo/tests/CheckoutTest.java`

```java
package com.saucedemo.tests;

import com.saucedemo.config.ProjectConfig;
import com.saucedemo.pages.*;
import io.qameta.allure.Epic;
import io.qameta.allure.Feature;
import io.qameta.allure.Severity;
import io.qameta.allure.SeverityLevel;
import io.qameta.allure.Story;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import static com.codeborne.selenide.CollectionCondition.size;
import static com.codeborne.selenide.Condition.*;

@Epic("SauceDemo")
@Feature("Оформление заказа")
class CheckoutTest extends BaseTest {

    private ProductsPage productsPage;

    @BeforeEach
    void login() {
        productsPage = loginAsStandardUser();
    }

    @Test
    @Story("Полный поток покупки")
    @DisplayName("E2E: логин → добавить товар → checkout → finish")
    @Severity(SeverityLevel.BLOCKER)
    void testCompleteCheckoutFlow() {
        // Шаг 1: Добавить товар в корзину
        productsPage.addToCart("sauce-labs-backpack");
        productsPage.getCartBadge().shouldHave(text("1"));

        // Шаг 2: Перейти в корзину
        CartPage cartPage = productsPage.goToCart();
        cartPage.getCartItems().shouldHave(size(1));

        // Шаг 3: Начать checkout
        CheckoutStepOnePage stepOne = cartPage.checkout();
        stepOne.getPageTitle().shouldHave(text("Checkout: Your Information"));

        // Шаг 4: Заполнить данные покупателя
        CheckoutStepTwoPage stepTwo = stepOne
                .fillCustomerInfo(
                        ProjectConfig.checkoutFirstName(),
                        ProjectConfig.checkoutLastName(),
                        ProjectConfig.checkoutPostalCode()
                )
                .clickContinue();

        stepTwo.getPageTitle().shouldHave(text("Checkout: Overview"));
        stepTwo.getCartItems().shouldHave(size(1));

        // Шаг 5: Подтвердить заказ
        CheckoutCompletePage completePage = stepTwo.clickFinish();

        // Проверка финальной страницы
        completePage.getPageTitle().shouldHave(text("Checkout: Complete!"));
        completePage.getCompleteHeader()
                .shouldBe(visible)
                .shouldHave(text("Thank you for your order!"));
        completePage.getPonyExpressImage().shouldBe(visible);
    }

    @Test
    @Story("Покупка нескольких товаров")
    @DisplayName("E2E: добавить 3 товара → checkout → finish")
    @Severity(SeverityLevel.CRITICAL)
    void testCheckoutMultipleItems() {
        // Добавить 3 товара
        productsPage
                .addToCart("sauce-labs-backpack")
                .addToCart("sauce-labs-bike-light")
                .addToCart("sauce-labs-bolt-t-shirt");
        productsPage.getCartBadge().shouldHave(text("3"));

        // Перейти к оформлению
        CartPage cartPage = productsPage.goToCart();
        cartPage.getCartItems().shouldHave(size(3));

        CheckoutStepOnePage stepOne = cartPage.checkout();
        CheckoutStepTwoPage stepTwo = stepOne
                .fillCustomerInfo("Jane", "Smith", "54321")
                .clickContinue();

        stepTwo.getCartItems().shouldHave(size(3));

        CheckoutCompletePage completePage = stepTwo.clickFinish();
        completePage.getCompleteHeader()
                .shouldHave(text("Thank you for your order!"));
    }

    @Test
    @Story("Валидация формы checkout")
    @DisplayName("Checkout без заполнения данных — ошибка валидации")
    @Severity(SeverityLevel.NORMAL)
    void testCheckoutValidation() {
        productsPage.addToCart("sauce-labs-backpack");
        CartPage cartPage = productsPage.goToCart();
        CheckoutStepOnePage stepOne = cartPage.checkout();

        // Нажать Continue без заполнения полей
        stepOne.enterFirstName("");
        stepOne.enterLastName("");
        stepOne.enterPostalCode("");
        // Нажимаем Continue напрямую, не через clickContinue(),
        // т.к. ожидаем остаться на той же странице
        stepOne.getPageTitle(); // Убедиться, что страница загружена
        // Используем клик через кнопку
        com.codeborne.selenide.Selenide.$("#continue").click();

        stepOne.getErrorMessage()
                .shouldBe(visible)
                .shouldHave(text("First Name is required"));
    }

    @Test
    @Story("Отмена checkout")
    @DisplayName("Cancel на первом шаге возвращает в корзину")
    @Severity(SeverityLevel.MINOR)
    void testCancelCheckout() {
        productsPage.addToCart("sauce-labs-backpack");
        CartPage cartPage = productsPage.goToCart();
        CheckoutStepOnePage stepOne = cartPage.checkout();

        CartPage returnedCart = stepOne.clickCancel();
        returnedCart.getPageTitle().shouldHave(text("Your Cart"));
        returnedCart.getCartItems().shouldHave(size(1));
    }
}
```

---

## Allure-интеграция

### Иерархия аннотаций в проекте

```
@Epic("SauceDemo")                         ← весь проект
├── @Feature("Авторизация")                ← LoginTest
│   ├── testSuccessfulLogin
│   ├── testLockedOutUser
│   └── ...
├── @Feature("Корзина")                    ← CartTest
│   ├── testAddSingleItem
│   ├── testRemoveItemFromCart
│   └── ...
├── @Feature("Сортировка товаров")         ← SortingTest
│   ├── testSortByNameAZ
│   └── ...
└── @Feature("Оформление заказа")          ← CheckoutTest
    ├── @Story("Полный поток покупки")
    │   └── testCompleteCheckoutFlow
    ├── @Story("Покупка нескольких товаров")
    │   └── testCheckoutMultipleItems
    └── @Story("Валидация формы checkout")
        └── testCheckoutValidation
```

### Что попадёт в Allure-отчёт автоматически

1. **Шаги** — все методы с `@Step` в Page Object
2. **Скриншоты** — при падении теста (через AllureSelenide)
3. **Selenide-действия** — каждый `click`, `setValue` логируется как подшаг
4. **Severity** — иконка критичности рядом с каждым тестом
5. **Группировка** — Behaviors: Epic → Feature → Story

---

## Запуск и отчёт

### Запуск всех тестов

```bash
mvn clean test
```

### Запуск конкретного тестового класса

```bash
mvn clean test -Dtest=CheckoutTest
```

### Запуск конкретного теста

```bash
mvn clean test -Dtest=LoginTest#testSuccessfulLogin
```

### Генерация и просмотр Allure-отчёта

```bash
# Запустить тесты и сразу открыть отчёт
mvn clean test ; allure serve target/allure-results
```

### Headless-режим для CI/CD

Замените в `selenide.properties`:

```properties
selenide.headless=true
```

### Пример CI/CD pipeline (GitHub Actions)

```yaml
name: UI Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      - name: Run tests
        run: mvn clean test -Dselenide.headless=true
      - name: Upload Allure results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: allure-results
          path: target/allure-results
```

---

## README-шаблон для проекта

```markdown
# SauceDemo Automation

Проект автоматизации тестирования [SauceDemo](https://www.saucedemo.com).

## Стек технологий

- Java 17
- Selenide 7.x
- JUnit 5
- Allure 2.x
- Maven

## Быстрый старт

### Предусловия

- Java 17+
- Maven 3.8+
- Allure CLI (для отчётов)
- Google Chrome

### Запуск тестов

    mvn clean test

### Allure-отчёт

    allure serve target/allure-results

### Headless-режим

    mvn clean test -Dselenide.headless=true

## Структура

    src/main/java/com/saucedemo/
    ├── config/          # Конфигурация проекта
    ├── pages/           # Page Object классы
    └── utils/           # Утилиты (Allure, скриншоты)

    src/test/java/com/saucedemo/tests/
    ├── BaseTest.java    # Базовый класс тестов
    ├── LoginTest.java   # Тесты авторизации
    ├── CartTest.java    # Тесты корзины
    ├── SortingTest.java # Тесты сортировки
    └── CheckoutTest.java # E2E checkout-поток

## Тестовое покрытие

| Область       | Кол-во тестов | Критичность |
|---------------|:------------:|:-----------:|
| Авторизация   |      5       |  BLOCKER    |
| Корзина       |      6       |  CRITICAL   |
| Сортировка    |      4       |  NORMAL     |
| Checkout E2E  |      5       |  BLOCKER    |
```

---

**Результат практики:** полный, готовый к запуску проект с 20 тестами, покрывающими авторизацию, корзину, сортировку и checkout-поток. Allure-отчёт с шагами, скриншотами и группировкой по Epic/Feature/Story. Запуск: `mvn clean test && allure serve target/allure-results`.
