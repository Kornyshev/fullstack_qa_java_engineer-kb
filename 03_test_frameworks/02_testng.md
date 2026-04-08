# TestNG

## Обзор

TestNG (Test Next Generation) — мощный тестовый фреймворк для Java, вдохновлённый JUnit и NUnit, но предоставляющий расширенные возможности «из коробки»: зависимости между тестами, группы, параллельное выполнение через XML-конфигурацию, data providers и гибкие listeners. TestNG традиционно популярен в проектах автоматизации тестирования, особенно в связке с Selenium WebDriver.

**Зависимость (Maven):**

```xml
<dependency>
    <groupId>org.testng</groupId>
    <artifactId>testng</artifactId>
    <version>7.9.0</version>
    <scope>test</scope>
</dependency>
```

---

## Сравнение TestNG и JUnit 5

| Возможность | JUnit 5 | TestNG |
|---|---|---|
| Аннотация теста | `@Test` | `@Test` |
| До каждого теста | `@BeforeEach` | `@BeforeMethod` |
| После каждого теста | `@AfterEach` | `@AfterMethod` |
| До всех тестов в классе | `@BeforeAll` | `@BeforeClass` |
| После всех тестов в классе | `@AfterAll` | `@AfterClass` |
| До тестового набора | — | `@BeforeSuite` |
| После тестового набора | — | `@AfterSuite` |
| До группы | — | `@BeforeGroups` |
| Параметризация | `@ParameterizedTest` + sources | `@DataProvider` |
| Группы | `@Tag` | `groups` атрибут в `@Test` |
| Зависимости тестов | Нет (через Extensions) | `dependsOnMethods`, `dependsOnGroups` |
| Параллельный запуск | `junit-platform.properties` | `testng.xml` (нативно) |
| Retry упавших тестов | Через Extension | `IRetryAnalyzer` (нативно) |
| Soft Assertions | Через AssertJ | `SoftAssert` (встроен) |
| XML конфигурация | Нет | `testng.xml` |
| Фабрика тестов | Нет | `@Factory` |
| Listener-система | Extensions API | `ITestListener` и другие |
| Отчёты | Через плагины | Встроенные HTML-отчёты |

---

## Основные аннотации

### @Test

Базовая аннотация, поддерживающая множество атрибутов.

```java
import org.testng.annotations.Test;
import static org.testng.Assert.*;

public class CalculatorTest {

    @Test(description = "Проверка сложения двух положительных чисел")
    public void shouldAddTwoNumbers() {
        Calculator calc = new Calculator();
        assertEquals(calc.add(2, 3), 5, "Результат сложения неверен");
    }

    @Test(enabled = false, description = "Временно отключён до BUG-555")
    public void disabledTest() {
        // Этот тест не будет выполнен
    }

    @Test(expectedExceptions = ArithmeticException.class,
          description = "Деление на ноль должно вызвать исключение")
    public void shouldThrowOnDivideByZero() {
        new Calculator().divide(10, 0);
    }

    @Test(timeOut = 2000, description = "Должен завершиться за 2 секунды")
    public void shouldCompleteQuickly() {
        // Тест упадёт, если выполнение займёт более 2000 мс
        apiClient.fetchData();
    }
}
```

### Аннотации жизненного цикла

```java
public class LifecycleTest {

    @BeforeSuite
    public void beforeSuite() {
        // Один раз перед всем набором тестов (suite)
        System.out.println("Инициализация тестового окружения");
    }

    @BeforeClass
    public void beforeClass() {
        // Один раз перед текущим классом
        System.out.println("Настройка класса");
    }

    @BeforeMethod
    public void beforeMethod() {
        // Перед каждым тестовым методом
        System.out.println("Подготовка к тесту");
    }

    @Test
    public void testOne() {
        System.out.println("Тест 1");
    }

    @Test
    public void testTwo() {
        System.out.println("Тест 2");
    }

    @AfterMethod
    public void afterMethod() {
        // После каждого тестового метода
        System.out.println("Очистка после теста");
    }

    @AfterClass
    public void afterClass() {
        System.out.println("Очистка класса");
    }

    @AfterSuite
    public void afterSuite() {
        System.out.println("Финализация тестового окружения");
    }
}
```

