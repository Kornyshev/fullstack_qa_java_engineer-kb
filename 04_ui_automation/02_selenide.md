# Selenide

## Обзор

Selenide — это фреймворк-обёртка над Selenium WebDriver, созданный для упрощения написания UI-тестов на Java. Разработан Андреем Солнцевым (Codeborne, Эстония). Selenide решает большинство проблем, с которыми сталкиваются разработчики тестов при работе с чистым Selenium: ожидания, поиск элементов, управление браузером.

Главная философия Selenide — **лаконичность** и **стабильность**. Один и тот же тест на Selenide занимает в 2-3 раза меньше кода, чем на чистом Selenium, и при этом работает стабильнее благодаря встроенным умным ожиданиям.

---

## Selenide vs Selenium: сравнение

| Аспект | Selenium WebDriver | Selenide |
|--------|-------------------|----------|
| **Поиск элементов** | `driver.findElement(By.id("x"))` | `$("#x")` или `$(By.id("x"))` |
| **Коллекции** | `driver.findElements(By.css(".item"))` | `$$(".item")` |
| **Ожидания** | Ручные: `WebDriverWait`, `ExpectedConditions` | Автоматические: встроены в каждое действие |
| **Управление драйвером** | Ручное создание, закрытие | Автоматическое — Selenide всё делает сам |
| **Скриншоты** | Ручная реализация | Автоматически при падении теста |
| **Assertions** | Через JUnit/TestNG + ручные ожидания | Fluent API: `shouldHave`, `shouldBe` |
| **Кривая обучения** | Крутая | Пологая |
| **Гибкость** | Максимальная | Ограничена (но достаточна для 95% задач) |
| **Документация** | Обширная, но разрозненная | Компактная и понятная |

---

## Настройка проекта

### Maven-зависимости

```xml
<dependencies>
    <!-- Selenide (уже включает Selenium внутри) -->
    <dependency>
        <groupId>com.codeborne</groupId>
        <artifactId>selenide</artifactId>
        <version>7.6.1</version>
        <scope>test</scope>
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

> **Важно:** Selenide уже содержит Selenium WebDriver как транзитивную зависимость. Не нужно добавлять Selenium отдельно.

---

## Селекторы: $ и $$

### $ — поиск одного элемента

```java
import static com.codeborne.selenide.Selenide.*;
import static com.codeborne.selenide.Condition.*;

// По CSS Selector (самый частый вариант)
$("#loginBtn").click();
$(".error-message").shouldBe(visible);
$("[data-testid='submit']").click();

// По By-локатору
$(By.id("username")).setValue("admin");
$(By.xpath("//button[text()='Войти']")).click();

// По тексту (обёртка Selenide)
$(byText("Отправить")).click();
$(withText("Ошибка")).shouldBe(visible);

// Вложенный поиск
$("#loginForm").$("button[type='submit']").click();
```

### $$ — поиск коллекции элементов

```java
// Все элементы по селектору
ElementsCollection items = $$(".product-card");

// Проверка размера коллекции
$$(".product-card").shouldHave(size(10));
$$(".product-card").shouldHave(sizeGreaterThan(0));
$$(".product-card").shouldHave(sizeLessThanOrEqual(20));

// Фильтрация коллекции
$$(".product-card").filterBy(text("iPhone")).shouldHave(size(2));
$$(".product-card").excludeWith(cssClass("out-of-stock")).shouldHave(sizeGreaterThan(0));

// Получение конкретного элемента
$$(".product-card").first().shouldHave(text("Товар 1"));
$$(".product-card").last().click();
$$(".product-card").get(2).shouldBe(visible);

// Извлечение текстов всех элементов
List<String> texts = $$(".product-card .title").texts();

// Перебор коллекции
$$(".product-card").forEach(card -> {
    card.$(".price").shouldBe(visible);
});
```

---

## Fluent API: проверки (Conditions)

Selenide предоставляет выразительный API для проверок элементов. Все проверки **автоматически ожидают** выполнения условия (по умолчанию 4 секунды).

### Проверки видимости и состояния

```java
// Видимость
$(".modal").shouldBe(visible);
$(".spinner").shouldNotBe(visible);
$(".spinner").shouldBe(hidden);

// Существование в DOM (может быть скрыт)
$(".lazy-element").should(exist);
$(".removed-element").shouldNot(exist);

// Активность элемента
$("button#submit").shouldBe(enabled);
$("button#submit").shouldBe(disabled);

// Чекбокс / радиокнопка
$("input#agree").shouldBe(checked);
$("input#agree").shouldNotBe(checked);

// Фокус
$("input#search").shouldBe(focused);
```

### Проверки текста и значений

```java
// Точное совпадение текста
$(".title").shouldHave(exactText("Добро пожаловать"));

// Содержание текста (регистронезависимо)
$(".title").shouldHave(text("добро"));

