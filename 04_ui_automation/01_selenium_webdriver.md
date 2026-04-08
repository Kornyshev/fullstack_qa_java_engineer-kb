# Selenium WebDriver

## Обзор

Selenium WebDriver — это основной инструмент для автоматизации UI-тестирования веб-приложений. Он предоставляет программный интерфейс для управления браузером: навигация по страницам, поиск элементов, взаимодействие с формами, кнопками и другими элементами интерфейса. Selenium WebDriver поддерживает все основные браузеры (Chrome, Firefox, Edge, Safari) и множество языков программирования, включая Java.

Для QA-инженера владение Selenium — это базовый навык, необходимый практически на любой позиции, связанной с автоматизацией тестирования.

---

## Архитектура Selenium WebDriver

### WebDriver Protocol (W3C)

Selenium WebDriver работает по клиент-серверной архитектуре на основе протокола W3C WebDriver:

```
┌──────────────┐      HTTP/JSON       ┌────────────────┐       Native       ┌──────────┐
│  Тестовый     │ ──────────────────► │  Browser Driver │ ──────────────►   │  Браузер │
│  скрипт (Java)│ ◄────────────────── │  (chromedriver) │ ◄──────────────   │ (Chrome) │
└──────────────┘      Ответ JSON      └────────────────┘     Команды       └──────────┘
```

1. **Тестовый скрипт** — ваш Java-код, использующий Selenium API.
2. **Browser Driver** — отдельный исполняемый файл (chromedriver, geckodriver и т.д.), принимающий HTTP-запросы и транслирующий их в нативные команды браузера.
3. **Браузер** — реальный браузер, в котором выполняются действия.

### Как работает взаимодействие

Каждый вызов метода Selenium (например, `findElement`, `click`) преобразуется в HTTP-запрос к драйверу браузера. Драйвер выполняет команду и возвращает результат в формате JSON. Это означает, что каждое действие — это сетевой вызов, что влияет на скорость выполнения тестов.

---

## Настройка проекта

### Maven-зависимости

```xml
<dependencies>
    <!-- Selenium WebDriver -->
    <dependency>
        <groupId>org.seleniumhq.selenium</groupId>
        <artifactId>selenium-java</artifactId>
        <version>4.27.0</version>
    </dependency>

    <!-- WebDriverManager для автоматического управления драйверами -->
    <dependency>
        <groupId>io.github.bonigarcia</groupId>
        <artifactId>webdrivermanager</artifactId>
        <version>5.9.2</version>
    </dependency>

    <!-- JUnit 5 -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.11.3</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### WebDriverManager

До появления WebDriverManager приходилось вручную скачивать драйверы браузеров и указывать путь к ним. WebDriverManager автоматизирует этот процесс:

```java
import io.github.bonigarcia.wdm.WebDriverManager;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;

public class DriverSetup {

    // Настройка с помощью WebDriverManager
    public static WebDriver createDriver() {
        WebDriverManager.chromedriver().setup();

        ChromeOptions options = new ChromeOptions();
        options.addArguments("--start-maximized");       // Открыть на весь экран
        options.addArguments("--disable-notifications");  // Отключить уведомления
        // options.addArguments("--headless=new");        // Безголовый режим (без GUI)

        return new ChromeDriver(options);
    }
}
```

> **Примечание:** Начиная с Selenium 4.6+, Selenium Manager встроен в библиотеку и может сам управлять драйверами, делая WebDriverManager необязательным. Однако WebDriverManager по-прежнему популярен и предоставляет более гибкую конфигурацию.

### Selenium Manager (встроенный, начиная с 4.6)

```java
// Без дополнительных зависимостей — Selenium сам загрузит нужный драйвер
WebDriver driver = new ChromeDriver();
```

---

## Локаторы (Locators)

Локаторы — это стратегии поиска элементов на странице. Выбор правильного локатора — ключевой навык QA-автоматизатора.

### Типы локаторов

| Локатор | Метод | Пример | Приоритет |
|---------|-------|--------|-----------|
| **ID** | `By.id("login")` | `<input id="login">` | Высший — самый быстрый и надёжный |
| **Name** | `By.name("username")` | `<input name="username">` | Высокий |
| **CSS Selector** | `By.cssSelector(".btn-primary")` | `<button class="btn-primary">` | Высокий — гибкий и быстрый |
| **XPath** | `By.xpath("//div[@class='card']")` | Любая структура DOM | Средний — мощный, но медленнее |
| **Class Name** | `By.className("active")` | `<div class="active">` | Средний |
| **Tag Name** | `By.tagName("input")` | Все `<input>` элементы | Низкий — слишком общий |
| **Link Text** | `By.linkText("Войти")` | `<a>Войти</a>` | Низкий — только для ссылок |
| **Partial Link Text** | `By.partialLinkText("Вой")` | `<a>Войти в систему</a>` | Низкий |

### Примеры CSS Selectors

```java
// По ID
driver.findElement(By.cssSelector("#loginForm"));

