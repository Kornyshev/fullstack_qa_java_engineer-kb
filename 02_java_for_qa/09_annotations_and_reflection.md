# Аннотации и Reflection

## Обзор

Аннотации (annotations) — это механизм метаданных в Java, позволяющий добавлять дополнительную информацию к классам, методам, полям и параметрам. Reflection — это возможность программы анализировать и изменять свою структуру во время выполнения. Для QA-инженера понимание этих механизмов критически важно, потому что на них построены **все** тестовые фреймворки: JUnit распознаёт `@Test` через Reflection, Allure собирает отчёты с помощью `@Step` и `@Epic`, Spring внедряет зависимости через `@Autowired`. Без понимания аннотаций невозможно эффективно строить тестовый фреймворк, создавать кастомные аннотации для маркировки тестов и понимать, почему что-то «магически» работает в тестовом коде.

---

## Встроенные аннотации Java

Java предоставляет набор стандартных аннотаций, которые QA-инженер встречает ежедневно.

### @Override

Указывает компилятору, что метод переопределяет метод родительского класса. Если родительский метод не существует — будет ошибка компиляции.

```java
public class BasePage {
    public void waitForPageLoaded() {
        // Базовая реализация ожидания загрузки страницы
    }
}

public class LoginPage extends BasePage {
    @Override
    public void waitForPageLoaded() {
        // Специфичное ожидание для страницы логина
        waitForElement(loginButton);
    }
}
```

**Зачем QA:** при создании Page Object иерархии `@Override` помогает убедиться, что вы правильно переопределяете метод базового класса, а не создаёте новый по ошибке.

### @Deprecated

Отмечает элемент как устаревший. Компилятор выдаст предупреждение при его использовании.

```java
public class TestUtils {

    @Deprecated(since = "2.0", forRemoval = true)
    public static void sleepSeconds(int seconds) {
        // Устаревший способ ожидания — используйте явные Waits
        Thread.sleep(seconds * 1000);
    }

    // Рекомендуемая замена
    public static void waitForCondition(ExpectedCondition<?> condition) {
        new WebDriverWait(driver, Duration.ofSeconds(10)).until(condition);
    }
}
```

**Зачем QA:** при рефакторинге тестового фреймворка `@Deprecated` помогает плавно выводить старые утилитные методы из использования, не ломая существующие тесты.

### @SuppressWarnings

Подавляет предупреждения компилятора. Используется осознанно, когда предупреждение понятно и неизбежно.

```java
@SuppressWarnings("unchecked")
public <T> List<T> getTestData(String jsonPath) {
    // Подавляем предупреждение о непроверяемом приведении типов
    return (List<T>) objectMapper.readValue(
        new File(jsonPath), List.class
    );
}
```

| Параметр | Что подавляет |
|----------|--------------|
| `"unchecked"` | Непроверяемые приведения типов (generics) |
| `"deprecation"` | Использование устаревших API |
| `"rawtypes"` | Использование raw types без generics |
| `"unused"` | Неиспользуемые переменные и методы |

### @FunctionalInterface

Указывает, что интерфейс содержит ровно один абстрактный метод и может использоваться как lambda-выражение.

```java
@FunctionalInterface
public interface RetryAction {
    // Действие, которое может быть выполнено с повторной попыткой
    void execute() throws Exception;
}

// Использование в тестовом фреймворке
public void withRetry(int maxAttempts, RetryAction action) {
    for (int i = 0; i < maxAttempts; i++) {
        try {
            action.execute();
            return;
        } catch (Exception e) {
            if (i == maxAttempts - 1) throw new RuntimeException(e);
        }
    }
}

// Вызов в тесте
withRetry(3, () -> loginPage.clickSubmitButton());
```

---

## Создание кастомных аннотаций

QA-инженеры часто создают собственные аннотации для маркировки тестов, управления тестовыми данными и конфигурацией.

### Синтаксис создания

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

// Аннотация для маркировки тестов по приоритету
@Target(ElementType.METHOD)          // Можно применять только к методам
@Retention(RetentionPolicy.RUNTIME)  // Доступна во время выполнения
public @interface TestPriority {
    Priority value() default Priority.MEDIUM;

    enum Priority {
        CRITICAL, HIGH, MEDIUM, LOW
    }
}
```

### Пример: аннотация для Jira-тикетов

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface JiraIssue {
    String value();              // Номер тикета: "QA-1234"
    String summary() default ""; // Краткое описание
}

// Использование
@Test
@JiraIssue(value = "QA-1234", summary = "Логин с невалидным паролем")
void shouldShowErrorForInvalidPassword() {
    // Тест привязан к конкретному тикету в Jira
    loginPage.enterCredentials("user@test.com", "wrong");
    loginPage.clickLogin();
    assertThat(loginPage.getErrorMessage())
        .isEqualTo("Неверный пароль");
}
```

