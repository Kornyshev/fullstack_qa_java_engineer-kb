# Page Object Model

## Обзор

Page Object Model (POM) — это паттерн проектирования в UI-автоматизации, при котором каждая страница (или значимая часть страницы) веб-приложения представлена отдельным Java-классом. Этот класс инкапсулирует локаторы элементов и методы взаимодействия с ними, отделяя логику тестов от деталей реализации UI.

POM — это **обязательный** паттерн для любого серьёзного проекта автоматизации. Без него тесты быстро превращаются в неподдерживаемый хаос. На собеседованиях POM спрашивают всегда.

---

## Проблемы без Page Object

### Код без POM (антипаттерн)

```java
@Test
void testPurchaseFlow() {
    // Логин
    driver.findElement(By.id("email")).sendKeys("user@test.com");
    driver.findElement(By.id("password")).sendKeys("pass123");
    driver.findElement(By.cssSelector("button[type='submit']")).click();

    // Добавление товара
    wait.until(ExpectedConditions.visibilityOfElementLocated(By.cssSelector(".product-card")));
    driver.findElement(By.cssSelector(".product-card:first-child .add-to-cart")).click();

    // Корзина
    driver.findElement(By.id("cartIcon")).click();
    String total = driver.findElement(By.cssSelector(".cart-total")).getText();
    assertEquals("1 000 ₽", total);
}

@Test
void testAddMultipleProducts() {
    // Снова логин — тот же код дублируется
    driver.findElement(By.id("email")).sendKeys("user@test.com");
    driver.findElement(By.id("password")).sendKeys("pass123");
    driver.findElement(By.cssSelector("button[type='submit']")).click();

    // Снова та же логика — те же локаторы повторяются
    wait.until(ExpectedConditions.visibilityOfElementLocated(By.cssSelector(".product-card")));
    driver.findElement(By.cssSelector(".product-card:nth-child(1) .add-to-cart")).click();
    driver.findElement(By.cssSelector(".product-card:nth-child(2) .add-to-cart")).click();
    // ...
}
```

### Что здесь плохо?

| Проблема | Последствие |
|----------|-------------|
| **Дублирование локаторов** | При изменении UI нужно менять локатор в десятках тестов |
| **Дублирование логики** | Логин повторяется в каждом тесте |
| **Смешение слоёв** | Тест знает о деталях UI (id, css-классы) |
| **Нечитаемость** | Тяжело понять, что проверяет тест |
| **Хрупкость** | Одно изменение в вёрстке ломает множество тестов |

---

## Структура проекта с POM

```
src/test/java/
├── pages/                       # Page Objects
│   ├── BasePage.java            # Базовый класс для всех страниц
│   ├── LoginPage.java           # Страница логина
│   ├── ProductsPage.java        # Страница каталога
│   ├── CartPage.java            # Страница корзины
│   └── components/              # Переиспользуемые компоненты
│       ├── HeaderComponent.java # Шапка сайта
│       ├── FooterComponent.java # Подвал
│       └── ModalDialog.java     # Модальное окно
├── tests/                       # Тесты
│   ├── BaseTest.java            # Базовый тестовый класс
│   ├── LoginTest.java
│   ├── ProductTest.java
│   └── CartTest.java
└── utils/                       # Утилиты
    ├── TestDataGenerator.java
    └── ScreenshotUtils.java
```

---

## BasePage — базовый класс

### С Selenium

