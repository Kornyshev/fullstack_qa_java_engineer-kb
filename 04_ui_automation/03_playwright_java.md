# Playwright для Java

## Обзор

Playwright — это современный фреймворк для end-to-end тестирования веб-приложений, разработанный компанией Microsoft. Изначально создан для JavaScript/TypeScript, но имеет полноценные биндинги для Java. Playwright предлагает принципиально иной подход к автоматизации по сравнению с Selenium: он работает через DevTools Protocol (CDP) и собственные протоколы, а не через WebDriver, что обеспечивает более высокую скорость и надёжность.

Ключевые преимущества Playwright: встроенный auto-wait, изолированные browser contexts, перехват сети, трассировка, запись видео и поддержка нескольких браузеров «из коробки».

---

## Архитектура Playwright

### Как Playwright отличается от Selenium

```
Selenium:
┌────────┐  HTTP/JSON  ┌──────────────┐  Native  ┌─────────┐
│  Тест  │ ──────────► │ ChromeDriver │ ───────► │ Chrome  │
└────────┘             └──────────────┘          └─────────┘

Playwright:
┌────────┐  WebSocket  ┌────────────────────┐  CDP/протокол  ┌─────────┐
│  Тест  │ ──────────► │ Playwright Server   │ ─────────────► │ Chrome  │
└────────┘             └────────────────────┘                └─────────┘
```

Playwright использует **прямое подключение** к браузеру через WebSocket и CDP (Chrome DevTools Protocol), что даёт:
- Более быструю передачу команд (один постоянный WebSocket вместо HTTP-запросов)
- Доступ к сетевому слою (перехват запросов, мониторинг)
- Управление несколькими страницами и контекстами в рамках одного процесса

### Browser, Context, Page

```
Browser (один процесс браузера)
├── BrowserContext 1 (изолированная сессия: cookies, localStorage)
│   ├── Page 1
│   └── Page 2
├── BrowserContext 2 (другая сессия, без пересечения с Context 1)
│   └── Page 3
└── ...
```

- **Browser** — экземпляр браузера (Chromium, Firefox, WebKit).
- **BrowserContext** — изолированная сессия. Каждый контекст имеет свои cookies, localStorage, авторизацию. Идеально для параллельных тестов.
- **Page** — отдельная вкладка или всплывающее окно.

---

## Настройка проекта

### Maven-зависимости

```xml
<dependencies>
    <!-- Playwright для Java -->
    <dependency>
        <groupId>com.microsoft.playwright</groupId>
        <artifactId>playwright</artifactId>
        <version>1.49.0</version>
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

### Установка браузеров

```bash
# Playwright сам скачивает нужные браузеры
mvn exec:java -e -D exec.mainClass=com.microsoft.playwright.CLI -D exec.args="install"

# Или только конкретный браузер
mvn exec:java -e -D exec.mainClass=com.microsoft.playwright.CLI -D exec.args="install chromium"
```

### Базовый тестовый класс

```java
import com.microsoft.playwright.*;
import org.junit.jupiter.api.*;

class BasePlaywrightTest {

    // Общие для всех тестов в классе
    static Playwright playwright;
    static Browser browser;

    // Изолированные для каждого теста
    BrowserContext context;
    Page page;

    @BeforeAll
    static void launchBrowser() {
        playwright = Playwright.create();
        browser = playwright.chromium().launch(
            new BrowserType.LaunchOptions()
                .setHeadless(true)       // Без GUI
                .setSlowMo(0)            // Замедление (для отладки: 100-500 мс)
        );
    }

    @AfterAll
    static void closeBrowser() {
        browser.close();
        playwright.close();
    }

    @BeforeEach
    void createContext() {
        context = browser.newContext(
            new Browser.NewContextOptions()
                .setViewportSize(1920, 1080)
                .setLocale("ru-RU")
        );
        page = context.newPage();
    }