**Полный порядок выполнения:**

```
@BeforeSuite → @BeforeTest → @BeforeClass →
    @BeforeMethod → @Test → @AfterMethod →
    @BeforeMethod → @Test → @AfterMethod →
@AfterClass → @AfterTest → @AfterSuite
```

---

## @DataProvider

Основной механизм параметризации в TestNG. Возвращает двумерный массив `Object[][]` или `Iterator<Object[]>`.

### Базовый пример

```java
public class LoginTest {

    @DataProvider(name = "validCredentials")
    public Object[][] provideValidCredentials() {
        return new Object[][] {
            {"admin", "admin123", "ADMIN"},
            {"user", "user123", "USER"},
            {"guest", "guest123", "GUEST"}
        };
    }

    @Test(dataProvider = "validCredentials",
          description = "Проверка входа с валидными учётными данными")
    public void shouldLoginWithValidCredentials(String username,
                                                 String password,
                                                 String expectedRole) {
        LoginResult result = authService.login(username, password);
        assertTrue(result.isSuccess(), "Вход должен быть успешным");
        assertEquals(result.getRole(), expectedRole, "Роль не совпадает");
    }
}
```

### DataProvider из отдельного класса

```java
// Отдельный класс с данными
public class TestData {

    @DataProvider(name = "emails")
    public static Object[][] provideEmails() {
        return new Object[][] {
            {"user@mail.com", true},
            {"invalid-email", false},
            {"", false},
            {"a@b.c", true},
            {"user@.com", false}
        };
    }
}

// Тестовый класс
public class EmailValidationTest {

    @Test(dataProvider = "emails", dataProviderClass = TestData.class,
          description = "Валидация email-адресов")
    public void shouldValidateEmail(String email, boolean expected) {
        assertEquals(EmailValidator.isValid(email), expected,
                "Неверный результат для: " + email);
    }
}
```

### DataProvider с Iterator (для больших объёмов данных)

```java
@DataProvider(name = "largeDataSet")
public Iterator<Object[]> provideLargeDataSet() {
    // Данные подгружаются лениво — экономия памяти
    List<Object[]> data = new ArrayList<>();
    try (BufferedReader reader = new BufferedReader(
            new FileReader("testdata/users.csv"))) {
        String line;
        while ((line = reader.readLine()) != null) {
            String[] parts = line.split(",");
            data.add(new Object[]{parts[0], parts[1], Integer.parseInt(parts[2])});
        }
    } catch (IOException e) {
        throw new RuntimeException("Ошибка чтения тестовых данных", e);
    }
    return data.iterator();
}
```

---

## testng.xml — конфигурация

XML-файл, управляющий составом и порядком запуска тестов.

### Базовая конфигурация

```xml
<!DOCTYPE suite SYSTEM "https://testng.org/testng-1.0.dtd">
<suite name="Regression Suite" verbose="1">

    <test name="API Tests">
        <classes>
            <class name="com.example.tests.api.UserApiTest"/>
            <class name="com.example.tests.api.OrderApiTest"/>
        </classes>
    </test>

    <test name="UI Tests">
        <packages>
            <package name="com.example.tests.ui.*"/>
        </packages>
    </test>

</suite>
```

### Конфигурация с группами

```xml
<suite name="Smoke Suite">
    <test name="Smoke Tests">
        <groups>
            <run>
                <include name="smoke"/>
                <exclude name="broken"/>
            </run>
        </groups>
        <packages>
            <package name="com.example.tests.*"/>
        </packages>
    </test>
</suite>
```

### Параметры из XML

```xml
<suite name="Cross-Browser Suite">
    <test name="Chrome Tests">
        <parameter name="browser" value="chrome"/>
        <parameter name="baseUrl" value="https://staging.example.com"/>
        <classes>
            <class name="com.example.tests.ui.LoginUiTest"/>
        </classes>
    </test>

    <test name="Firefox Tests">
        <parameter name="browser" value="firefox"/>
        <parameter name="baseUrl" value="https://staging.example.com"/>
        <classes>
            <class name="com.example.tests.ui.LoginUiTest"/>
        </classes>
    </test>
</suite>
```