// По классу
driver.findElement(By.cssSelector(".error-message"));

// По атрибуту
driver.findElement(By.cssSelector("input[type='email']"));

// По атрибуту data-testid (рекомендуемый подход)
driver.findElement(By.cssSelector("[data-testid='submit-button']"));

// Комбинированный: элемент внутри другого
driver.findElement(By.cssSelector("form#login .submit-btn"));

// Псевдоклассы
driver.findElement(By.cssSelector("ul > li:first-child"));
driver.findElement(By.cssSelector("tr:nth-child(3)"));
```

### Примеры XPath

```java
// Абсолютный путь (НЕЛЬЗЯ использовать — крайне хрупкий)
driver.findElement(By.xpath("/html/body/div[1]/form/input"));

// Относительный путь
driver.findElement(By.xpath("//input[@id='username']"));

// Поиск по тексту
driver.findElement(By.xpath("//button[text()='Отправить']"));

// Частичное совпадение текста
driver.findElement(By.xpath("//button[contains(text(),'Отпр')]"));

// Поиск по нескольким атрибутам
driver.findElement(By.xpath("//input[@type='text' and @name='query']"));

// Навигация к родителю
driver.findElement(By.xpath("//span[@class='error']/.."));

// Поиск элемента, следующего за другим (following-sibling)
driver.findElement(By.xpath("//label[text()='Email']/following-sibling::input"));
```

> **Рекомендация:** Отдавайте предпочтение `data-testid` атрибутам и CSS Selectors. XPath используйте, когда CSS не справляется (например, поиск по тексту или навигация вверх по DOM).

---

## Поиск элементов

### findElement vs findElements

```java
// Возвращает ОДИН элемент; бросает NoSuchElementException, если не найден
WebElement button = driver.findElement(By.id("submitBtn"));

// Возвращает СПИСОК элементов; возвращает пустой список, если ничего не найдено
List<WebElement> items = driver.findElements(By.cssSelector(".product-card"));

// Проверка наличия элемента без исключения
boolean isPresent = !driver.findElements(By.id("errorMsg")).isEmpty();
```

### Поиск элементов внутри элементов

```java
// Поиск вложенного элемента относительно родительского
WebElement form = driver.findElement(By.id("loginForm"));
WebElement emailField = form.findElement(By.name("email"));
WebElement submitBtn = form.findElement(By.cssSelector("button[type='submit']"));
```

---

## Взаимодействие с элементами

### Основные действия

```java
WebDriver driver = new ChromeDriver();
driver.get("https://example.com/login");

// Ввод текста
WebElement emailField = driver.findElement(By.id("email"));
emailField.clear();           // Очистить поле перед вводом
emailField.sendKeys("user@example.com");

// Клик по кнопке
driver.findElement(By.id("loginBtn")).click();

// Получение текста элемента
String errorText = driver.findElement(By.cssSelector(".alert")).getText();

// Получение значения атрибута
String placeholder = driver.findElement(By.id("search")).getAttribute("placeholder");
String inputValue = driver.findElement(By.id("email")).getAttribute("value");

// Проверка состояния элемента
boolean isDisplayed = driver.findElement(By.id("modal")).isDisplayed();
boolean isEnabled = driver.findElement(By.id("submitBtn")).isEnabled();
boolean isSelected = driver.findElement(By.id("rememberMe")).isSelected();

// Работа с выпадающим списком (Select)
WebElement dropdown = driver.findElement(By.id("country"));
Select select = new Select(dropdown);
select.selectByVisibleText("Россия");
select.selectByValue("RU");
select.selectByIndex(2);
String selectedOption = select.getFirstSelectedOption().getText();
```

---

## Ожидания (Waits)

Ожидания — критически важная тема. Веб-приложения динамичны: элементы загружаются асинхронно, появляются и исчезают. Без правильных ожиданий тесты будут нестабильными (flaky).

### Thread.sleep — АНТИПАТТЕРН

```java
// НИКОГДА не используйте в реальных тестах!
Thread.sleep(3000); // Ждёт ровно 3 секунды, даже если элемент появился через 0.5 с
```

### Implicit Wait (неявное ожидание)

```java
// Устанавливается один раз, действует на все findElement / findElements
driver.manage().timeouts().implicitlyWait(Duration.ofSeconds(10));