// Значение поля ввода
$("input#email").shouldHave(value("user@example.com"));

// Атрибут
$("a.download").shouldHave(attribute("href", "/files/doc.pdf"));
$("div.item").shouldHave(attribute("data-id"));

// CSS-класс
$(".alert").shouldHave(cssClass("alert-danger"));

// CSS-значение
$(".header").shouldHave(cssValue("color", "rgba(255, 0, 0, 1)"));
```

### Проверки коллекций

```java
import static com.codeborne.selenide.CollectionCondition.*;

// Размер
$$(".item").shouldHave(size(5));
$$(".item").shouldHave(sizeGreaterThan(0));

// Тексты
$$(".menu-item").shouldHave(texts("Главная", "Каталог", "Контакты"));
$$(".menu-item").shouldHave(exactTexts("Главная", "Каталог", "О нас", "Контакты"));

// Наличие элемента с текстом
$$(".item").shouldHave(itemWithText("Молоко"));

// Пустая коллекция
$$(".error").shouldBe(CollectionCondition.empty);
```

---

## Основные действия

```java
// Ввод текста (очищает поле перед вводом)
$("input#name").setValue("Иван Иванов");

// Ввод без очистки (дописать к существующему)
$("input#search").append(" дополнение");

// Нажатие клавиш
$("input#search").pressEnter();
$("input#field").pressTab();
$("input#field").pressEscape();

// Клик
$("button#save").click();
$("button#save").doubleClick();
$("button#save").contextClick(); // Правый клик

// Hover
$(".menu-item").hover();

// Очистка поля
$("input#name").clear();

// Получение текста и значений
String text = $(".message").getText();
String value = $("input#email").getValue();
String attr = $("a.link").getAttribute("href");
String css = $(".box").getCssValue("background-color");

// Прокрутка к элементу
$("footer").scrollTo();
$(".far-element").scrollIntoView(true);  // true = вверх, false = вниз
```

---

## Конфигурация

### Программная конфигурация (Configuration class)

```java
import com.codeborne.selenide.Configuration;

// В @BeforeAll или @BeforeEach
Configuration.baseUrl = "https://example.com";       // Базовый URL
Configuration.browser = "chrome";                     // Браузер
Configuration.browserSize = "1920x1080";              // Размер окна
Configuration.headless = true;                        // Headless-режим
Configuration.timeout = 8000;                         // Таймаут ожидания (мс)
Configuration.pageLoadTimeout = 30000;                // Таймаут загрузки страницы
Configuration.screenshots = true;                     // Скриншоты при падении
Configuration.reportsFolder = "build/reports/tests";  // Папка для скриншотов
Configuration.holdBrowserOpen = false;                // Не держать браузер открытым
Configuration.downloadsFolder = "build/downloads";    // Папка для загрузок
```

### Конфигурация через selenide.properties

Создайте файл `src/test/resources/selenide.properties`:

```properties
selenide.baseUrl=https://example.com
selenide.browser=chrome
selenide.browserSize=1920x1080
selenide.headless=false
selenide.timeout=8000
selenide.reportsFolder=build/reports/tests
selenide.screenshots=true
selenide.pageLoadTimeout=30000
```

### Конфигурация через системные свойства (удобно для CI)

```bash
mvn test -Dselenide.baseUrl=https://staging.example.com \
         -Dselenide.browser=firefox \
         -Dselenide.headless=true
```

---

## Сравнение кода: Selenium vs Selenide

### Пример: Авторизация

**Selenium:**

```java
@Test
void loginTest() {
    WebDriverManager.chromedriver().setup();
    WebDriver driver = new ChromeDriver();
    WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));

    try {
        driver.get("https://example.com/login");
        driver.manage().window().maximize();

        wait.until(ExpectedConditions.visibilityOfElementLocated(By.id("email")))
            .sendKeys("user@example.com");

        driver.findElement(By.id("password")).sendKeys("password123");
        driver.findElement(By.cssSelector("button[type='submit']")).click();

        WebElement welcome = wait.until(
            ExpectedConditions.visibilityOfElementLocated(By.cssSelector(".welcome"))
        );
        assertTrue(welcome.getText().contains("Добро пожаловать"));
    } finally {
        driver.quit();
    }
}
```

**Selenide:**

```java
@Test
void loginTest() {
    open("/login");

    $("#email").setValue("user@example.com");
    $("#password").setValue("password123");
    $("button[type='submit']").click();

    $(".welcome").shouldHave(text("Добро пожаловать"));
}
```

> Разница очевидна: 20+ строк Selenium превращаются в 6 строк Selenide. При этом Selenide сам управляет драйвером, ожиданиями и скриншотами.

### Пример: Проверка списка товаров

**Selenium:**

```java
@Test
void checkProductList() {
    driver.get("https://example.com/products");
    WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));

    // Ждём загрузки списка
    wait.until(ExpectedConditions.numberOfElementsToBeMoreThan(
        By.cssSelector(".product-card"), 0
    ));

    List<WebElement> products = driver.findElements(By.cssSelector(".product-card"));
    assertEquals(12, products.size());

    // Проверяем, что все видимы
    for (WebElement product : products) {
        assertTrue(product.isDisplayed());
    }

    // Проверяем первый товар
    String firstTitle = products.get(0)
        .findElement(By.cssSelector(".title"))
        .getText();
    assertEquals("iPhone 15", firstTitle);
}
```

**Selenide:**

```java
@Test
void checkProductList() {
    open("/products");

    $$(".product-card").shouldHave(size(12));
    $$(".product-card").first().$(".title").shouldHave(exactText("iPhone 15"));
}
```

---

## Загрузка и скачивание файлов

### Загрузка файла (upload)

```java
// Загрузка файла через input[type="file"]
$("input[type='file']").uploadFile(new File("src/test/resources/test.pdf"));

