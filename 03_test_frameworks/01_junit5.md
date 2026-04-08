# JUnit 5

## Обзор

JUnit 5 — это современный фреймворк для модульного тестирования в экосистеме Java. В отличие от предыдущих версий, JUnit 5 спроектирован как модульная платформа, состоящая из трёх ключевых компонентов. Это стандарт де-факто для написания автоматизированных тестов на Java и Kotlin, и знание этого фреймворка обязательно для любого QA-инженера, работающего с Java-стеком.

**Минимальная зависимость (Maven):**

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.10.2</version>
    <scope>test</scope>
</dependency>
```

---

## Архитектура JUnit 5

JUnit 5 = **JUnit Platform** + **JUnit Jupiter** + **JUnit Vintage**

### JUnit Platform

Фундамент для запуска тестовых фреймворков на JVM. Предоставляет `TestEngine` API для разработки собственных тестовых движков. Используется IDE и инструментами сборки (Maven Surefire, Gradle) для обнаружения и запуска тестов.

### JUnit Jupiter

Новая модель программирования и расширения для написания тестов в JUnit 5. Содержит все новые аннотации (`@Test`, `@ParameterizedTest`, `@Nested` и т.д.) и механизм расширений (Extensions).

### JUnit Vintage

Обеспечивает обратную совместимость — позволяет запускать тесты, написанные на JUnit 3 и JUnit 4, на платформе JUnit 5. Полезен при постепенной миграции проекта.

```
┌─────────────────────────────────────────────────┐
│              JUnit Platform                     │
│  (запуск, обнаружение тестов, TestEngine API)   │
├────────────────────┬────────────────────────────┤
│   JUnit Jupiter    │      JUnit Vintage         │
│  (JUnit 5 тесты)  │  (JUnit 3/4 совместимость) │
└────────────────────┴────────────────────────────┘
```

---

## Основные аннотации

### @Test

Помечает метод как тестовый. В отличие от JUnit 4, не принимает параметров (нет `expected`, `timeout`).

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class CalculatorTest {

    @Test
    void shouldAddTwoPositiveNumbers() {
        // Arrange — подготовка данных
        Calculator calculator = new Calculator();

        // Act — выполнение действия
        int result = calculator.add(2, 3);

        // Assert — проверка результата
        assertEquals(5, result);
    }
}
```

### @DisplayName

Задаёт читаемое имя теста, которое отображается в отчётах и IDE.

```java
@Test
@DisplayName("Сложение двух отрицательных чисел должно вернуть отрицательное число")
void addNegativeNumbers() {
    assertEquals(-5, calculator.add(-2, -3));
}
```

### @Disabled

Отключает тест или весь тестовый класс. Всегда указывайте причину отключения.

```java
@Test
@Disabled("BUG-1234: ждём исправления бага на бэкенде")
void shouldHandleEdgeCase() {
    // Тест временно отключён
}
```

### @Tag

Позволяет категоризировать тесты для выборочного запуска.

```java
@Test
@Tag("smoke")
void criticalFunctionalityTest() {
    // Этот тест войдёт в smoke-набор
}

@Test
@Tag("regression")
void detailedFunctionalityTest() {
    // Этот тест только для regression
}
```

**Запуск тестов по тегу (Maven):**

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <groups>smoke</groups>
    </configuration>
</plugin>
```

---

## Жизненный цикл теста

### Аннотации жизненного цикла

```java
class LifecycleTest {

    @BeforeAll
    static void setUpOnce() {
        // Выполняется ОДИН РАЗ перед всеми тестами в классе
        // Метод должен быть static (если жизненный цикл PER_METHOD)
        System.out.println("Инициализация общих ресурсов");
    }

    @BeforeEach
    void setUp() {
        // Выполняется ПЕРЕД КАЖДЫМ тестом
        System.out.println("Подготовка к тесту");
    }

    @Test
    void testOne() {
        System.out.println("Тест 1");
    }

    @Test
    void testTwo() {
        System.out.println("Тест 2");
    }

    @AfterEach
    void tearDown() {
        // Выполняется ПОСЛЕ КАЖДОГО теста
        System.out.println("Очистка после теста");
    }

    @AfterAll
    static void tearDownOnce() {
        // Выполняется ОДИН РАЗ после всех тестов
        System.out.println("Освобождение общих ресурсов");
    }
}
```

**Порядок выполнения для двух тестов:**

```
@BeforeAll
  @BeforeEach → testOne → @AfterEach
  @BeforeEach → testTwo → @AfterEach