    @AfterEach
    void closeContext() {
        context.close(); // Закрывает все страницы в контексте
    }
}
```

---

## Локаторы

Playwright предоставляет семантические локаторы, привязанные к ролям и текстам, что делает тесты более устойчивыми к изменениям вёрстки.

### Рекомендуемые локаторы (по приоритету)

```java
// 1. getByRole — лучший вариант, основан на accessibility-ролях
page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Войти")).click();
page.getByRole(AriaRole.LINK, new Page.GetByRoleOptions().setName("Регистрация")).click();
page.getByRole(AriaRole.HEADING, new Page.GetByRoleOptions().setName("Добро пожаловать")).isVisible();
page.getByRole(AriaRole.TEXTBOX, new Page.GetByRoleOptions().setName("Email")).fill("user@example.com");
page.getByRole(AriaRole.CHECKBOX, new Page.GetByRoleOptions().setName("Запомнить меня")).check();

// 2. getByTestId — по атрибуту data-testid (надёжный, но требует разработки)
page.getByTestId("submit-button").click();
page.getByTestId("login-form").isVisible();

// 3. getByText — по видимому тексту
page.getByText("Добро пожаловать").isVisible();
page.getByText("Подробнее", new Page.GetByTextOptions().setExact(true)).click();

// 4. getByLabel — по тексту label (для форм)
page.getByLabel("Email").fill("user@example.com");
page.getByLabel("Пароль").fill("secret");

// 5. getByPlaceholder — по тексту placeholder
page.getByPlaceholder("Введите имя").fill("Иван");

// 6. getByAltText — по alt-тексту изображений
page.getByAltText("Логотип компании").click();

// 7. getByTitle — по атрибуту title
page.getByTitle("Закрыть").click();
```

### CSS и XPath локаторы

```java
// CSS Selector
page.locator("#loginBtn").click();
page.locator(".product-card").first().click();
page.locator("[data-testid='menu']").isVisible();

// XPath
page.locator("xpath=//button[text()='Отправить']").click();

// Комбинирование локаторов (фильтрация)
page.locator(".product-card")
    .filter(new Locator.FilterOptions().setHasText("iPhone"))
    .first()
    .click();

// Вложенный поиск
page.locator("#sidebar").locator(".menu-item").nth(2).click();

// Фильтрация по дочернему элементу
page.locator(".card")
    .filter(new Locator.FilterOptions().setHas(page.locator(".price", new Page.LocatorOptions())))
    .first();
```

---

## Auto-Wait механизм

Одно из главных преимуществ Playwright — **автоматическое ожидание**. Перед выполнением действия Playwright проверяет, что элемент:

1. **Attached** — присутствует в DOM
2. **Visible** — видим на странице
3. **Stable** — не анимируется
4. **Enabled** — не заблокирован (не disabled)
5. **Receives Events** — не перекрыт другим элементом

Это означает, что в большинстве случаев вам **не нужно** писать явные ожидания:

```java
// Playwright сам дождётся, пока кнопка станет кликабельной
page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Сохранить")).click();

// Если нужно настроить таймаут для конкретного действия
page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Загрузить"))
    .click(new Locator.ClickOptions().setTimeout(30000)); // 30 секунд

// Ожидание конкретного состояния
page.locator("#spinner").waitFor(new Locator.WaitForOptions()
    .setState(WaitForSelectorState.HIDDEN)
    .setTimeout(15000));

// Ожидание навигации
page.waitForURL("**/dashboard");
page.waitForURL(url -> url.contains("/success"));
```

---

## Assertions (Проверки)

Playwright предоставляет assertions с автоматическим ожиданием (авто-retry до таймаута):

```java
import static com.microsoft.playwright.assertions.PlaywrightAssertions.assertThat;

// Проверка видимости
assertThat(page.getByText("Добро пожаловать")).isVisible();
assertThat(page.locator("#spinner")).isHidden();

// Проверка текста
assertThat(page.locator(".title")).hasText("Главная страница");
assertThat(page.locator(".title")).containsText("Главная");

// Проверка значения поля
assertThat(page.getByLabel("Email")).hasValue("user@example.com");
assertThat(page.getByLabel("Email")).isEmpty();

// Проверка атрибутов
assertThat(page.locator("a.download")).hasAttribute("href", "/files/doc.pdf");