```java
import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.support.ui.ExpectedConditions;
import org.openqa.selenium.support.ui.WebDriverWait;

import java.time.Duration;

public abstract class BasePage {

    protected WebDriver driver;
    protected WebDriverWait wait;

    public BasePage(WebDriver driver) {
        this.driver = driver;
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(10));
    }

    // Общие методы для всех страниц
    protected WebElement waitForVisible(By locator) {
        return wait.until(ExpectedConditions.visibilityOfElementLocated(locator));
    }

    protected WebElement waitForClickable(By locator) {
        return wait.until(ExpectedConditions.elementToBeClickable(locator));
    }

    protected void click(By locator) {
        waitForClickable(locator).click();
    }

    protected void type(By locator, String text) {
        WebElement element = waitForVisible(locator);
        element.clear();
        element.sendKeys(text);
    }

    protected String getText(By locator) {
        return waitForVisible(locator).getText();
    }

    protected boolean isDisplayed(By locator) {
        try {
            return driver.findElement(locator).isDisplayed();
        } catch (Exception e) {
            return false;
        }
    }

    public String getTitle() {
        return driver.getTitle();
    }

    public String getCurrentUrl() {
        return driver.getCurrentUrl();
    }
}
```

### С Selenide

```java
import com.codeborne.selenide.SelenideElement;
import org.openqa.selenium.By;

import static com.codeborne.selenide.Selenide.page;

public abstract class BasePage {

    // В Selenide не нужен WebDriver и явные ожидания —
    // всё встроено в фреймворк.

    // Метод для проверки, что страница загружена
    // Каждый наследник определяет свой критерий
    public abstract BasePage waitForPageLoaded();
}
```

> **Замечание:** С Selenide базовый класс значительно проще, потому что ожидания и управление драйвером встроены в фреймворк.

---

## Конкретные Page Objects

### LoginPage

**Selenium-версия:**

```java
public class LoginPage extends BasePage {

    // Локаторы — приватные, инкапсулированы внутри класса
    private final By emailField = By.id("email");
    private final By passwordField = By.id("password");
    private final By loginButton = By.cssSelector("button[type='submit']");
    private final By errorMessage = By.cssSelector(".alert-danger");
    private final By forgotPasswordLink = By.linkText("Забыли пароль?");

    public LoginPage(WebDriver driver) {
        super(driver);
    }

    // Действия на странице
    public LoginPage enterEmail(String email) {
        type(emailField, email);
        return this; // Fluent-паттерн — возвращаем текущую страницу
    }

    public LoginPage enterPassword(String password) {
        type(passwordField, password);
        return this;
    }

    public ProductsPage clickLogin() {
        click(loginButton);
        return new ProductsPage(driver); // Переход на другую страницу
    }

    public LoginPage clickLoginExpectingError() {
        click(loginButton);
        return this; // Остаёмся на той же странице
    }

    // Составной метод — выполняет полный сценарий логина
    public ProductsPage loginAs(String email, String password) {
        enterEmail(email);
        enterPassword(password);
        return clickLogin();
    }

    // Методы для проверок (геттеры данных)
    public String getErrorMessage() {
        return getText(errorMessage);
    }

    public boolean isErrorDisplayed() {
        return isDisplayed(errorMessage);
    }

    @Override
    public LoginPage waitForPageLoaded() {
        waitForVisible(emailField);
        return this;
    }
}
```

**Selenide-версия:**

```java
import com.codeborne.selenide.SelenideElement;

import static com.codeborne.selenide.Condition.*;
import static com.codeborne.selenide.Selenide.$;

public class LoginPage extends BasePage {

    // Локаторы как SelenideElement
    private final SelenideElement emailField = $("#email");
    private final SelenideElement passwordField = $("#password");
    private final SelenideElement loginButton = $("button[type='submit']");
    private final SelenideElement errorMessage = $(".alert-danger");

    public LoginPage enterEmail(String email) {
        emailField.setValue(email);
        return this;
    }

    public LoginPage enterPassword(String password) {
        passwordField.setValue(password);
        return this;
    }

    public ProductsPage clickLogin() {
        loginButton.click();
        return new ProductsPage();
    }

    public ProductsPage loginAs(String email, String password) {
        enterEmail(email);
        enterPassword(password);
        return clickLogin();
    }

    public LoginPage errorMessageShouldBe(String expectedText) {
        errorMessage.shouldHave(text(expectedText));
        return this;
    }

    @Override
    public LoginPage waitForPageLoaded() {
        emailField.shouldBe(visible);
        return this;
    }
}
```

### ProductsPage