```java
public class LoginUiTest {

    @Parameters({"browser", "baseUrl"})
    @BeforeMethod
    public void setUp(String browser, String baseUrl) {
        // Инициализация WebDriver на основе параметра из testng.xml
        driver = WebDriverFactory.create(browser);
        driver.get(baseUrl);
    }
}
```

---

## Группы

Позволяют категоризировать тесты и управлять запуском через `testng.xml`.

```java
public class UserServiceTest {

    @Test(groups = {"smoke", "api"})
    public void shouldCreateUser() {
        // Входит и в smoke, и в api группу
    }

    @Test(groups = {"regression", "api"})
    public void shouldUpdateUserProfile() {
        // Только regression и api
    }

    @Test(groups = {"regression", "api"},
          dependsOnGroups = {"smoke"})
    public void shouldDeleteUser() {
        // Запустится только после успешного прохождения всех smoke-тестов
    }
}
```

---

## dependsOnMethods

Задаёт зависимости между тестами — зависимый тест будет пропущен (SKIP), если базовый упал.

```java
public class OrderFlowTest {

    private Long orderId;

    @Test(description = "Шаг 1: Создание заказа")
    public void createOrder() {
        orderId = orderService.create(new OrderRequest("item-1", 2));
        assertNotNull(orderId, "Заказ должен быть создан");
    }

    @Test(dependsOnMethods = "createOrder",
          description = "Шаг 2: Оплата заказа")
    public void payForOrder() {
        PaymentResult result = paymentService.pay(orderId, "CARD");
        assertTrue(result.isSuccess());
    }

    @Test(dependsOnMethods = "payForOrder",
          description = "Шаг 3: Отправка заказа")
    public void shipOrder() {
        ShippingResult result = shippingService.ship(orderId);
        assertEquals(result.getStatus(), "SHIPPED");
    }
}
```

> **Предупреждение:** зависимости между тестами нарушают принцип изоляции. Используйте только для E2E-сценариев, где порядок шагов принципиален.

---

## Listeners

### ITestListener

Перехватывает события жизненного цикла тестов — полезен для логирования, скриншотов, отчётов.

```java
public class TestListener implements ITestListener {

    @Override
    public void onTestStart(ITestResult result) {
        System.out.println("Старт теста: " + result.getName());
    }

    @Override
    public void onTestSuccess(ITestResult result) {
        System.out.println("PASSED: " + result.getName());
    }

    @Override
    public void onTestFailure(ITestResult result) {
        System.out.println("FAILED: " + result.getName());
        System.out.println("Причина: " + result.getThrowable().getMessage());
        // Здесь можно сделать скриншот при UI-тестировании
        captureScreenshot(result.getName());
    }

    @Override
    public void onTestSkipped(ITestResult result) {
        System.out.println("SKIPPED: " + result.getName());
    }

    private void captureScreenshot(String testName) {
        // Логика сохранения скриншота
    }
}
```

**Подключение listener:**

```java
// Через аннотацию
@Listeners(TestListener.class)
public class MyTest { ... }
```

```xml
<!-- Через testng.xml — применяется ко всем тестам -->
<suite name="Suite">
    <listeners>
        <listener class-name="com.example.listeners.TestListener"/>
    </listeners>
</suite>
```

### IRetryAnalyzer — перезапуск упавших тестов

```java
public class RetryAnalyzer implements IRetryAnalyzer {

    private int retryCount = 0;
    private static final int MAX_RETRIES = 3;

    @Override
    public boolean retry(ITestResult result) {
        if (retryCount < MAX_RETRIES) {
            retryCount++;
            System.out.printf("Повторный запуск теста '%s': попытка %d из %d%n",
                    result.getName(), retryCount, MAX_RETRIES);
            return true; // Повторить тест
        }
        return false; // Больше не повторять
    }
}

// Использование
@Test(retryAnalyzer = RetryAnalyzer.class,
      description = "Тест с нестабильным внешним сервисом")
public void flakyExternalServiceTest() {
    // Будет перезапущен до 3 раз при падении
    String response = externalApi.call("/health");
    assertEquals(response, "OK");
}
```

---

## Soft Assertions