// Загрузка из classpath
$("input[type='file']").uploadFromClasspath("test.pdf");

// Проверка успешной загрузки
$(".upload-status").shouldHave(text("Файл загружен"));
```

### Скачивание файла (download)

```java
// Скачивание файла
File downloadedFile = $("a.download-link").download();

// Проверка содержимого скачанного файла
assertThat(downloadedFile.getName()).isEqualTo("report.pdf");
assertThat(downloadedFile.length()).isGreaterThan(0);

// Скачивание с таймаутом
File file = $("a#export").download(30000); // 30 секунд
```

---

## Выполнение JavaScript

```java
// Выполнение JS-кода
executeJavaScript("document.querySelector('#hidden').style.display = 'block'");

// С возвращаемым значением
String title = executeJavaScript("return document.title");
Long scrollY = executeJavaScript("return window.scrollY");

// Прокрутка страницы
executeJavaScript("window.scrollTo(0, document.body.scrollHeight)");

// Клик через JS (когда обычный клик не работает из-за перекрытия)
executeJavaScript("arguments[0].click()", $("button#hidden"));

// Изменение значения поля
executeJavaScript("arguments[0].value = arguments[1]", $("input#date"), "2025-01-15");
```

---

## Selenide + Allure интеграция

### Maven-зависимости

```xml
<dependency>
    <groupId>io.qameta.allure</groupId>
    <artifactId>allure-selenide</artifactId>
    <version>2.29.1</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.qameta.allure</groupId>
    <artifactId>allure-junit5</artifactId>
    <version>2.29.1</version>
    <scope>test</scope>
</dependency>
```

### Настройка

```java
import com.codeborne.selenide.logevents.SelenideLogger;
import io.qameta.allure.selenide.AllureSelenide;
import org.junit.jupiter.api.BeforeAll;

class BaseTest {

    @BeforeAll
    static void setUpAllure() {
        // Подключаем Allure listener — все действия Selenide попадут в отчёт
        SelenideLogger.addListener("AllureSelenide",
            new AllureSelenide()
                .screenshots(true)          // Скриншоты при падении
                .savePageSource(true)        // Сохранять HTML-код страницы
                .includeSelenideSteps(true)  // Логировать шаги Selenide
        );
    }
}
```

### Тест с Allure-аннотациями

```java
import io.qameta.allure.*;

@Epic("Авторизация")
@Feature("Вход в систему")
class LoginTest extends BaseTest {

    @Test
    @Story("Успешный вход")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("Пользователь может войти с валидными данными")
    void successfulLogin() {
        open("/login");

        step("Ввести email", () ->
            $("#email").setValue("user@example.com"));

        step("Ввести пароль", () ->
            $("#password").setValue("password123"));

        step("Нажать кнопку входа", () ->
            $("button[type='submit']").click());

        step("Проверить приветственное сообщение", () ->
            $(".welcome").shouldHave(text("Добро пожаловать")));
    }
}
```

---

## Дополнительные команды Selenide

```java
// Навигация
open("https://example.com");      // Открыть URL
open("/relative-path");           // Относительно baseUrl
back();                           // Назад
forward();                        // Вперёд
refresh();                        // Обновить страницу

// Информация о странице
String url = url();               // Текущий URL
String title = title();           // Заголовок страницы
String source = source();         // HTML-код страницы

// Переключение контекста
switchTo().frame("frameName");    // Переключиться в iframe
switchTo().defaultContent();      // Вернуться из iframe
switchTo().window(1);             // Переключиться на окно по индексу
switchTo().alert().accept();      // Принять alert

// Работа с cookies
WebDriverRunner.getWebDriver().manage().addCookie(new Cookie("token", "abc123"));
WebDriverRunner.getWebDriver().manage().deleteAllCookies();