// Если элемент не найден сразу, Selenium будет повторять попытки в течение 10 секунд
WebElement element = driver.findElement(By.id("dynamicElement"));
```

**Минусы:** Нельзя задать конкретное условие; применяется глобально; может конфликтовать с explicit wait.

### Explicit Wait (явное ожидание) — рекомендуемый подход

```java
WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(15));

// Ждём, пока элемент станет видимым
WebElement element = wait.until(
    ExpectedConditions.visibilityOfElementLocated(By.id("result"))
);

// Ждём, пока элемент станет кликабельным
wait.until(ExpectedConditions.elementToBeClickable(By.id("submitBtn"))).click();

// Ждём, пока текст появится в элементе
wait.until(ExpectedConditions.textToBePresentInElementLocated(
    By.id("status"), "Успешно"
));

// Ждём, пока элемент исчезнет
wait.until(ExpectedConditions.invisibilityOfElementLocated(By.id("spinner")));

// Ждём, пока URL будет содержать подстроку
wait.until(ExpectedConditions.urlContains("/dashboard"));

// Ждём, пока появится определённое количество элементов
wait.until(ExpectedConditions.numberOfElementsToBe(By.cssSelector(".item"), 5));

// Собственное условие
wait.until(driver -> {
    // Возвращаем не-null / true, когда условие выполнено
    String value = driver.findElement(By.id("counter")).getText();
    return Integer.parseInt(value) > 10 ? value : null;
});
```

### Fluent Wait (гибкое ожидание)

```java
Wait<WebDriver> fluentWait = new FluentWait<>(driver)
    .withTimeout(Duration.ofSeconds(30))            // Максимальное время ожидания
    .pollingEvery(Duration.ofMillis(500))            // Интервал между попытками
    .ignoring(NoSuchElementException.class)          // Игнорируемые исключения
    .ignoring(StaleElementReferenceException.class)
    .withMessage("Элемент так и не появился на странице");

WebElement element = fluentWait.until(
    ExpectedConditions.visibilityOfElementLocated(By.id("data"))
);
```

> **Важно:** Не смешивайте implicit и explicit waits — это может привести к непредсказуемому поведению и увеличению времени ожидания.

---

## Actions Class

Класс `Actions` позволяет выполнять сложные пользовательские взаимодействия: наведение мыши, перетаскивание, контекстное меню и т.д.

```java
Actions actions = new Actions(driver);

// Наведение мыши (hover)
WebElement menu = driver.findElement(By.id("mainMenu"));
actions.moveToElement(menu).perform();

// Двойной клик
WebElement cell = driver.findElement(By.cssSelector(".editable-cell"));
actions.doubleClick(cell).perform();

// Правый клик (контекстное меню)
WebElement item = driver.findElement(By.id("file"));
actions.contextClick(item).perform();

// Drag and Drop
WebElement source = driver.findElement(By.id("draggable"));
WebElement target = driver.findElement(By.id("droppable"));
actions.dragAndDrop(source, target).perform();

// Зажатие клавиши при клике (Ctrl+Click)
actions.keyDown(Keys.CONTROL)
       .click(driver.findElement(By.id("item1")))
       .click(driver.findElement(By.id("item2")))
       .keyUp(Keys.CONTROL)
       .perform();

// Цепочка действий
actions.moveToElement(menu)
       .pause(Duration.ofMillis(500))
       .click()
       .perform();
```

---

## Работа с Alerts

```java
// Переключение на alert
Alert alert = driver.switchTo().alert();

// Получить текст alert
String alertText = alert.getText();

// Принять alert (нажать OK)
alert.accept();

// Отклонить alert (нажать Cancel)
alert.dismiss();

// Ввести текст в prompt
alert.sendKeys("Введённый текст");

// Ожидание появления alert
WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(5));
wait.until(ExpectedConditions.alertIsPresent());
```

---

## Работа с IFrames

```java
// Переключение на iframe по элементу
WebElement iframeElement = driver.findElement(By.id("contentFrame"));
driver.switchTo().frame(iframeElement);

// Переключение по индексу
driver.switchTo().frame(0);

// Переключение по имени или ID
driver.switchTo().frame("frameName");

// Работа с элементами внутри iframe
driver.findElement(By.id("innerButton")).click();

// Возврат в основной контент
driver.switchTo().defaultContent();

