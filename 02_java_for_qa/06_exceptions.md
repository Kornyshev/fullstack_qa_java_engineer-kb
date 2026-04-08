# Исключения в Java

## Обзор

Механизм исключений (Exceptions) — это фундаментальная часть Java, обеспечивающая обработку ошибок
и нештатных ситуаций в программе. Для QA-инженера понимание исключений критически важно:
при автоматизации тестирования мы постоянно сталкиваемся с `NullPointerException`,
`NoSuchElementException`, `TimeoutException` и другими. Умение правильно обрабатывать исключения
и создавать собственные — ключевой навык для написания надёжных и читаемых тестов.

---

## Иерархия исключений

```
                        Throwable
                       /         \
                    Error       Exception
                   /    \       /         \
          OutOfMemoryError  IOException  RuntimeException
          StackOverflowError            /       |        \
                              NullPointer  IndexOutOf  IllegalArgument
                              Exception    BoundsExc.  Exception
```

### Throwable — корень иерархии

Все исключения и ошибки наследуются от `java.lang.Throwable`. Только объекты типа `Throwable`
(и его наследников) можно бросать (`throw`) и ловить (`catch`).

### Error — фатальные ошибки JVM

```java
// Error — ошибки, которые обычно НЕ нужно обрабатывать
// Они сигнализируют о критических проблемах среды выполнения

// OutOfMemoryError — нехватка памяти
// StackOverflowError — переполнение стека (бесконечная рекурсия)
// NoClassDefFoundError — класс не найден во время выполнения
```

> **Для QA:** Error'ы обычно не ловят. Но если тест падает с `OutOfMemoryError`,
> это серьёзный баг, который нужно зарепортить с соответствующим приоритетом.

### Exception — исключения, которые можно и нужно обрабатывать

Делятся на две категории: **checked** и **unchecked**.

---

## Checked vs Unchecked исключения

### Сравнительная таблица

| Критерий | Checked (проверяемые) | Unchecked (непроверяемые) |
|---|---|---|
| Наследование | `Exception` (не `RuntimeException`) | `RuntimeException` и его потомки |
| Проверка компилятором | Да, обязательно обработать или объявить | Нет, компилятор не требует обработки |
| Когда возникают | Внешние обстоятельства (IO, сеть) | Ошибки программиста (логика, null) |
| `throws` в сигнатуре | Обязателен, если не ловим | Не обязателен |
| Типичные примеры | `IOException`, `SQLException`, `FileNotFoundException` | `NullPointerException`, `IllegalArgumentException`, `IndexOutOfBoundsException` |
| Философия | "Восстановимые" ошибки | "Программистские" ошибки |

### Checked — проверяемые исключения

```java
// Компилятор ТРЕБУЕТ обработки checked-исключений

// Вариант 1: обработать через try-catch
public String readTestData(String filePath) {
    try {
        return Files.readString(Path.of(filePath));
    } catch (IOException e) {
        throw new RuntimeException("Не удалось прочитать тестовые данные: " + filePath, e);
    }
}

// Вариант 2: пробросить через throws
public String readTestDataThrows(String filePath) throws IOException {
    return Files.readString(Path.of(filePath));
}
```

### Unchecked — непроверяемые исключения

```java
// Компилятор НЕ требует обработки — они наследуют RuntimeException

public WebElement findButton(String text) {
    // NullPointerException — если driver == null
    // NoSuchElementException — если элемент не найден (unchecked в Selenium)
    return driver.findElement(By.xpath("//button[text()='" + text + "']"));
}
```

---

## Конструкции обработки исключений

### try-catch-finally

```java
// Базовая конструкция обработки исключений
WebDriver driver = null;
try {
    driver = new ChromeDriver();
    driver.get("https://example.com");
    // тестовые действия...
} catch (WebDriverException e) {
    // Обработка ошибки WebDriver
    System.err.println("Ошибка WebDriver: " + e.getMessage());
    throw e; // пробрасываем дальше, чтобы тест упал
} finally {
    // finally выполняется ВСЕГДА, даже если было исключение
    if (driver != null) {
        driver.quit();
    }
}
```

### Множественный catch (Multi-catch)

```java
// Несколько catch-блоков (от частного к общему)
try {
    WebElement element = driver.findElement(By.id("submit-btn"));
    element.click();
} catch (NoSuchElementException e) {
    // Элемент не найден на странице
    logger.error("Кнопка submit не найдена: {}", e.getMessage());
    takeScreenshot("submit_not_found");
} catch (ElementNotInteractableException e) {
    // Элемент найден, но не кликабелен
    logger.error("Кнопка submit не интерактивна: {}", e.getMessage());
    takeScreenshot("submit_not_interactable");
} catch (WebDriverException e) {
    // Общая ошибка WebDriver (ловит всё остальное)
    logger.error("Ошибка WebDriver: {}", e.getMessage());
}

// Multi-catch (Java 7+) — один блок для нескольких типов исключений
try {
    performAction();
} catch (NoSuchElementException | TimeoutException e) {
    // Обработка обоих типов одинаково
    logger.error("Элемент недоступен: {}", e.getMessage());
    takeScreenshot("element_unavailable");
}
```