```java
import static com.codeborne.selenide.Condition.*;
import static com.codeborne.selenide.CollectionCondition.*;
import static com.codeborne.selenide.Selenide.*;

public class ProductsPage extends BasePage {

    private final ElementsCollection productCards = $$(".product-card");
    private final SelenideElement searchField = $("[data-testid='search']");
    private final SelenideElement cartIcon = $("#cartIcon");
    private final SelenideElement cartBadge = $(".cart-badge");

    public ProductsPage searchFor(String query) {
        searchField.setValue(query).pressEnter();
        return this;
    }

    public ProductsPage addProductToCart(int index) {
        productCards.get(index).$(".add-to-cart").click();
        return this;
    }

    public ProductsPage addProductToCartByName(String productName) {
        productCards.filterBy(text(productName))
                    .first()
                    .$(".add-to-cart")
                    .click();
        return this;
    }

    public CartPage openCart() {
        cartIcon.click();
        return new CartPage();
    }

    // Методы для проверок
    public ProductsPage shouldHaveProducts(int count) {
        productCards.shouldHave(size(count));
        return this;
    }

    public ProductsPage cartBadgeShouldShow(String count) {
        cartBadge.shouldHave(text(count));
        return this;
    }

    @Override
    public ProductsPage waitForPageLoaded() {
        productCards.first().shouldBe(visible);
        return this;
    }
}
```

### CartPage

```java
import static com.codeborne.selenide.Condition.*;
import static com.codeborne.selenide.CollectionCondition.*;
import static com.codeborne.selenide.Selenide.*;

public class CartPage extends BasePage {

    private final ElementsCollection cartItems = $$(".cart-item");
    private final SelenideElement totalPrice = $(".cart-total");
    private final SelenideElement checkoutButton = $("[data-testid='checkout']");
    private final SelenideElement emptyCartMessage = $(".empty-cart");

    public CartPage removeItem(int index) {
        cartItems.get(index).$(".remove-btn").click();
        return this;
    }

    public CartPage changeQuantity(int index, int quantity) {
        cartItems.get(index).$(".quantity-input").setValue(String.valueOf(quantity));
        return this;
    }

    public CheckoutPage proceedToCheckout() {
        checkoutButton.click();
        return new CheckoutPage();
    }

    // Проверки
    public CartPage shouldHaveItems(int count) {
        cartItems.shouldHave(size(count));
        return this;
    }

    public CartPage totalPriceShouldBe(String price) {
        totalPrice.shouldHave(text(price));
        return this;
    }

    public CartPage shouldBeEmpty() {
        emptyCartMessage.shouldBe(visible);
        cartItems.shouldHave(size(0));
        return this;
    }

    @Override
    public CartPage waitForPageLoaded() {
        $(".cart-content").shouldBe(visible);
        return this;
    }
}
```

---

## Компоненты (Components)

Компоненты — это переиспользуемые части UI, которые присутствуют на нескольких страницах: шапка, подвал, модальные окна, панель навигации.

### HeaderComponent

```java
import static com.codeborne.selenide.Condition.*;
import static com.codeborne.selenide.Selenide.$;

public class HeaderComponent {

    private final SelenideElement logo = $(".header-logo");
    private final SelenideElement searchField = $(".header-search input");
    private final SelenideElement cartIcon = $(".header-cart");
    private final SelenideElement cartBadge = $(".header-cart .badge");
    private final SelenideElement userMenu = $(".user-menu");
    private final SelenideElement logoutButton = $(".user-menu .logout");

    public HeaderComponent searchFor(String query) {
        searchField.setValue(query).pressEnter();
        return this;
    }

    public CartPage openCart() {
        cartIcon.click();
        return new CartPage();
    }

    public HeaderComponent openUserMenu() {
        userMenu.click();
        return this;
    }

    public LoginPage logout() {
        openUserMenu();
        logoutButton.click();
        return new LoginPage();
    }

    public HeaderComponent cartBadgeShouldShow(String count) {
        cartBadge.shouldHave(text(count));
        return this;
    }
}
```