### Пример: аннотация для управления тестовыми данными

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface TestData {
    String file();                     // Путь к файлу с тестовыми данными
    String dataset() default "default"; // Набор данных внутри файла
}
```

---

## Мета-аннотации

Мета-аннотации — это аннотации, которые применяются к другим аннотациям и определяют их поведение.

| Мета-аннотация | Назначение | Значения |
|----------------|-----------|----------|
| `@Target` | Где можно использовать аннотацию | `TYPE`, `METHOD`, `FIELD`, `PARAMETER`, `CONSTRUCTOR` |
| `@Retention` | Когда аннотация доступна | `SOURCE`, `CLASS`, `RUNTIME` |
| `@Documented` | Включать ли в Javadoc | Маркерная (без параметров) |
| `@Inherited` | Наследуется ли подклассами | Маркерная (без параметров) |
| `@Repeatable` | Можно ли применять несколько раз | Указывает контейнерную аннотацию |

### @Retention — уровни сохранения

```
SOURCE  ──→ Только в исходном коде, отбрасывается компилятором
              Пример: @Override, @SuppressWarnings

CLASS   ──→ Сохраняется в .class файле, но недоступна через Reflection
              Пример: редко используется в тестах

RUNTIME ──→ Доступна во время выполнения через Reflection
              Пример: @Test, @Step, @Autowired — всё, что нужно QA
```

**Для тестирования важно:** почти все аннотации в тестовых фреймворках имеют `RetentionPolicy.RUNTIME`, потому что фреймворк должен «видеть» их во время выполнения.

### @Target — допустимые цели

```java
// Аннотация только для методов (как @Test)
@Target(ElementType.METHOD)

// Аннотация для классов и методов (как @DisplayName)
@Target({ElementType.TYPE, ElementType.METHOD})

// Аннотация для параметров (как @ParameterizedTest источники)
@Target(ElementType.PARAMETER)
```

---

## Reflection API

Reflection — механизм, позволяющий получать информацию о классах, методах и полях во время выполнения программы. Это «магия», на которой работают все тестовые фреймворки.

### Объект Class

Каждый класс в Java представлен объектом `Class<?>`, через который можно получить всю информацию.

```java
// Три способа получить объект Class
Class<?> clazz1 = LoginPage.class;                    // Через литерал класса
Class<?> clazz2 = loginPage.getClass();                // Через экземпляр объекта
Class<?> clazz3 = Class.forName("com.test.LoginPage"); // По имени (строкой)
```

### Получение методов

```java
import java.lang.reflect.Method;

Class<?> testClass = UserApiTest.class;

// Все публичные методы (включая унаследованные)
Method[] publicMethods = testClass.getMethods();

// Все объявленные методы (включая private, но без унаследованных)
Method[] declaredMethods = testClass.getDeclaredMethods();

// Поиск методов с аннотацией @Test
for (Method method : declaredMethods) {
    if (method.isAnnotationPresent(Test.class)) {
        System.out.println("Тестовый метод: " + method.getName());
    }
}
```

### Получение полей

```java
import java.lang.reflect.Field;

Class<?> pageClass = LoginPage.class;

// Получаем все поля, включая приватные
Field[] fields = pageClass.getDeclaredFields();

for (Field field : fields) {
    field.setAccessible(true); // Открываем доступ к приватным полям
    System.out.println(field.getName() + " : " + field.getType().getSimpleName());
}
```

### Вызов методов через Reflection

```java
// Динамический вызов метода — так работают тестовые фреймворки
Method testMethod = testClass.getDeclaredMethod("shouldCreateUser");
testMethod.setAccessible(true);

// Создаём экземпляр тестового класса и вызываем метод
Object testInstance = testClass.getDeclaredConstructor().newInstance();
testMethod.invoke(testInstance);
```

### Чтение значений аннотаций

```java
// Получаем значение кастомной аннотации
Method method = testClass.getDeclaredMethod("shouldShowErrorForInvalidPassword");