Встроенная поддержка soft assertions — все проверки выполняются, ошибки собираются и выбрасываются в конце.

```java
@Test(description = "Проверка профиля пользователя со сбором всех ошибок")
public void shouldReturnCorrectUserProfile() {
    User user = userService.getById(1L);
    SoftAssert softAssert = new SoftAssert();

    softAssert.assertEquals(user.getFirstName(), "Иван", "Имя не совпадает");
    softAssert.assertEquals(user.getLastName(), "Петров", "Фамилия не совпадает");
    softAssert.assertEquals(user.getEmail(), "ivan@mail.com", "Email не совпадает");
    softAssert.assertTrue(user.isActive(), "Пользователь должен быть активен");
    softAssert.assertNotNull(user.getCreatedAt(), "Дата создания обязательна");

    // ОБЯЗАТЕЛЬНО вызвать assertAll() — иначе ошибки будут проигнорированы!
    softAssert.assertAll();
}
```

> **Критическая ошибка:** забыть вызвать `softAssert.assertAll()` — тест пройдёт «зелёным», даже если все проверки провалились.

---

## Параллельное выполнение

TestNG предоставляет гибкую настройку параллельности прямо в `testng.xml`.

```xml
<!-- Параллельное выполнение на уровне методов -->
<suite name="Parallel Suite" parallel="methods" thread-count="4">
    <test name="All Tests">
        <classes>
            <class name="com.example.tests.UserTest"/>
            <class name="com.example.tests.OrderTest"/>
        </classes>
    </test>
</suite>

<!-- Параллельное выполнение на уровне классов -->
<suite name="Parallel Suite" parallel="classes" thread-count="3">
    <test name="All Tests">
        <packages>
            <package name="com.example.tests.*"/>
        </packages>
    </test>
</suite>

<!-- Параллельное выполнение на уровне тестов (блоков <test>) -->
<suite name="Cross-Browser" parallel="tests" thread-count="2">
    <test name="Chrome">
        <parameter name="browser" value="chrome"/>
        <classes>
            <class name="com.example.tests.LoginTest"/>
        </classes>
    </test>
    <test name="Firefox">
        <parameter name="browser" value="firefox"/>
        <classes>
            <class name="com.example.tests.LoginTest"/>
        </classes>
    </test>
</suite>
```

**Уровни параллельности:**
- `parallel="methods"` — методы одного класса в разных потоках
- `parallel="classes"` — каждый класс в своём потоке
- `parallel="tests"` — каждый блок `<test>` в своём потоке
- `parallel="instances"` — экземпляры классов в разных потоках

---

## @Factory

Создаёт несколько экземпляров тестового класса с разными данными.

```java
public class LoginFactoryTest {

    private final String username;
    private final String password;
    private final boolean expectedResult;

    public LoginFactoryTest(String username, String password, boolean expectedResult) {
        this.username = username;
        this.password = password;
        this.expectedResult = expectedResult;
    }

    @Test
    public void shouldLoginWithGivenCredentials() {
        boolean result = authService.login(username, password);
        assertEquals(result, expectedResult,
                String.format("Вход для %s: ожидалось %s", username, expectedResult));
    }

    @Factory
    public static Object[] createInstances() {
        return new Object[] {
            new LoginFactoryTest("admin", "admin123", true),
            new LoginFactoryTest("user", "wrong", false),
            new LoginFactoryTest("", "", false),
            new LoginFactoryTest("blocked", "pass123", false)
        };
    }
}
```

---

## Когда выбрать TestNG, а когда JUnit 5

| Критерий | Рекомендация |
|---|---|
| Новый проект с Spring Boot | **JUnit 5** — нативная интеграция, поддержка по умолчанию |
| Legacy-проект с Selenium | **TestNG** — часто уже используется, много существующих `testng.xml` |
| Нужны зависимости между тестами | **TestNG** — `dependsOnMethods` из коробки |
| Кросс-браузерное тестирование | **TestNG** — удобная параллелизация через XML |
| Data-driven тестирование | **Оба** — `@DataProvider` (TestNG) vs `@ParameterizedTest` (JUnit 5) |
| Retry flaky-тестов | **TestNG** — `IRetryAnalyzer` нативно, в JUnit 5 — через extension |
| Модульное тестирование | **JUnit 5** — стандарт в мире Java |
| CI/CD интеграция | **Оба** — одинаково хорошо поддерживаются |