### Использование компонентов в Page Objects

```java
public class ProductsPage extends BasePage {

    // Компонент шапки — доступен на этой странице
    private final HeaderComponent header = new HeaderComponent();

    // ... остальные элементы и методы страницы ...

    public HeaderComponent header() {
        return header;
    }

    @Override
    public ProductsPage waitForPageLoaded() {
        // ...
        return this;
    }
}
```

### ModalDialog (универсальный компонент)

```java
public class ConfirmDialog {

    private final SelenideElement dialog = $(".modal-dialog");
    private final SelenideElement title = dialog.$(".modal-title");
    private final SelenideElement message = dialog.$(".modal-body");
    private final SelenideElement confirmButton = dialog.$(".btn-confirm");
    private final SelenideElement cancelButton = dialog.$(".btn-cancel");

    public ConfirmDialog shouldHaveTitle(String expectedTitle) {
        title.shouldHave(text(expectedTitle));
        return this;
    }

    public ConfirmDialog shouldHaveMessage(String expectedMessage) {
        message.shouldHave(text(expectedMessage));
        return this;
    }

    public void confirm() {
        confirmButton.click();
        dialog.shouldNotBe(visible);
    }

    public void cancel() {
        cancelButton.click();
        dialog.shouldNotBe(visible);
    }
}
```

---

## Page Factory (@FindBy)

Page Factory — это встроенный механизм Selenium для инициализации элементов через аннотации. Менее популярен при использовании Selenide, но часто встречается в legacy-проектах.

```java
import org.openqa.selenium.WebElement;
import org.openqa.selenium.support.FindBy;
import org.openqa.selenium.support.PageFactory;

public class LoginPage extends BasePage {

    @FindBy(id = "email")
    private WebElement emailField;

    @FindBy(id = "password")
    private WebElement passwordField;

    @FindBy(css = "button[type='submit']")
    private WebElement loginButton;

    @FindBy(css = ".alert-danger")
    private WebElement errorMessage;

    public LoginPage(WebDriver driver) {
        super(driver);
        // Инициализация элементов через PageFactory
        PageFactory.initElements(driver, this);
    }

    public void enterEmail(String email) {
        emailField.clear();
        emailField.sendKeys(email);
    }

    public void clickLogin() {
        loginButton.click();
    }
}
```

### Проблемы Page Factory

| Проблема | Описание |
|----------|----------|
| `StaleElementReferenceException` | Элементы кешируются при инициализации и могут устареть при перерисовке DOM |
| Нет поддержки коллекций с условиями | `@FindBy` не позволяет фильтровать |
| Обязательный вызов `PageFactory.initElements` | Легко забыть, получишь `NullPointerException` |
| Не работает с Selenide | Selenide использует свой механизм lazy-инициализации |

> **Рекомендация:** В новых проектах предпочитайте обычные `By`-локаторы (Selenium) или `$`/`$$` (Selenide) вместо Page Factory.

---

## Fluent POM (цепочки вызовов)

Fluent POM — это стиль, при котором методы Page Object возвращают объект страницы, позволяя строить выразительные цепочки:

```java
@Test
void testPurchaseFlow() {
    new LoginPage()
        .enterEmail("user@test.com")
        .enterPassword("pass123")
        .clickLogin()                        // Возвращает ProductsPage
        .waitForPageLoaded()
        .searchFor("Ноутбук")
        .addProductToCartByName("MacBook Pro")
        .cartBadgeShouldShow("1")
        .openCart()                           // Возвращает CartPage
        .shouldHaveItems(1)
        .totalPriceShouldBe("149 990 ₽")
        .proceedToCheckout();                 // Возвращает CheckoutPage
}
```

### Правило возвращаемых типов

- Если действие **остаётся на текущей странице** → возвращаем `this`
- Если действие **переводит на другую страницу** → возвращаем новый Page Object
- Если **результат неизвестен** (успех или ошибка) → создаём два метода:

```java
public class LoginPage extends BasePage {

    // Успешный логин → переход на ProductsPage
    public ProductsPage clickLogin() {
        loginButton.click();
        return new ProductsPage();
    }

    // Неуспешный логин → остаёмся на LoginPage
    public LoginPage clickLoginExpectingError() {
        loginButton.click();
        return this;
    }
}
```

---

## Step Object Pattern

Step Object — это дополнительный уровень абстракции над Page Objects, группирующий действия в бизнес-шаги:

```java
public class AuthSteps {

    public ProductsPage loginAsStandardUser() {
        return open("/login", LoginPage.class)
            .loginAs("standard_user@test.com", "password123");
    }

    public LoginPage loginWithInvalidCredentials(String email, String password) {
        return open("/login", LoginPage.class)
            .enterEmail(email)
            .enterPassword(password)
            .clickLoginExpectingError();
    }
}

public class CartSteps {

    public CartPage addProductAndOpenCart(String productName) {
        return new ProductsPage()
            .waitForPageLoaded()
            .addProductToCartByName(productName)
            .openCart();
    }
}
```

### Использование в тестах

```java
class PurchaseTest extends BaseTest {

    private final AuthSteps authSteps = new AuthSteps();
    private final CartSteps cartSteps = new CartSteps();

    @Test
    void userCanPurchaseProduct() {
        authSteps.loginAsStandardUser();

        cartSteps.addProductAndOpenCart("MacBook Pro")
            .shouldHaveItems(1)
            .totalPriceShouldBe("149 990 ₽")
            .proceedToCheckout();
    }
}
```

> Step Object особенно полезен, когда одна и та же последовательность действий повторяется во многих тестах (например, логин, навигация к определённому разделу).

---

## Дискуссия: Assertions в POM

Это одна из самых обсуждаемых тем в QA-сообществе.

### Подход 1: Assertions только в тестах (классический)

```java
// Page Object — только действия и геттеры
public class ProductsPage {
    public int getProductCount() {
        return $$(".product-card").size();
    }
}

// Тест — содержит assertions
@Test
void shouldShowTenProducts() {
    assertEquals(10, productsPage.getProductCount());
}
```

**За:** Чистое разделение ответственности, Page Object не зависит от assert-библиотеки.

### Подход 2: Assertions в POM (с Selenide)

```java
// Page Object — содержит встроенные проверки
public class ProductsPage {
    public ProductsPage shouldHaveProducts(int count) {
        $$(".product-card").shouldHave(size(count));
        return this;
    }
}

// Тест — вызывает методы с проверками
@Test
void shouldShowTenProducts() {
    productsPage.shouldHaveProducts(10);
}
```

**За:** Fluent API, автоматические ожидания Selenide, более читаемые тесты.

### Рекомендация

В проектах с **Selenide** методы-проверки (`shouldHave*`, `shouldBe*`) в Page Objects — **нормальная практика**, так как они неотделимы от механизма ожиданий Selenide. В проектах с чистым **Selenium** лучше держать assertions в тестах, а в Page Objects предоставлять данные через геттеры.

---

## POM с Selenide vs Selenium

| Аспект | POM + Selenium | POM + Selenide |
|--------|----------------|----------------|
| **BasePage** | Нужен: WebDriver, WebDriverWait, утилиты | Минимальный или не нужен |
| **Конструктор** | `new LoginPage(driver)` — передаём драйвер | `new LoginPage()` — драйвер глобальный |
| **Локаторы** | `By.id("email")` | `$("#email")` как `SelenideElement` |
| **Page Factory** | Можно использовать `@FindBy` | Не нужен — `$` и так lazy |
| **Ожидания** | Ручные в каждом методе | Встроены в `$` и `shouldHave` |
| **Assertions в POM** | Не рекомендуется | Допустимо (`shouldHave`) |
| **Объём кода** | Больше (boilerplate) | Меньше |

---

## Полный пример: тестовый класс