if (method.isAnnotationPresent(JiraIssue.class)) {
    JiraIssue jira = method.getAnnotation(JiraIssue.class);
    System.out.println("Jira тикет: " + jira.value());
    System.out.println("Описание: " + jira.summary());
}
```

---

## Аннотации в тестовых фреймворках

### JUnit 5

| Аннотация | Назначение | Пример |
|-----------|-----------|--------|
| `@Test` | Отмечает тестовый метод | `@Test void shouldLogin()` |
| `@BeforeEach` | Выполняется перед каждым тестом | Инициализация WebDriver |
| `@AfterEach` | Выполняется после каждого теста | Закрытие браузера |
| `@BeforeAll` | Один раз перед всеми тестами класса | Запуск тестового окружения |
| `@DisplayName` | Человекочитаемое имя теста | `@DisplayName("Авторизация с валидными данными")` |
| `@Tag` | Категоризация тестов | `@Tag("smoke")`, `@Tag("regression")` |
| `@Disabled` | Отключение теста | `@Disabled("Блокер BUG-456")` |
| `@ParameterizedTest` | Параметризованный тест | С `@CsvSource`, `@MethodSource` |
| `@Nested` | Вложенные тестовые классы | Группировка тестов по сценариям |

```java
@DisplayName("Тесты авторизации")
class LoginTest extends BaseTest {

    @BeforeEach
    void openLoginPage() {
        // Открываем страницу перед каждым тестом
        loginPage.open();
    }

    @Test
    @Tag("smoke")
    @DisplayName("Успешная авторизация с валидными данными")
    void shouldLoginWithValidCredentials() {
        loginPage.enterUsername("admin");
        loginPage.enterPassword("password123");
        loginPage.clickLogin();
        assertThat(dashboardPage.isDisplayed()).isTrue();
    }

    @ParameterizedTest
    @CsvSource({
        "'', password123, 'Введите логин'",
        "admin, '', 'Введите пароль'"
    })
    @DisplayName("Валидация обязательных полей")
    void shouldShowValidationErrors(String user, String pass, String error) {
        loginPage.enterUsername(user);
        loginPage.enterPassword(pass);
        loginPage.clickLogin();
        assertThat(loginPage.getErrorMessage()).isEqualTo(error);
    }
}
```

### Allure

| Аннотация | Назначение |
|-----------|-----------|
| `@Step` | Шаг в отчёте — самый часто используемый |
| `@Epic` | Эпик (большая функциональная область) |
| `@Feature` | Фича внутри эпика |
| `@Story` | Пользовательская история внутри фичи |
| `@Severity` | Критичность теста |
| `@Description` | Описание теста в отчёте |
| `@Link` | Ссылка на тикет или документацию |

```java
@Epic("Авторизация")
@Feature("Вход по логину и паролю")
public class LoginTest {

    @Test
    @Story("Успешный вход")
    @Severity(SeverityLevel.BLOCKER)
    @Description("Проверяем, что пользователь может войти с валидными данными")
    void shouldLoginSuccessfully() {
        openLoginPage();
        enterCredentials("admin", "password123");
        clickLoginButton();
        verifyDashboardDisplayed();
    }

    @Step("Открываем страницу авторизации")
    void openLoginPage() {
        Selenide.open("/login");
    }

    @Step("Вводим логин '{username}' и пароль")
    void enterCredentials(String username, String password) {
        $("[name=username]").setValue(username);
        $("[name=password]").setValue(password);
    }

    @Step("Нажимаем кнопку 'Войти'")
    void clickLoginButton() {
        $("[type=submit]").click();
    }

    @Step("Проверяем отображение Dashboard")
    void verifyDashboardDisplayed() {
        $(".dashboard").shouldBe(visible);
    }
}
```

### Spring (для тестов)

| Аннотация | Где QA встречает |
|-----------|-----------------|
| `@Autowired` | Внедрение зависимостей в тестовые конфигурации |
| `@Value` | Чтение значений из properties-файлов |
| `@Component` / `@Service` | Понимание структуры тестируемого приложения |
| `@SpringBootTest` | Запуск Spring-контекста для интеграционных тестов |
| `@MockBean` | Замена бина моком в тестах |

```java
@SpringBootTest
class UserServiceIntegrationTest {

    @Autowired
    private UserService userService; // Spring внедряет зависимость через Reflection

    @MockBean
    private EmailService emailService; // Мок вместо реального сервиса

    @Value("${test.admin.email}")
    private String adminEmail; // Значение из application-test.properties

    @Test
    void shouldCreateUserAndSendEmail() {
        userService.createUser("Иван", adminEmail);
        verify(emailService).sendWelcomeEmail(adminEmail);
    }
}
```

### REST Assured

```java
// REST Assured использует аннотации Jackson для сериализации
public class UserRequest {
    @JsonProperty("user_name")  // Jackson аннотация — сериализация через Reflection
    private String name;