@AfterAll
```

### Режимы жизненного цикла

По умолчанию для каждого теста создаётся **новый экземпляр** класса (`PER_METHOD`). Можно переключить на `PER_CLASS` — один экземпляр на весь класс:

```java
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class SharedInstanceTest {

    // @BeforeAll и @AfterAll могут быть НЕ static
    @BeforeAll
    void setUpOnce() {
        // Нестатический метод — один экземпляр класса
    }
}
```

---

## @ParameterizedTest

Позволяет запускать один и тот же тест с разными наборами данных. Незаменим для data-driven тестирования.

### @ValueSource

Простейший источник — массив литералов одного типа.

```java
@ParameterizedTest
@ValueSource(strings = {"admin@mail.com", "user@test.org", "qa+tag@company.co"})
@DisplayName("Валидные email-адреса должны проходить валидацию")
void shouldAcceptValidEmails(String email) {
    assertTrue(EmailValidator.isValid(email),
            () -> "Email должен быть валидным: " + email);
}

@ParameterizedTest
@ValueSource(ints = {1, 5, 10, 100})
void shouldReturnPositiveForPositiveNumbers(int number) {
    assertTrue(number > 0);
}
```

### @CsvSource

Несколько параметров в CSV-формате — удобно для пар «вход-выход».

```java
@ParameterizedTest
@CsvSource({
    "1, 1, 2",
    "0, 0, 0",
    "-1, 1, 0",
    "100, 200, 300"
})
@DisplayName("Проверка сложения: {0} + {1} = {2}")
void shouldAddCorrectly(int a, int b, int expected) {
    assertEquals(expected, calculator.add(a, b));
}

@ParameterizedTest
@CsvSource(value = {
    "admin | ADMIN",
    "User  | USER",
    "tEsT  | TEST"
}, delimiter = '|')
void shouldConvertToUpperCase(String input, String expected) {
    assertEquals(expected, input.strip().toUpperCase());
}
```

### @MethodSource

Ссылка на статический метод, возвращающий `Stream`, `Collection` или массив аргументов.

```java
@ParameterizedTest
@MethodSource("provideUserData")
void shouldCreateUserWithValidData(String name, int age, String role) {
    User user = new User(name, age, role);
    assertAll(
        () -> assertNotNull(user),
        () -> assertEquals(name, user.getName()),
        () -> assertTrue(user.getAge() > 0)
    );
}

static Stream<Arguments> provideUserData() {
    return Stream.of(
        Arguments.of("Иван", 25, "TESTER"),
        Arguments.of("Мария", 30, "DEVELOPER"),
        Arguments.of("Пётр", 45, "MANAGER")
    );
}
```

### @EnumSource

Проходит по значениям enum — удобно для тестирования всех статусов, ролей и т.д.

```java
enum OrderStatus { NEW, PROCESSING, SHIPPED, DELIVERED, CANCELLED }

@ParameterizedTest
@EnumSource(OrderStatus.class)
void shouldHaveNonNullName(OrderStatus status) {
    assertNotNull(status.name());
}

@ParameterizedTest
@EnumSource(value = OrderStatus.class, names = {"NEW", "PROCESSING"})
void shouldBeActiveStatus(OrderStatus status) {
    assertTrue(status.ordinal() < 2);
}
```

---

## @Nested классы

Группировка тестов для лучшей читаемости и структурирования. Каждый вложенный класс может иметь свой `@BeforeEach`.

```java
@DisplayName("Тесты UserService")
class UserServiceTest {

    private UserService userService;

    @BeforeEach
    void setUp() {
        userService = new UserService();
    }

    @Nested
    @DisplayName("При создании пользователя")
    class WhenCreatingUser {

        @Test
        @DisplayName("должен создать пользователя с валидными данными")
        void shouldCreateWithValidData() {
            User user = userService.create("Иван", "ivan@mail.com");
            assertNotNull(user.getId());
        }

        @Test
        @DisplayName("должен бросить исключение при пустом email")
        void shouldThrowOnEmptyEmail() {
            assertThrows(IllegalArgumentException.class,
                () -> userService.create("Иван", ""));
        }
    }

    @Nested
    @DisplayName("При удалении пользователя")
    class WhenDeletingUser {