```java
import com.codeborne.selenide.Configuration;
import com.codeborne.selenide.logevents.SelenideLogger;
import io.qameta.allure.selenide.AllureSelenide;
import org.junit.jupiter.api.*;

import static com.codeborne.selenide.Selenide.open;

class PurchaseFlowTest {

    @BeforeAll
    static void setUpAll() {
        Configuration.baseUrl = "https://example.com";
        Configuration.browserSize = "1920x1080";
        SelenideLogger.addListener("allure", new AllureSelenide());
    }

    @BeforeEach
    void setUp() {
        open("/login");
    }

    @Test
    @DisplayName("Пользователь может купить товар через поиск")
    void userCanSearchAndPurchaseProduct() {
        new LoginPage()
            .loginAs("user@test.com", "password123")
            .waitForPageLoaded()
            .searchFor("Наушники")
            .shouldHaveProducts(5)
            .addProductToCartByName("AirPods Pro")
            .cartBadgeShouldShow("1")
            .openCart()
            .shouldHaveItems(1)
            .totalPriceShouldBe("24 990 ₽");
    }

    @Test
    @DisplayName("Пользователь может удалить товар из корзины")
    void userCanRemoveItemFromCart() {
        new LoginPage()
            .loginAs("user@test.com", "password123")
            .waitForPageLoaded()
            .addProductToCart(0)
            .openCart()
            .shouldHaveItems(1)
            .removeItem(0)
            .shouldBeEmpty();
    }

    @Test
    @DisplayName("Отображение ошибки при неверном пароле")
    void errorOnInvalidLogin() {
        new LoginPage()
            .enterEmail("user@test.com")
            .enterPassword("wrongpassword")
            .clickLoginExpectingError()
            .errorMessageShouldBe("Неверный email или пароль");
    }
}
```

---

## Best Practices

### Правила для Page Objects

1. **Один класс = одна страница (или логический блок)**. Не объединяйте несколько страниц в один класс.
2. **Локаторы — приватные**. Тесты не должны знать о внутренней структуре DOM.
3. **Методы — публичные, понятные**. Имена должны отражать бизнес-действие: `loginAs()`, а не `fillFieldsAndClickButton()`.
4. **Не дублируйте логику** — выносите общие действия в BasePage или Steps.
5. **Используйте компоненты** для повторяющихся блоков (Header, Footer, Sidebar).
6. **Fluent-стиль** — методы возвращают объект страницы для построения цепочек.
7. **Не храните состояние** — Page Object не должен хранить данные между вызовами.
8. **Понятная структура пакетов** — `pages/`, `components/`, `tests/`, `steps/`.

### Антипаттерны

| Антипаттерн | Почему плохо | Как исправить |
|-------------|------------|---------------|
| God Page Object (всё в одном классе) | Невозможно поддерживать | Разбить на отдельные страницы и компоненты |
| Публичные локаторы | Тесты привязаны к реализации UI | Сделать локаторы приватными |
| Бизнес-логика в Page Object | Нарушение SRP | Логику — в тесты или Steps |
| Page Object без методов (только локаторы) | Не инкапсулирует поведение | Добавить методы взаимодействия |
| Наследование для несвязанных страниц | Хрупкая иерархия | Композиция (компоненты) вместо наследования |
| Методы `void` (не Fluent) | Нельзя строить цепочки | Возвращать `this` или новый Page Object |

---

## Связь с тестированием

- POM — **основной архитектурный паттерн** в UI-автоматизации на любом проекте.
- Обеспечивает **масштабируемость**: легко добавлять новые тесты и страницы.
- Снижает **стоимость поддержки**: изменение UI требует правки только в одном месте.
- Повышает **читаемость** тестов — тест читается как пользовательский сценарий.
- Является **обязательной темой** на собеседованиях для QA Automation Engineer.

---

## Типичные ошибки