    @JsonProperty("user_email")
    private String email;
}
```

---

## Как Reflection работает «под капотом» фреймворков

Вот упрощённая схема того, как JUnit 5 находит и запускает ваши тесты:

```
1. JUnit сканирует classpath на наличие тестовых классов
2. Для каждого класса через Reflection получает все методы
3. Фильтрует методы с аннотацией @Test
4. Проверяет @BeforeEach, @AfterEach, @BeforeAll, @AfterAll
5. Создаёт экземпляр тестового класса через Reflection (конструктор)
6. Вызывает @BeforeEach методы через Method.invoke()
7. Вызывает @Test метод через Method.invoke()
8. Вызывает @AfterEach методы через Method.invoke()
9. Обрабатывает исключения и формирует результат
```

Аналогично работает Allure: он перехватывает вызовы методов с `@Step` и записывает их в отчёт.

---

## Связь с тестированием

1. **Каждый тестовый метод с `@Test`** — это аннотация, которую JUnit находит через Reflection. Без понимания этого механизма невозможно разобраться, почему тест не запускается (например, если метод `private` — Reflection не найдёт его в некоторых конфигурациях).

2. **Allure-отчёты** строятся на аннотациях `@Step`, `@Epic`, `@Feature`. Умение правильно их расставлять — ключевой навык для читаемых отчётов.

3. **Создание кастомных аннотаций** позволяет маркировать тесты: `@Smoke`, `@Regression`, `@JiraIssue("QA-123")`, что упрощает фильтрацию и запуск подмножеств тестов.

4. **Spring-тесты** активно используют `@Autowired`, `@MockBean`, `@Value` — все они работают через Reflection. Без понимания этого Spring-конфигурация кажется «магией».

5. **Selenium PageFactory** (устаревший, но встречается) использует `@FindBy` аннотации и Reflection для инициализации WebElement-полей.

---

## Типичные ошибки

1. **Забывают `@Retention(RUNTIME)`** при создании кастомной аннотации. Без этого аннотация не будет видна во время выполнения, и фреймворк её не обнаружит. Тест просто не запустится или аннотация будет проигнорирована.

2. **Используют `@Override` неправильно** — пишут аннотацию, но меняют сигнатуру метода (другие параметры или тип возвращаемого значения). Получают ошибку компиляции и не понимают причину.

3. **Не понимают, зачем нужен `setAccessible(true)`** при работе с Reflection. Пытаются обратиться к `private`-полю и получают `IllegalAccessException`.

4. **Злоупотребляют Reflection в тестовом коде.** Reflection — мощный инструмент, но в обычных тестах он не нужен. Если вам приходится через Reflection менять `private`-поле тестируемого объекта, скорее всего, архитектура тестов или приложения нуждается в улучшении.

---

## Вопросы на интервью

- 🟢 **Q:** Что такое аннотация в Java? Приведите примеры встроенных аннотаций.
- **A:** Аннотация — это метаданные, добавляемые к элементам кода (классам, методам, полям). Встроенные: `@Override` (переопределение метода), `@Deprecated` (устаревший элемент), `@SuppressWarnings` (подавление предупреждений), `@FunctionalInterface` (маркер функционального интерфейса).

- 🟢 **Q:** Какие аннотации JUnit 5 вы используете чаще всего?
- **A:** `@Test` — тестовый метод, `@BeforeEach` / `@AfterEach` — настройка/очистка перед/после каждого теста, `@DisplayName` — читаемое имя, `@Tag` — категоризация, `@ParameterizedTest` — параметризация, `@Disabled` — отключение теста.

- 🟢 **Q:** Что такое Reflection?
- **A:** Механизм, позволяющий анализировать и модифицировать структуру программы во время выполнения: получать информацию о классах, методах, полях, вызывать методы динамически. На Reflection построены JUnit, Spring, Allure.

- 🟡 **Q:** Как создать кастомную аннотацию? Где это может понадобиться QA?
- **A:** Используется ключевое слово `@interface`, мета-аннотации `@Target` и `@Retention`. QA может создать аннотации для маркировки тестов: `@Smoke`, `@JiraIssue("QA-123")`, `@TestPriority(CRITICAL)` — для фильтрации и отчётности.

- 🟡 **Q:** В чём разница между `RetentionPolicy.SOURCE`, `CLASS` и `RUNTIME`?
- **A:** `SOURCE` — аннотация существует только в исходном коде (например, `@Override`). `CLASS` — сохраняется в `.class`-файле, но недоступна в runtime. `RUNTIME` — доступна через Reflection во время выполнения. Для тестовых фреймворков нужен `RUNTIME`.

- 🟡 **Q:** Как Allure-отчёт узнаёт о шагах теста?
- **A:** Allure использует аспектно-ориентированное программирование (AOP) и Reflection. Методы с `@Step` перехватываются, их имена и параметры записываются в отчёт. `@Epic`, `@Feature`, `@Story` считываются через Reflection при формировании структуры отчёта.

- 🟡 **Q:** Что делает `@Autowired` в Spring и как это связано с Reflection?
- **A:** `@Autowired` указывает Spring, что нужно автоматически внедрить зависимость. Spring через Reflection находит поле/конструктор с этой аннотацией и подставляет нужный бин из контекста. QA сталкивается с этим в интеграционных тестах.

- 🔴 **Q:** Можно ли через Reflection изменить значение `final`-поля? Зачем это может понадобиться в тестах?
- **A:** Технически возможно через `Field.setAccessible(true)` и манипуляции с модификаторами, но это хак, который может не работать в новых версиях Java (модульная система). В тестах иногда используется для подмены конфигурации, но лучше использовать моки или DI.

- 🔴 **Q:** Как написать JUnit 5 Extension, использующую кастомную аннотацию?
- **A:** Нужно создать класс, реализующий интерфейс Extension (например, `BeforeEachCallback`), который через Reflection считывает аннотации с тестового метода и выполняет логику. Например, Extension для автоматической привязки скриншотов к Allure-отчёту при падении теста.

---

## Практические задания

### Задание 1 (базовое): Создание кастомной аннотации

Создайте аннотацию `@TestAuthor` с полями `name` и `team`. Примените её к нескольким тестовым методам. Напишите утилитный метод, который через Reflection находит все тестовые методы класса и выводит автора каждого теста.

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface TestAuthor {
    String name();
    String team() default "QA";
}

// Применение
@Test
@TestAuthor(name = "Иван Петров", team = "Backend QA")
void shouldCreateOrder() { /* ... */ }

// Утилита для вывода авторов тестов
public static void printTestAuthors(Class<?> testClass) {
    for (Method method : testClass.getDeclaredMethods()) {
        if (method.isAnnotationPresent(TestAuthor.class)) {
            TestAuthor author = method.getAnnotation(TestAuthor.class);
            System.out.printf("Тест: %s | Автор: %s | Команда: %s%n",
                method.getName(), author.name(), author.team());
        }
    }
}
```