### try-with-resources (Java 7+)

```java
// Автоматическое закрытие ресурсов, реализующих AutoCloseable
// Ресурс закрывается автоматически после блока try

// Чтение тестовых данных из файла
try (BufferedReader reader = new BufferedReader(new FileReader("test_data.csv"))) {
    String line;
    while ((line = reader.readLine()) != null) {
        processTestData(line);
    }
} catch (IOException e) {
    throw new RuntimeException("Ошибка чтения тестовых данных", e);
}
// reader.close() вызывается автоматически

// Несколько ресурсов
try (
    InputStream input = new FileInputStream("config.properties");
    Connection conn = DriverManager.getConnection(dbUrl)
) {
    // Используем оба ресурса
    // Оба будут закрыты автоматически (в обратном порядке)
} catch (IOException | SQLException e) {
    logger.error("Ошибка инициализации: {}", e.getMessage());
}

// Применение в тестах — WebDriver как AutoCloseable (обёртка)
public class DriverResource implements AutoCloseable {
    private final WebDriver driver;

    public DriverResource() {
        this.driver = new ChromeDriver();
    }

    public WebDriver getDriver() {
        return driver;
    }

    @Override
    public void close() {
        if (driver != null) {
            driver.quit();
        }
    }
}

// Использование
try (DriverResource resource = new DriverResource()) {
    WebDriver driver = resource.getDriver();
    driver.get("https://example.com");
    // тест...
} // driver.quit() вызывается автоматически
```

---

## Создание собственных исключений

### Custom Exceptions в тестовом фреймворке

```java
// Базовое исключение фреймворка
public class TestFrameworkException extends RuntimeException {

    public TestFrameworkException(String message) {
        super(message);
    }

    public TestFrameworkException(String message, Throwable cause) {
        super(message, cause);
    }
}

// Исключение при ошибке конфигурации тестов
public class TestConfigurationException extends TestFrameworkException {

    public TestConfigurationException(String property) {
        super("Отсутствует обязательная конфигурация: " + property);
    }
}

// Исключение при ошибке подготовки тестовых данных
public class TestDataException extends TestFrameworkException {

    private final String dataSource;

    public TestDataException(String dataSource, String message) {
        super("Ошибка тестовых данных [" + dataSource + "]: " + message);
        this.dataSource = dataSource;
    }

    public String getDataSource() {
        return dataSource;
    }
}

// Исключение при несовпадении ожиданий на странице
public class PageValidationException extends TestFrameworkException {

    private final String pageName;
    private final String expectedState;
    private final String actualState;

    public PageValidationException(String pageName, String expectedState, String actualState) {
        super(String.format("Страница '%s': ожидалось '%s', получено '%s'",
                pageName, expectedState, actualState));
        this.pageName = pageName;
        this.expectedState = expectedState;
        this.actualState = actualState;
    }

    // Геттеры для доступа к деталям ошибки в отчётах
    public String getPageName() { return pageName; }
    public String getExpectedState() { return expectedState; }
    public String getActualState() { return actualState; }
}
```

### Использование собственных исключений

```java
public class LoginPage {

    public void login(String username, String password) {
        if (username == null || username.isBlank()) {
            throw new TestDataException("LoginPage", "username не может быть пустым");
        }

        WebElement loginForm = findLoginForm();
        if (loginForm == null) {
            throw new PageValidationException("LoginPage", "Форма логина видна", "Форма не найдена");
        }

        // выполняем логин...
    }

    private WebElement findLoginForm() {
        try {
            return driver.findElement(By.id("login-form"));
        } catch (NoSuchElementException e) {
            return null;
        }
    }
}
```

---

## Частые исключения, с которыми сталкивается QA

### NullPointerException (NPE)

```java
// Самое частое исключение — обращение к null-ссылке
String title = driver.getTitle(); // NPE, если driver == null

// Как избежать:
// 1. Проверка на null
if (driver != null) {
    String title = driver.getTitle();
}

// 2. Optional (предпочтительно)
Optional.ofNullable(driver)
        .map(WebDriver::getTitle)
        .orElse("Driver не инициализирован");

// 3. Objects.requireNonNull — fail-fast
public void setDriver(WebDriver driver) {
    this.driver = Objects.requireNonNull(driver, "WebDriver не может быть null");
}
```