// Проверка CSS-классов
assertThat(page.locator(".alert")).hasClass("alert alert-danger");

// Проверка количества элементов
assertThat(page.locator(".product-card")).hasCount(12);

// Проверка URL и title страницы
assertThat(page).hasURL("https://example.com/dashboard");
assertThat(page).hasTitle("Dashboard");

// Проверка состояния
assertThat(page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Отправить"))).isEnabled();
assertThat(page.getByRole(AriaRole.CHECKBOX)).isChecked();

// Установка таймаута для конкретной проверки
assertThat(page.locator(".result")).hasText("Готово",
    new LocatorAssertions.HasTextOptions().setTimeout(20000));
```

---

## Перехват сети (Network Interception)

Playwright позволяет перехватывать и модифицировать сетевые запросы — мощный инструмент для тестирования:

```java
// Перехват и подмена ответа API
page.route("**/api/products", route -> {
    // Возвращаем мок-данные вместо реального ответа
    route.fulfill(new Route.FulfillOptions()
        .setStatus(200)
        .setContentType("application/json")
        .setBody("[{\"id\": 1, \"name\": \"Мок-товар\", \"price\": 999}]")
    );
});

// Блокировка запросов (например, аналитика или реклама)
page.route("**/*.{png,jpg,jpeg}", Route::abort);
page.route("**/analytics/**", Route::abort);

// Модификация запроса
page.route("**/api/data", route -> {
    // Продолжить запрос с модифицированными заголовками
    route.resume(new Route.ResumeOptions()
        .setHeaders(Map.of("Authorization", "Bearer test-token"))
    );
});

// Ожидание конкретного сетевого запроса
Response response = page.waitForResponse("**/api/products", () -> {
    page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Загрузить")).click();
});
assertEquals(200, response.status());

// Мониторинг запросов
page.onRequest(request -> System.out.println(">> " + request.method() + " " + request.url()));
page.onResponse(response -> System.out.println("<< " + response.status() + " " + response.url()));
```

---

## Трассировка (Tracing)

Трассировка записывает все действия, сетевую активность, скриншоты и DOM-снимки. Это бесценный инструмент для отладки падений:

```java
@BeforeEach
void setUp() {
    context = browser.newContext();

    // Начинаем трассировку
    context.tracing().start(new Tracing.StartOptions()
        .setScreenshots(true)    // Скриншоты на каждом шаге
        .setSnapshots(true)      // DOM-снимки
        .setSources(true)        // Исходный код тестов
    );

    page = context.newPage();
}

@AfterEach
void tearDown(TestInfo testInfo) {
    // Сохраняем трассировку (можно условно — только при падении)
    context.tracing().stop(new Tracing.StopOptions()
        .setPath(Path.of("traces/" + testInfo.getDisplayName() + ".zip"))
    );
    context.close();
}
```

Просмотр трассировки:

```bash
# Открывает Trace Viewer — интерактивный инструмент для просмотра
mvn exec:java -e -D exec.mainClass=com.microsoft.playwright.CLI -D exec.args="show-trace traces/test-trace.zip"

# Или через онлайн-сервис
# https://trace.playwright.dev
```

---

## Скриншоты и видео

### Скриншоты

```java
// Скриншот всей страницы
page.screenshot(new Page.ScreenshotOptions()
    .setPath(Path.of("screenshots/full_page.png"))
    .setFullPage(true));   // Вся страница, включая скролл

// Скриншот видимой области
page.screenshot(new Page.ScreenshotOptions()
    .setPath(Path.of("screenshots/viewport.png")));

// Скриншот конкретного элемента
page.locator(".error-panel").screenshot(new Locator.ScreenshotOptions()
    .setPath(Path.of("screenshots/error.png")));

// Скриншот как массив байт
byte[] bytes = page.screenshot();
```

### Видеозапись

```java
// Включается на уровне BrowserContext
context = browser.newContext(new Browser.NewContextOptions()
    .setRecordVideoDir(Path.of("videos/"))
    .setRecordVideoSize(1280, 720)
);

page = context.newPage();

// Действия в тесте...
page.navigate("https://example.com");
page.getByLabel("Email").fill("test@test.com");

// Видео сохраняется при закрытии контекста
page.close();
Path videoPath = page.video().path();
context.close();
```

---

## Поддержка нескольких браузеров

```java
// Chromium (Chrome, Edge)
Browser chromium = playwright.chromium().launch();

// Firefox
Browser firefox = playwright.firefox().launch();

// WebKit (Safari)
Browser webkit = playwright.webkit().launch();

// Параметризованный тест для всех браузеров
@ParameterizedTest
@ValueSource(strings = {"chromium", "firefox", "webkit"})
void crossBrowserTest(String browserType) {
    Browser browser = switch (browserType) {
        case "chromium" -> playwright.chromium().launch();
        case "firefox"  -> playwright.firefox().launch();
        case "webkit"   -> playwright.webkit().launch();
        default -> throw new IllegalArgumentException("Неизвестный браузер: " + browserType);
    };

    Page page = browser.newPage();
    page.navigate("https://example.com");
    assertThat(page).hasTitle("Example Domain");
    browser.close();
}
```

---

## Полный пример теста

```java
import com.microsoft.playwright.*;
import com.microsoft.playwright.options.AriaRole;
import org.junit.jupiter.api.*;

import static com.microsoft.playwright.assertions.PlaywrightAssertions.assertThat;

class LoginPlaywrightTest {

    static Playwright playwright;
    static Browser browser;
    BrowserContext context;
    Page page;

    @BeforeAll
    static void init() {
        playwright = Playwright.create();
        browser = playwright.chromium().launch(
            new BrowserType.LaunchOptions().setHeadless(true)
        );
    }

    @AfterAll
    static void cleanup() {
        browser.close();
        playwright.close();
    }

    @BeforeEach
    void setUp() {
        context = browser.newContext();
        page = context.newPage();
    }

    @AfterEach
    void tearDown() {
        context.close();
    }

    @Test
    @DisplayName("Успешная авторизация")
    void successfulLogin() {
        page.navigate("https://example.com/login");

        // Заполняем форму, используя семантические локаторы
        page.getByLabel("Email").fill("user@example.com");
        page.getByLabel("Пароль").fill("password123");
        page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Войти")).click();

        // Проверяем успешный вход
        assertThat(page).hasURL("https://example.com/dashboard");
        assertThat(page.getByRole(AriaRole.HEADING)).hasText("Добро пожаловать");
    }

    @Test
    @DisplayName("Ошибка при неверном пароле")
    void loginWithInvalidPassword() {
        page.navigate("https://example.com/login");

        page.getByLabel("Email").fill("user@example.com");
        page.getByLabel("Пароль").fill("wrongpass");
        page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Войти")).click();

        // Проверяем сообщение об ошибке
        assertThat(page.locator(".error-message")).hasText("Неверный email или пароль");
        assertThat(page).hasURL("https://example.com/login");
    }
}
```

---

## Сравнение: Playwright vs Selenium vs Selenide

| Критерий | Selenium | Selenide | Playwright |
|----------|----------|----------|------------|
| **Протокол** | W3C WebDriver (HTTP) | W3C WebDriver (через Selenium) | CDP / собственный (WebSocket) |
| **Скорость** | Средняя | Средняя | Высокая |
| **Auto-wait** | Нет (ручные ожидания) | Да (в assertions) | Да (во всех действиях) |
| **Изоляция тестов** | Новый драйвер на тест | Selenide управляет | Browser Contexts |
| **Сеть** | Нет доступа | Нет доступа | Перехват, мок, мониторинг |
| **Трассировка** | Нет | Скриншоты | Полная: скриншоты, DOM, сеть, видео |
| **Браузеры** | Chrome, Firefox, Edge, Safari | Через Selenium | Chromium, Firefox, WebKit |
| **Параллелизм** | Сложная настройка (Grid) | Через JUnit/TestNG | Встроен через contexts |
| **Порог входа** | Средний | Низкий | Средний |
| **Экосистема Java** | Огромная | Большая (СНГ) | Растущая |
| **Мобильное тестирование** | Через Appium | Нет | Эмуляция мобильных устройств |

---

## Связь с тестированием

- Playwright — **современная альтернатива** Selenium для end-to-end тестирования.
- Встроенный auto-wait **драматически снижает** количество flaky-тестов.
- Перехват сети позволяет создавать **изолированные** тесты без зависимости от бэкенда.
- Browser Contexts обеспечивают **параллельный запуск** без дополнительной инфраструктуры.
- Трассировка значительно **упрощает отладку** падений в CI/CD.
- Поддержка видеозаписи — ценный артефакт для **анализа дефектов**.

---

## Типичные ошибки

| Ошибка | Последствия | Правильный подход |
|--------|-------------|-------------------|
| Не закрывать `Playwright` / `Browser` | Утечка процессов | `@AfterAll` с `browser.close()` и `playwright.close()` |
| Использовать один Context для всех тестов | Тесты влияют друг на друга (cookies, state) | Новый `BrowserContext` для каждого теста |
| CSS/XPath вместо семантических локаторов | Хрупкие тесты | `getByRole`, `getByLabel`, `getByTestId` |
| Игнорировать трассировку | Сложная отладка CI-падений | Включить `tracing` хотя бы для retry-попыток |
| Не установить браузеры перед запуском | `BrowserType.launch: Executable doesn't exist` | `playwright install` в CI-скриптах |
| Переносить паттерны Selenium в Playwright | Лишний код, упущены возможности | Изучить API Playwright, использовать его идиомы |

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Что такое Playwright и чем он отличается от Selenium?
2. Что такое Browser Context в Playwright?
3. Какие локаторы рекомендуются в Playwright? Почему `getByRole` предпочтительнее?
4. Как работает auto-wait в Playwright?
5. Как установить Playwright и браузеры для Java-проекта?

### 🟡 Средний уровень
6. Как перехватить API-запрос и вернуть мок-данные?
7. Как организовать параллельный запуск тестов с Playwright?
8. Что такое трассировка и как её использовать?
9. Как записать видео выполнения теста?
10. Сравните Playwright и Selenide: когда что выбрать?

### 🔴 Продвинутый уровень
11. Как Playwright обеспечивает стабильность тестов на уровне архитектуры?
12. Как тестировать приложение с Server-Sent Events или WebSocket через Playwright?
13. Как интегрировать Playwright Java с Allure?
14. Как эмулировать мобильное устройство в Playwright?
15. Какие ограничения имеет Playwright для Java по сравнению с TypeScript-версией?

---

## Практические задания

### Задание 1: Первый тест
Настройте Maven-проект с Playwright для Java. Напишите тест, который открывает `https://example.com`, проверяет заголовок страницы и делает скриншот.

### Задание 2: Мок API
Напишите тест, который перехватывает запрос к API и подставляет мок-данные. Проверьте, что UI отображает именно мок-данные.

### Задание 3: Кроссбраузерное тестирование
Создайте параметризованный тест, который запускается в трёх браузерах (Chromium, Firefox, WebKit) и проверяет одну и ту же функциональность.

### Задание 4: Трассировка и отладка
Напишите тест с включённой трассировкой. Намеренно сделайте тест падающим. Откройте трассировку в Trace Viewer и найдите шаг, на котором произошла ошибка.

### Задание 5: Сравнение фреймворков
Реализуйте один и тот же E2E-сценарий (регистрация + авторизация + действие + проверка) на Selenium, Selenide и Playwright. Сравните количество кода, читаемость и стабильность.

---

## Дополнительные ресурсы

- [Playwright for Java — Official Docs](https://playwright.dev/java/docs/intro)
- [Playwright GitHub](https://github.com/microsoft/playwright-java)
- [Playwright Trace Viewer](https://trace.playwright.dev/)
- [Playwright Locators Guide](https://playwright.dev/java/docs/locators)
- [Playwright API Reference (Java)](https://playwright.dev/java/docs/api/class-playwright)
- [Playwright Best Practices](https://playwright.dev/java/docs/best-practices)