### Задание 2 (среднее): Анализ тестового класса через Reflection

Напишите метод `analyzeTestClass(Class<?> clazz)`, который выводит:
- Количество тестовых методов (с `@Test`)
- Количество параметризованных тестов (с `@ParameterizedTest`)
- Список тегов (с `@Tag`)
- Есть ли `@BeforeEach` / `@AfterEach` методы

### Задание 3 (продвинутое): Кастомная JUnit 5 Extension

Создайте JUnit 5 Extension `TimingExtension`, которая замеряет время выполнения каждого теста и логирует результат. Используйте `BeforeTestExecutionCallback` и `AfterTestExecutionCallback`.

```java
public class TimingExtension implements
        BeforeTestExecutionCallback, AfterTestExecutionCallback {

    @Override
    public void beforeTestExecution(ExtensionContext context) {
        // Сохраняем время начала теста в Store
        context.getStore(ExtensionContext.Namespace.GLOBAL)
            .put("startTime", System.currentTimeMillis());
    }

    @Override
    public void afterTestExecution(ExtensionContext context) {
        long startTime = context.getStore(ExtensionContext.Namespace.GLOBAL)
            .get("startTime", Long.class);
        long duration = System.currentTimeMillis() - startTime;
        System.out.printf("Тест '%s' выполнен за %d мс%n",
            context.getDisplayName(), duration);
    }
}
```

---

## Дополнительные ресурсы

- [Baeldung — Java Annotations](https://www.baeldung.com/java-annotations) — подробный гайд по аннотациям
- [Baeldung — Java Reflection](https://www.baeldung.com/java-reflection) — Reflection API с примерами
- [JUnit 5 User Guide — Annotations](https://junit.org/junit5/docs/current/user-guide/#writing-tests-annotations) — официальная документация JUnit 5
- [Allure Framework — Аннотации](https://docs.qameta.io/allure/) — документация Allure
- [Baeldung — Guide to JUnit 5 Extensions](https://www.baeldung.com/junit-5-extensions) — создание кастомных Extensions
- [Oracle — The Reflection API](https://docs.oracle.com/javase/tutorial/reflect/) — официальный туториал Oracle