        @Test
        @DisplayName("должен удалить существующего пользователя")
        void shouldDeleteExistingUser() {
            User user = userService.create("Тест", "test@mail.com");
            assertTrue(userService.delete(user.getId()));
        }
    }
}
```

---

## @RepeatedTest

Запуск теста несколько раз — полезно для выявления flaky-тестов и проверок на стабильность.

```java
@RepeatedTest(value = 5, name = "Попытка {currentRepetition} из {totalRepetitions}")
void shouldGenerateUniqueId() {
    String id1 = IdGenerator.generate();
    String id2 = IdGenerator.generate();
    assertNotEquals(id1, id2, "ID должны быть уникальными при каждом вызове");
}
```

---

## Assertions

### Базовые проверки

```java
@Test
void basicAssertions() {
    assertEquals(4, calculator.add(2, 2), "2 + 2 должно равняться 4");
    assertNotEquals(5, calculator.add(2, 2));
    assertTrue(result > 0, "Результат должен быть положительным");
    assertFalse(list.isEmpty(), "Список не должен быть пустым");
    assertNull(user.getMiddleName(), "Отчество может быть null");
    assertNotNull(user.getId(), "ID не должен быть null");
}
```

### assertThrows

Проверка выбрасывания исключений — ключевая операция при тестировании негативных сценариев.

```java
@Test
void shouldThrowOnDivisionByZero() {
    ArithmeticException exception = assertThrows(
        ArithmeticException.class,
        () -> calculator.divide(10, 0)
    );
    assertEquals("/ by zero", exception.getMessage());
}
```

### assertAll — групповые проверки

Выполняет **все** проверки, даже если первая упала. Показывает **все** ошибки разом.

```java
@Test
void shouldReturnCorrectUserInfo() {
    User user = userService.findById(1L);

    assertAll("Проверка полей пользователя",
        () -> assertEquals("Иван", user.getFirstName()),
        () -> assertEquals("Петров", user.getLastName()),
        () -> assertEquals("ivan@mail.com", user.getEmail()),
        () -> assertTrue(user.isActive(), "Пользователь должен быть активен"),
        () -> assertNotNull(user.getCreatedAt(), "Дата создания обязательна")
    );
}
```

### assertTimeout

Проверка, что операция выполняется за допустимое время.

```java
@Test
void shouldRespondWithinTimeout() {
    // Тест упадёт, если операция займёт более 2 секунд
    assertTimeout(Duration.ofSeconds(2), () -> {
        apiClient.fetchData("/api/users");
    });
}

@Test
void shouldRespondQuickly() {
    // Прерывает выполнение сразу при превышении таймаута
    assertTimeoutPreemptively(Duration.ofMillis(500), () -> {
        apiClient.healthCheck();
    });
}
```

---

## Assumptions

Условное выполнение тестов — тест **пропускается** (а не падает), если условие не выполнено.

```java
@Test
void shouldRunOnlyOnLinux() {
    assumeTrue(System.getProperty("os.name").contains("Linux"),
            "Тест только для Linux");
    // Код теста, специфичного для Linux
}

@Test
void shouldConnectToStagingDatabase() {
    assumeFalse(System.getenv("CI") != null,
            "Пропускаем на CI — нет доступа к staging БД");
    // Тест с реальным подключением к базе
}
```

---

## Extensions (Расширения)

Механизм расширений в JUnit 5 заменяет `@Rule` и `@Runner` из JUnit 4.

### Пример кастомного Extension

```java
// Расширение для замера времени выполнения теста
public class TimingExtension implements BeforeEachCallback, AfterEachCallback {

    private static final Map<String, Long> startTimes = new ConcurrentHashMap<>();

    @Override
    public void beforeEach(ExtensionContext context) {
        // Запоминаем время начала теста
        startTimes.put(context.getUniqueId(), System.currentTimeMillis());
    }

    @Override
    public void afterEach(ExtensionContext context) {
        // Вычисляем и выводим длительность
        long start = startTimes.remove(context.getUniqueId());
        long duration = System.currentTimeMillis() - start;
        System.out.printf("Тест '%s' выполнился за %d мс%n",
                context.getDisplayName(), duration);
    }
}
```

### Подключение расширения

```java
@ExtendWith(TimingExtension.class)
class PerformanceTest {

    @Test
    void shouldBeQuick() {
        // Время выполнения этого теста будет залогировано
    }
}
```

### Встроенные расширения

```java
// Для работы с Mockito
@ExtendWith(MockitoExtension.class)
class ServiceTest {
    @Mock
    private Repository repository;
    // ...
}

// Для работы с TempDir
@Test
void shouldWriteToFile(@TempDir Path tempDir) {
    Path file = tempDir.resolve("test.txt");
    Files.writeString(file, "Hello");
    assertTrue(Files.exists(file));
}
```

---

## Параллельное выполнение тестов

Настройка через файл `junit-platform.properties` в `src/test/resources`:

```properties
# Включаем параллельное выполнение
junit.jupiter.execution.parallel.enabled=true

# Стратегия по умолчанию: SAME_THREAD или CONCURRENT
junit.jupiter.execution.parallel.mode.default=concurrent