### NoSuchElementException (Selenium)

```java
// Элемент не найден на странице
try {
    driver.findElement(By.id("non-existent"));
} catch (NoSuchElementException e) {
    // Элемент не найден — возможно, страница не загрузилась
    // или локатор некорректный
}

// Лучше использовать явное ожидание вместо перехвата
WebElement element = new WebDriverWait(driver, Duration.ofSeconds(10))
        .until(ExpectedConditions.presenceOfElementLocated(By.id("target")));
```

### TimeoutException

```java
// Превышено время ожидания
try {
    new WebDriverWait(driver, Duration.ofSeconds(5))
            .until(ExpectedConditions.visibilityOfElementLocated(By.id("loading-spinner")));
} catch (TimeoutException e) {
    logger.warn("Элемент не появился за 5 секунд");
    takeScreenshot("timeout_spinner");
    throw new PageValidationException("Dashboard", "Спиннер виден", "Спиннер не появился за 5с");
}
```

### StaleElementReferenceException

```java
// Ссылка на элемент устарела — DOM обновился
WebElement button = driver.findElement(By.id("submit"));
// ... страница перезагружается или DOM изменяется ...
button.click(); // StaleElementReferenceException

// Решение: повторный поиск элемента
public void safeClick(By locator) {
    int attempts = 0;
    while (attempts < 3) {
        try {
            driver.findElement(locator).click();
            return;
        } catch (StaleElementReferenceException e) {
            attempts++;
            logger.warn("Stale element, попытка {}/3", attempts);
        }
    }
    throw new TestFrameworkException("Не удалось кликнуть после 3 попыток: " + locator);
}
```

### AssertionError

```java
// Не исключение, а Error — но QA видит его постоянно
// Выбрасывается при неуспешных assert'ах

import static org.assertj.core.api.Assertions.assertThat;

// AssertionError с понятным сообщением
assertThat(actualTitle)
        .as("Заголовок страницы после логина")
        .isEqualTo("Dashboard");
// При несовпадении:
// AssertionError: [Заголовок страницы после логина]
// expected: "Dashboard" but was: "Login"
```

---

## Лучшие практики обработки исключений в тестах

### 1. Не глотайте исключения

```java
// ПЛОХО — тест "зелёный", но ничего не проверяет
@Test
public void testLogin() {
    try {
        loginPage.login("user", "pass");
        assertThat(dashboardPage.isLoaded()).isTrue();
    } catch (Exception e) {
        // Молча проглотили — тест пройдёт, даже если упал
    }
}

// ХОРОШО — пусть тест упадёт с понятным сообщением
@Test
public void testLogin() {
    loginPage.login("user", "pass");
    assertThat(dashboardPage.isLoaded())
            .as("Dashboard должен загрузиться после успешного логина")
            .isTrue();
}
```

### 2. Используйте assertThrows для проверки исключений

```java
// Проверка, что метод выбрасывает ожидаемое исключение
@Test
void shouldThrowExceptionForInvalidCredentials() {
    TestDataException exception = assertThrows(
            TestDataException.class,
            () -> loginPage.login("", "password")
    );

    assertThat(exception.getMessage()).contains("username не может быть пустым");
    assertThat(exception.getDataSource()).isEqualTo("LoginPage");
}
```

### 3. Скриншот и логирование при падении

```java
// Слушатель TestNG для автоматического скриншота при падении
public class ScreenshotOnFailureListener implements ITestListener {

    @Override
    public void onTestFailure(ITestResult result) {
        Throwable throwable = result.getThrowable();
        logger.error("Тест {} упал: {}", result.getName(), throwable.getMessage());

        // Сохраняем скриншот
        WebDriver driver = getDriverFromTest(result);
        if (driver != null) {
            File screenshot = ((TakesScreenshot) driver).getScreenshotAs(OutputType.FILE);
            // Сохраняем в папку отчётов
            saveScreenshot(screenshot, result.getName());
        }
    }
}
```

### 4. Оборачивайте низкоуровневые исключения

```java
// ПЛОХО — тестовый код пробрасывает IOException
@Test
public void testDataProcessing() throws IOException {
    String data = Files.readString(Path.of("test_data.json"));
    // ...
}

// ХОРОШО — оборачиваем в исключение фреймворка с контекстом
@Test
public void testDataProcessing() {
    String data = TestDataReader.readJson("test_data.json"); // внутри обёрнуто в TestDataException
    // ...
}
```

### 5. Цепочка исключений (Exception Chaining)