// Возврат на один уровень вверх (из вложенного iframe)
driver.switchTo().parentFrame();
```

---

## Работа с несколькими окнами и вкладками

```java
// Запоминаем текущее окно
String originalWindow = driver.getWindowHandle();

// Клик, открывающий новое окно/вкладку
driver.findElement(By.linkText("Открыть в новом окне")).click();

// Получаем все окна
Set<String> allWindows = driver.getWindowHandles();

// Переключаемся на новое окно
for (String windowHandle : allWindows) {
    if (!windowHandle.equals(originalWindow)) {
        driver.switchTo().window(windowHandle);
        break;
    }
}

// Работаем в новом окне
String title = driver.getTitle();

// Закрываем текущее окно и возвращаемся в исходное
driver.close();
driver.switchTo().window(originalWindow);

// Открытие новой вкладки программно (Selenium 4+)
driver.switchTo().newWindow(WindowType.TAB);
driver.get("https://example.com");

// Открытие нового окна
driver.switchTo().newWindow(WindowType.WINDOW);
```

---

## Скриншоты

```java
// Скриншот всей страницы
File screenshot = ((TakesScreenshot) driver).getScreenshotAs(OutputType.FILE);
Files.copy(screenshot.toPath(), Path.of("screenshots/full_page.png"));

// Скриншот конкретного элемента
WebElement element = driver.findElement(By.id("errorPanel"));
File elementScreenshot = element.getScreenshotAs(OutputType.FILE);
Files.copy(elementScreenshot.toPath(), Path.of("screenshots/error_panel.png"));

// Скриншот как массив байт (удобно для Allure)
byte[] screenshotBytes = ((TakesScreenshot) driver).getScreenshotAs(OutputType.BYTES);
```

### Автоматический скриншот при падении теста (JUnit 5)

```java
import org.junit.jupiter.api.extension.ExtensionContext;
import org.junit.jupiter.api.extension.TestWatcher;

public class ScreenshotOnFailure implements TestWatcher {

    @Override
    public void testFailed(ExtensionContext context, Throwable cause) {
        // Получаем драйвер из контекста теста
        WebDriver driver = getDriverFromContext(context);
        if (driver != null) {
            byte[] screenshot = ((TakesScreenshot) driver).getScreenshotAs(OutputType.BYTES);
            String testName = context.getDisplayName();
            // Сохраняем или прикрепляем к отчёту
            saveScreenshot(screenshot, testName);
        }
    }
}
```

---

## Полный пример теста

```java
import io.github.bonigarcia.wdm.WebDriverManager;
import org.junit.jupiter.api.*;
import org.openqa.selenium.*;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.support.ui.ExpectedConditions;
import org.openqa.selenium.support.ui.WebDriverWait;

import java.time.Duration;

import static org.junit.jupiter.api.Assertions.*;

class LoginTest {

    private WebDriver driver;
    private WebDriverWait wait;

    @BeforeEach
    void setUp() {
        WebDriverManager.chromedriver().setup();
        driver = new ChromeDriver();
        driver.manage().window().maximize();
        wait = new WebDriverWait(driver, Duration.ofSeconds(10));
    }

    @AfterEach
    void tearDown() {
        if (driver != null) {
            driver.quit(); // Закрывает все окна и завершает сессию
        }
    }