# Стратегия для верхнеуровневых классов
junit.jupiter.execution.parallel.mode.classes.default=concurrent

# Количество потоков (dynamic — на основе числа процессоров)
junit.jupiter.execution.parallel.config.strategy=dynamic
junit.jupiter.execution.parallel.config.dynamic.factor=1
```

**Управление на уровне теста:**

```java
@Execution(ExecutionMode.CONCURRENT)
class ParallelTests {

    @Test
    void testA() {
        // Выполняется параллельно
    }

    @Test
    void testB() {
        // Выполняется параллельно
    }
}
```

---

## Связь с тестированием

JUnit 5 — это **основной инструмент** QA-инженера в Java-проектах:

- **Unit-тесты**: проверка отдельных методов и классов
- **Интеграционные тесты**: в сочетании с Spring Boot Test, Testcontainers
- **API-тесты**: как каркас для REST Assured, HttpClient
- **UI-тесты**: как runner для Selenium WebDriver, Playwright
- **Параметризация**: data-driven подход к тестированию граничных значений
- **Теги**: разделение smoke / regression / e2e наборов для CI/CD

---

## Типичные ошибки

1. **Забыть импорт из Jupiter** — использование `org.junit.Test` (JUnit 4) вместо `org.junit.jupiter.api.Test`
2. **Не static `@BeforeAll`** — при дефолтном `PER_METHOD` lifecycle эти методы должны быть static
3. **Порядок аргументов в `assertEquals`** — первый аргумент это `expected`, второй `actual`
4. **Отсутствие сообщения в assertion** — без сообщения трудно понять причину падения
5. **Слишком много логики в одном тесте** — один тест должен проверять одну вещь
6. **Не использовать `assertAll`** — при нескольких проверках первый упавший assert скрывает остальные
7. **Зависимости между тестами** — тесты должны быть независимы; не полагайтесь на порядок выполнения
8. **Игнорировать `@Disabled` без причины** — всегда пишите причину отключения теста

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Из каких модулей состоит JUnit 5? Зачем нужен Vintage?
2. Какие аннотации жизненного цикла вы знаете? В каком порядке они вызываются?
3. Чем `@BeforeEach` отличается от `@BeforeAll`?
4. Как отключить тест? Как пометить тест тегом?
5. Как проверить, что метод выбрасывает исключение?

### 🟡 Средний уровень
6. Чем `assertTimeout` отличается от `assertTimeoutPreemptively`?
7. Зачем нужен `assertAll`? Приведите пример.
8. Какие источники данных можно использовать в `@ParameterizedTest`?
9. Для чего нужен `@Nested`? Когда его применять?
10. Как настроить параллельный запуск тестов?

### 🔴 Продвинутый уровень
11. Как написать своё Extension? Какие callback-интерфейсы доступны?
12. Чем lifecycle `PER_METHOD` отличается от `PER_CLASS`? Когда использовать `PER_CLASS`?
13. Как организовать запуск только smoke-тестов на CI? Покажите конфигурацию.
14. Как Resolver-расширения (ParameterResolver) работают и когда их использовать?
15. Как обеспечить изоляцию тестов при параллельном выполнении?

---

## Практические задания

### Задание 1: Базовый тест
Напишите тестовый класс для метода `StringUtils.reverse(String)`. Покройте: обычную строку, пустую строку, `null`, строку с пробелами, палиндром.

### Задание 2: Параметризованный тест
Создайте `@ParameterizedTest` с `@CsvSource` для проверки метода `MathUtils.isPrime(int)`. Включите простые числа, составные числа, граничные случаи (0, 1, 2, отрицательные).

### Задание 3: Nested-тесты
Напишите `@Nested`-структуру для `ShoppingCart`: группы «При пустой корзине», «При добавлении товара», «При удалении товара». Каждая группа — 2-3 теста.

### Задание 4: Custom Extension
Создайте расширение `RetryExtension`, которое повторяет упавший тест до 3 раз (для борьбы с flaky-тестами). Подключите его через `@ExtendWith`.

### Задание 5: Параллельный запуск
Настройте параллельное выполнение тестов в проекте. Обнаружьте и устраните конфликт общего состояния между тестами.

---

## Дополнительные ресурсы

- [JUnit 5 User Guide](https://junit.org/junit5/docs/current/user-guide/)
- [JUnit 5 GitHub](https://github.com/junit-team/junit5)
- [Baeldung — JUnit 5](https://www.baeldung.com/junit-5)
- [JUnit 5 Samples](https://github.com/junit-team/junit5-samples)
- [JUnit 5 Extensions Guide](https://junit.org/junit5/docs/current/user-guide/#extensions)