```java
// Сохраняйте исходное исключение как cause
public List<User> loadTestUsers(String filePath) {
    try {
        String json = Files.readString(Path.of(filePath));
        return objectMapper.readValue(json, new TypeReference<>() {});
    } catch (IOException e) {
        // Оригинальное исключение сохранено как cause
        throw new TestDataException("users_file",
                "Не удалось загрузить пользователей из " + filePath, e);
    }
}
```

---

## Связь с тестированием

- **Selenium/WebDriver** — `NoSuchElementException`, `TimeoutException`, `StaleElementReferenceException`
  возникают при работе с элементами страницы
- **REST API тесты** — `IOException`, `HttpTimeoutException` при сетевых запросах
- **Тестовые данные** — `FileNotFoundException`, `JsonParseException` при загрузке данных
- **Assertions** — `AssertionError` при несовпадении ожидаемого и фактического результата
- **Фреймворки** — TestNG и JUnit имеют механизмы ожидания исключений (`@Test(expected=...)`, `assertThrows`)
- **Custom Exceptions** — собственные исключения делают отчёты о падениях тестов информативнее

---

## Типичные ошибки

1. **Слишком широкий catch** — `catch (Exception e)` ловит всё, скрывая реальную проблему
2. **Пустой catch-блок** — проглатывание исключения без логирования и реакции
3. **Потеря cause** — `throw new RuntimeException(e.getMessage())` вместо `new RuntimeException(msg, e)`
4. **Catch и return null** — возвращение null вместо пробрасывания; приводит к NPE позже
5. **Использование исключений для управления потоком** — дорогая операция, лучше `if/else`
6. **Неправильный порядок catch** — общий тип перед частным: `catch (Exception e)` до `catch (IOException e)` — код не скомпилируется
7. **finally без try-with-resources** — ручное закрытие ресурсов часто содержит ошибки

---

## Вопросы на интервью

### Уровень Junior

- Что такое исключения в Java? Зачем они нужны?
- Какова иерархия исключений?
- В чём разница между `checked` и `unchecked` исключениями?
- Что делают `try`, `catch`, `finally`?
- Что такое `try-with-resources`?
- Что произойдёт, если в `finally` бросить исключение?

### Уровень Middle

- Чем `Error` отличается от `Exception`?
- Зачем создавать собственные исключения? Приведите пример из тестового фреймворка.
- Как работает multi-catch? Каковы ограничения?
- Что такое цепочка исключений (exception chaining)?
- Как правильно обрабатывать `NoSuchElementException` в Selenium-тестах?
- В чём разница между `throw` и `throws`?
- Может ли `finally` не выполниться?

### Уровень Senior

- Как исключения влияют на производительность? Почему не стоит использовать их для управления потоком?
- Как устроен механизм раскрутки стека (stack unwinding)?
- Как обрабатывать исключения в `Stream API`?
- Как реализовать retry-механизм с exponential backoff для flaky-тестов?
- Как спроектировать иерархию исключений для тестового фреймворка?
- Как `suppressed exceptions` работают в `try-with-resources`?

---

## Практические задания

1. **Custom Exception:** Создайте иерархию исключений для тестового фреймворка: `TestFrameworkException` (базовое), `ElementNotFoundException` (элемент не найден), `ApiResponseException` (неожиданный HTTP-статус, с полями `statusCode` и `responseBody`).

2. **Retry-механизм:** Напишите метод `retryOnException(Runnable action, int maxRetries, Class<? extends Exception>... retryOn)`, который повторяет действие указанное количество раз при определённых исключениях.

3. **Safe wrapper:** Создайте утилитный метод `safeFind(WebDriver driver, By locator)`, который возвращает `Optional<WebElement>` вместо выбрасывания `NoSuchElementException`.

4. **try-with-resources:** Напишите класс `TestEnvironment`, реализующий `AutoCloseable`, который при создании инициализирует WebDriver и подключение к БД, а при закрытии корректно освобождает оба ресурса.

5. **Exception Logging:** Реализуйте TestNG Listener, который при падении теста сохраняет: скриншот, URL текущей страницы, stack trace и время падения.

---

## Дополнительные ресурсы

- [Oracle — Exceptions Tutorial](https://docs.oracle.com/javase/tutorial/essential/exceptions/)
- [Baeldung — Exception Handling in Java](https://www.baeldung.com/java-exceptions)
- [Baeldung — Custom Exceptions](https://www.baeldung.com/java-new-custom-exception)
- [Effective Java — Joshua Bloch (глава 10: Exceptions)](https://www.oreilly.com/library/view/effective-java/9780134686097/)
- [Selenium WebDriver Exceptions](https://www.selenium.dev/documentation/webdriver/troubleshooting/errors/)