// Доступ к оригинальному WebDriver (когда нужна полная гибкость Selenium)
WebDriver driver = WebDriverRunner.getWebDriver();
```

---

## Связь с тестированием

- Selenide — наиболее популярный фреймворк для UI-автоматизации в Java-проектах на территории СНГ.
- Значительно **снижает порог входа** в автоматизацию для ручных тестировщиков.
- Встроенные умные ожидания **устраняют** большинство причин flaky-тестов.
- Отлично интегрируется с **Allure** для генерации наглядных отчётов.
- Позволяет сосредоточиться на **бизнес-логике** тестов, а не на борьбе с инфраструктурой.

---

## Типичные ошибки

| Ошибка | Последствия | Правильный подход |
|--------|-------------|-------------------|
| Использование `getText()` вместо `shouldHave(text(...))` | Отсутствие автоматического ожидания | Всегда используйте `shouldHave` / `shouldBe` для проверок |
| Игнорирование `Configuration` | Дефолтные настройки не подходят для проекта | Настройте `baseUrl`, `timeout`, `browserSize` |
| Обращение к Selenium API вместо Selenide | Потеря преимуществ Selenide | `$` и `$$` вместо `driver.findElement` |
| `$(".item").isDisplayed()` для проверки | Не ждёт, возвращает boolean сразу | `$(".item").shouldBe(visible)` |
| Не подключён Allure listener | Скриншоты не попадают в отчёт | `SelenideLogger.addListener(...)` в `@BeforeAll` |
| Хардкод абсолютных URL | Нельзя переключаться между окружениями | Используйте `Configuration.baseUrl` + `open("/path")` |
| Не закрывают браузер между тестами | Утечки ресурсов (хотя Selenide делает это сам) | Selenide закрывает браузер автоматически — не мешайте ему |

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Что такое Selenide и зачем он нужен?
2. Чем `$` отличается от `$$`?
3. Как в Selenide реализованы ожидания? В чём отличие от Selenium?
4. Какие проверки (`Condition`) вы знаете?
5. Как настроить базовый URL и таймаут?
6. Как загрузить/скачать файл в Selenide?

### 🟡 Средний уровень
7. Сравните написание теста на Selenium и Selenide. Какие преимущества у Selenide?
8. Как интегрировать Selenide с Allure? Что даёт эта интеграция?
9. Как работать с коллекциями элементов? Приведите примеры фильтрации.
10. Как переключаться между iframe и окнами в Selenide?
11. Как выполнить JavaScript в Selenide? Когда это нужно?
12. Как настроить Selenide для запуска в CI (headless, remote)?

### 🔴 Продвинутый уровень
13. Как Selenide обрабатывает `StaleElementReferenceException` по сравнению с чистым Selenium?
14. Как написать свой собственный `Condition` для Selenide?
15. Как настроить Selenide для работы с Selenoid / Selenium Grid?
16. Как организовать Page Object с Selenide? Нужен ли `PageFactory`?
17. Какие ограничения есть у Selenide по сравнению с чистым Selenium?
18. Как Selenide работает под капотом? Объясните механизм проксирования элементов.

---

## Практические задания

### Задание 1: Миграция теста
Перепишите следующий тест с Selenium на Selenide:
```java
WebDriver driver = new ChromeDriver();
driver.get("https://the-internet.herokuapp.com/checkboxes");
List<WebElement> checkboxes = driver.findElements(By.cssSelector("input[type='checkbox']"));
if (!checkboxes.get(0).isSelected()) {
    checkboxes.get(0).click();
}
assertTrue(checkboxes.get(0).isSelected());
driver.quit();
```

### Задание 2: Работа с коллекциями
Напишите тест, который открывает страницу с таблицей (`https://the-internet.herokuapp.com/tables`), находит все строки первой таблицы, проверяет количество строк и извлекает текст ячеек второго столбца.

### Задание 3: Конфигурация
Создайте базовый тестовый класс с настройкой Selenide через `selenide.properties` и Allure-интеграцией. Напишите два теста, наследующих от этого класса.

### Задание 4: Собственный Condition
Напишите кастомный `Condition`, который проверяет, что числовой текст элемента больше заданного значения. Например: `$(".price").shouldHave(numericValueGreaterThan(100))`.

---

## Дополнительные ресурсы

- [Selenide Official Site](https://selenide.org/)
- [Selenide GitHub](https://github.com/selenide/selenide)
- [Selenide Wiki](https://github.com/selenide/selenide/wiki)
- [Selenide Javadoc](https://selenide.org/javadoc.html)
- [Блог Андрея Солнцева](https://asolntsev.github.io/)
- [Selenide + Allure Integration Guide](https://github.com/selenide/selenide/wiki/Allure)
- [Selenide YouTube канал](https://www.youtube.com/@selenide)