---

## Связь с тестированием

TestNG активно используется QA-инженерами в следующих контекстах:

- **Selenium-проекты**: управление набором UI-тестов, кросс-браузерное тестирование, скриншоты через listeners
- **API-тестирование**: data-driven подход через `@DataProvider`, группировка тестов
- **E2E-сценарии**: `dependsOnMethods` для последовательных шагов
- **CI/CD**: выборочный запуск групп тестов, параллельное выполнение для ускорения пайплайна
- **Отчётность**: встроенные HTML-отчёты, интеграция с Allure через listeners

---

## Типичные ошибки

1. **Забыть `softAssert.assertAll()`** — тест не зафиксирует ни одной ошибки
2. **Злоупотребление `dependsOnMethods`** — нарушает изоляцию тестов, создаёт хрупкую цепочку
3. **Мутабельное состояние при параллельном запуске** — общие переменные экземпляра класса приводят к гонкам данных
4. **Не указать `dataProviderClass`** — если `@DataProvider` в другом классе, TestNG его не найдёт
5. **Путаница `assertEquals` аргументов** — в TestNG порядок: `actual, expected` (обратный по сравнению с JUnit!)
6. **Listener подключён дважды** — через аннотацию и XML одновременно, события дублируются
7. **`@BeforeMethod` без `alwaysRun = true`** — при зависимостях может быть пропущен

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Какие аннотации жизненного цикла есть в TestNG? В каком порядке выполняются?
2. Чем `@DataProvider` отличается от `@Parameters`?
3. Как отключить тест в TestNG?
4. Что такое `testng.xml` и зачем он нужен?
5. Как задать группу теста и запустить только определённую группу?

### 🟡 Средний уровень
6. Чем `SoftAssert` отличается от обычных assert? Какую ошибку легко допустить?
7. Как настроить параллельный запуск тестов? Какие уровни параллельности есть?
8. Для чего используется `IRetryAnalyzer`? Приведите пример.
9. Сравните TestNG `@DataProvider` и JUnit 5 `@ParameterizedTest`.
10. Когда использовать `dependsOnMethods`, а когда — независимые тесты?

### 🔴 Продвинутый уровень
11. Как реализовать кросс-браузерное тестирование через `testng.xml`?
12. Чем `@Factory` отличается от `@DataProvider`? Когда использовать каждый?
13. Как написать кастомный `ITestListener` для интеграции с Allure?
14. Как обеспечить потокобезопасность при `parallel="methods"`?
15. Сравните архитектуру расширений TestNG (Listeners) и JUnit 5 (Extensions).

---

## Практические задания

### Задание 1: DataProvider
Создайте тест с `@DataProvider`, проверяющий валидацию пароля. Пароль должен быть не менее 8 символов, содержать цифру и спецсимвол. Покройте 10+ кейсов.

### Задание 2: testng.xml
Напишите `testng.xml` для запуска: smoke-тестов в Chrome, regression-тестов в Firefox, параллельно в 2 потока.

### Задание 3: Retry Analyzer
Реализуйте `IRetryAnalyzer` с настраиваемым количеством попыток из системных свойств. Подключите глобально через listener.

### Задание 4: SoftAssert утилита
Создайте утилитный класс `AssertionHelper`, оборачивающий `SoftAssert` с автоматическим вызовом `assertAll()` в `@AfterMethod`.

### Задание 5: E2E-сценарий
Напишите E2E-тест: регистрация -> логин -> создание заказа -> отмена заказа. Используйте `dependsOnMethods` и `@DataProvider` для разных типов пользователей.

---

## Дополнительные ресурсы

- [TestNG Documentation](https://testng.org/doc/documentation-main.html)
- [TestNG GitHub](https://github.com/testng-team/testng)
- [Baeldung — TestNG](https://www.baeldung.com/testng)
- [TestNG vs JUnit 5](https://www.baeldung.com/junit-vs-testng)
- [TestNG Maven Surefire Plugin](https://maven.apache.org/surefire/maven-surefire-plugin/examples/testng.html)