    @Test
    @DisplayName("Успешная авторизация с валидными учётными данными")
    void testSuccessfulLogin() {
        driver.get("https://example.com/login");

        // Вводим логин
        wait.until(ExpectedConditions.visibilityOfElementLocated(By.id("email")))
            .sendKeys("user@example.com");

        // Вводим пароль
        driver.findElement(By.id("password")).sendKeys("SecurePass123");

        // Нажимаем кнопку входа
        driver.findElement(By.cssSelector("button[type='submit']")).click();

        // Проверяем успешный переход на dashboard
        wait.until(ExpectedConditions.urlContains("/dashboard"));
        WebElement welcomeMsg = wait.until(
            ExpectedConditions.visibilityOfElementLocated(By.cssSelector(".welcome-message"))
        );

        assertTrue(welcomeMsg.getText().contains("Добро пожаловать"));
    }
}
```

---

## Связь с тестированием

- **Selenium WebDriver** — фундамент UI-автоматизации. Все высокоуровневые фреймворки (Selenide, Serenity) построены поверх него.
- Позволяет автоматизировать **функциональные**, **регрессионные** и **smoke-тесты** веб-приложений.
- В CI/CD запускается в **headless-режиме** (без графического интерфейса) для ускорения.
- Скриншоты и логи помогают при **анализе падений** тестов.
- Правильный выбор локаторов напрямую влияет на **стабильность** тестов.

---

## Типичные ошибки

| Ошибка | Последствия | Правильный подход |
|--------|-------------|-------------------|
| Использование `Thread.sleep()` | Медленные, нестабильные тесты | Explicit Wait с `ExpectedConditions` |
| Смешивание implicit и explicit waits | Непредсказуемое время ожидания | Используйте только explicit waits |
| Абсолютные XPath (`/html/body/div[1]...`) | Тесты ломаются при любом изменении вёрстки | Относительные XPath или CSS Selectors |
| `driver.close()` вместо `driver.quit()` | Утечка процессов браузера | `quit()` в `@AfterEach` |
| Отсутствие ожиданий перед взаимодействием | `NoSuchElementException`, `ElementNotInteractableException` | Всегда ждите нужного состояния элемента |
| Хардкод тестовых данных в локаторах | Хрупкие тесты | Использовать `data-testid` атрибуты |
| Один огромный тестовый метод | Сложная отладка, непонятные ошибки | Один тест — один сценарий |
| Не закрывать драйвер в `finally` / `@AfterEach` | Накопление процессов, зависание CI | Гарантированное закрытие в tearDown |

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Что такое Selenium WebDriver и как он работает?
2. Перечислите основные типы локаторов. Какой из них самый надёжный?
3. В чём разница между `findElement` и `findElements`?
4. Какие виды ожиданий есть в Selenium? Почему `Thread.sleep` — плохая практика?
5. В чём разница между `driver.close()` и `driver.quit()`?
6. Как работать с выпадающим списком (`<select>`)?

### 🟡 Средний уровень
7. В чём разница между implicit и explicit wait? Можно ли их смешивать?
8. Что такое `StaleElementReferenceException` и как с ним бороться?
9. Как работать с iframe? Как переключаться между несколькими окнами?
10. Объясните разницу между CSS Selector и XPath. Когда что использовать?
11. Как сделать скриншот при падении теста автоматически?
12. Что такое `Actions` class? Приведите примеры использования.

### 🔴 Продвинутый уровень
13. Опишите архитектуру Selenium WebDriver. Что происходит при вызове `findElement`?
14. Как вы организуете параллельный запуск тестов с Selenium Grid / Selenoid?
15. Как бы вы обработали элемент, перекрытый другим элементом (overlay/spinner)?
16. Как написать свой custom `ExpectedCondition`?
17. Как минимизировать flaky-тесты при работе с Selenium?
18. В чём отличие протокола W3C WebDriver от старого JSON Wire Protocol?

---

## Практические задания

### Задание 1: Базовое взаимодействие
Напишите тест, который:
- Открывает `https://the-internet.herokuapp.com/login`
- Вводит логин `tomsmith` и пароль `SuperSecretPassword!`
- Нажимает Login
- Проверяет, что появилось сообщение об успешном входе

### Задание 2: Динамический контент
Напишите тест для `https://the-internet.herokuapp.com/dynamic_loading/1`:
- Нажмите кнопку Start
- Дождитесь появления текста "Hello World!" с помощью explicit wait
- Проверьте текст

### Задание 3: IFrame
Напишите тест для `https://the-internet.herokuapp.com/iframe`:
- Переключитесь в iframe текстового редактора
- Очистите содержимое и введите новый текст
- Проверьте, что текст отображается

### Задание 4: Множественные окна
Напишите тест для `https://the-internet.herokuapp.com/windows`:
- Кликните ссылку, открывающую новое окно
- Переключитесь на новое окно
- Проверьте заголовок
- Закройте новое окно и вернитесь в исходное

### Задание 5: Скриншот при ошибке
Реализуйте JUnit 5 Extension, который автоматически делает скриншот при падении теста и сохраняет его в папку `build/screenshots/`.

---

## Дополнительные ресурсы

- [Selenium Official Documentation](https://www.selenium.dev/documentation/)
- [W3C WebDriver Specification](https://www.w3.org/TR/webdriver2/)
- [WebDriverManager GitHub](https://github.com/bonigarcia/webdrivermanager)
- [The Internet — тестовое приложение для практики](https://the-internet.herokuapp.com/)
- [Selenium WebDriver Java API](https://www.selenium.dev/selenium/docs/api/java/)
- [CSS Selectors Reference (MDN)](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Selectors)
- [XPath Cheat Sheet](https://devhints.io/xpath)