| Ошибка | Последствия | Правильный подход |
|--------|-------------|-------------------|
| Отсутствие POM в проекте | Копипаст локаторов, высокая стоимость поддержки | Внедрить POM с первого теста |
| Слишком глубокая иерархия наследования | Сложно понять, что откуда берётся | Максимум 2 уровня: `BasePage` → `ConcretePage` |
| Page Object знает о других Page Objects слишком много | Связанность (coupling) | Возвращать новый Page Object из метода, но не вызывать его методы |
| Все методы в одном BasePage | God Object | BasePage — только общие утилиты |
| Хранение WebDriver в static-поле | Проблемы при параллельном запуске | ThreadLocal или DI (Dependency Injection) |
| Отсутствие `waitForPageLoaded()` | Тесты падают из-за незагруженных страниц | Каждый Page Object проверяет ключевой элемент |

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Что такое Page Object Model? Зачем он нужен?
2. Из чего состоит типичный Page Object?
3. Что такое BasePage и зачем он нужен?
4. Как POM помогает при изменении UI?
5. Что такое Page Factory? Какие у него плюсы и минусы?

### 🟡 Средний уровень
6. Что такое Fluent POM? Как строятся цепочки вызовов?
7. Как вы организуете переиспользуемые компоненты (Header, Modal)?
8. Должны ли assertions быть в Page Object или только в тестах? Обоснуйте.
9. Чем отличается POM с Selenium от POM с Selenide?
10. Что такое Step Object Pattern? Когда его применять?
11. Как организовать POM для Single Page Application (SPA)?

### 🔴 Продвинутый уровень
12. Как обеспечить потокобезопасность Page Objects при параллельном запуске тестов?
13. Как вы реализуете POM для приложения с динамически генерируемыми страницами?
14. Как бы вы спроектировали POM для приложения с ролевой моделью (разные UI для разных ролей)?
15. Предложите архитектуру тестового фреймворка: Page Objects, Steps, Tests, Utilities, Configuration. Какие принципы SOLID здесь применяются?
16. Как вы обрабатываете ситуацию, когда одно действие может привести к разным страницам в зависимости от данных?

---

## Практические задания

### Задание 1: Создание POM с нуля
Для сайта `https://www.saucedemo.com/`:
- Создайте `LoginPage`, `InventoryPage`, `CartPage`
- Реализуйте `BasePage` с общими методами
- Напишите 3 теста, используя Page Objects

### Задание 2: Рефакторинг
Перепишите следующий тест с использованием POM:
```java
@Test
void test() {
    open("https://the-internet.herokuapp.com/login");
    $("#username").setValue("tomsmith");
    $("#password").setValue("SuperSecretPassword!");
    $("button[type='submit']").click();
    $(".flash.success").shouldBe(visible);
    $(".button.secondary").click(); // Logout
    $(".flash.success").shouldBe(visible);
}
```

### Задание 3: Компоненты
Добавьте к проекту из задания 1:
- `HeaderComponent` с методами навигации и поиска
- `ConfirmDialog` для обработки модальных окон
- Используйте компоненты в Page Objects

### Задание 4: Step Object
Создайте `AuthSteps` и `PurchaseSteps` поверх существующих Page Objects. Перепишите тесты, используя Step Objects.

### Задание 5: Архитектурное решение
Спроектируйте POM-архитектуру для e-commerce приложения с:
- 10+ страницами
- 5 переиспользуемыми компонентами
- Авторизацией с разными ролями (покупатель, продавец, админ)
- Нарисуйте UML-диаграмму классов

---

## Дополнительные ресурсы

- [Martin Fowler — PageObject](https://martinfowler.com/bliki/PageObject.html)
- [Selenium Wiki — Page Object Pattern](https://www.selenium.dev/documentation/test_practices/encouraged/page_object_models/)
- [Selenide Wiki — Page Objects](https://github.com/selenide/selenide/wiki/Page-Objects)
- [Baeldung — Page Object Pattern with Selenium](https://www.baeldung.com/selenium-webdriver-page-object)
- [SauceDemo — тестовое приложение для практики](https://www.saucedemo.com/)
- [Test Automation University — Page Object Model](https://testautomationu.applitools.com/)
