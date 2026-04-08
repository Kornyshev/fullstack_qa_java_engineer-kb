# Функциональные интерфейсы и Lambda

## Обзор

Функциональные интерфейсы и lambda-выражения — краеугольные камни функционального программирования
в Java, появившиеся в Java 8. Для QA-инженера эти концепции имеют прямое практическое значение:
условия ожидания в Selenide (`Condition`), фильтрация в Stream API, матчеры в REST Assured,
кастомные валидаторы и retry-механизмы — всё это построено на функциональных интерфейсах.

Функциональный интерфейс — это интерфейс, содержащий ровно один абстрактный метод.
Lambda-выражение — компактная запись анонимной реализации такого интерфейса. Вместе они позволяют
передавать поведение как параметр, делая код лаконичнее и выразительнее.

---

## Функциональный интерфейс — определение

```java
// Функциональный интерфейс — ОДИН абстрактный метод
// Аннотация @FunctionalInterface — необязательна, но рекомендуется
// Она заставляет компилятор проверять, что интерфейс содержит ровно один абстрактный метод

@FunctionalInterface
public interface TestValidator {
    boolean validate(String input); // единственный абстрактный метод

    // default и static методы НЕ считаются абстрактными
    default String getDescription() {
        return "Пользовательский валидатор";
    }

    static TestValidator alwaysTrue() {
        return input -> true;
    }
}

// Использование
TestValidator notEmpty = input -> input != null && !input.isBlank();
boolean isValid = notEmpty.validate("test"); // true
```

---

## Основные функциональные интерфейсы из `java.util.function`

### Predicate<T> — проверка условия

```java
// Принимает T, возвращает boolean
// Метод: boolean test(T t)

import java.util.function.Predicate;

// Простые предикаты
Predicate<String> isNotEmpty = s -> s != null && !s.isEmpty();
Predicate<Integer> isPositive = n -> n > 0;
Predicate<TestResult> isPassed = r -> r.getStatus() == Status.PASSED;

// Проверка
boolean result = isNotEmpty.test("Hello"); // true
boolean check = isPassed.test(new TestResult("test", Status.FAILED)); // false

// Комбинирование предикатов
Predicate<TestResult> isPassedAndFast = isPassed
        .and(r -> r.getDuration() < 1000); // И длительность < 1 сек

Predicate<TestResult> isFailedOrSkipped = isPassed
        .negate(); // отрицание — всё, что не PASSED

Predicate<String> isAlphaOrNumeric = s -> s.matches("[a-zA-Z]+");
Predicate<String> combined = isAlphaOrNumeric.or(s -> s.matches("[0-9]+"));

// Использование в Stream API
List<TestResult> failedTests = results.stream()
        .filter(isPassed.negate()) // все не-PASSED
        .collect(Collectors.toList());

// Использование в Selenide-стиле
public void waitUntil(Predicate<WebElement> condition, Duration timeout) {
    // Ожидание, пока элемент удовлетворяет условию
    long deadline = System.currentTimeMillis() + timeout.toMillis();
    WebElement element = driver.findElement(locator);
    while (System.currentTimeMillis() < deadline) {
        if (condition.test(element)) return;
        try { Thread.sleep(200); } catch (InterruptedException ignored) {}
    }
    throw new TimeoutException("Условие не выполнено за " + timeout);
}

// Вызов
waitUntil(el -> el.isDisplayed() && el.isEnabled(), Duration.ofSeconds(10));
```

### Function<T, R> — преобразование

```java
// Принимает T, возвращает R
// Метод: R apply(T t)

import java.util.function.Function;

// Преобразования
Function<TestResult, String> toReportLine = r ->
        String.format("[%s] %s (%dms)", r.getStatus(), r.getTestName(), r.getDuration());

Function<String, Integer> stringLength = String::length;

Function<WebElement, String> getText = WebElement::getText;

// Применение
String line = toReportLine.apply(new TestResult("login_test", Status.PASSED, 150));
// "[PASSED] login_test (150ms)"

// Композиция функций
Function<String, String> trim = String::trim;
Function<String, String> toLowerCase = String::toLowerCase;
Function<String, String> normalizeInput = trim.andThen(toLowerCase);

String normalized = normalizeInput.apply("  HELLO World  "); // "hello world"

// compose — применяет сначала переданную функцию, потом текущую
Function<String, Integer> trimAndCount = stringLength.compose(trim);
Integer count = trimAndCount.apply("  test  "); // 4 (а не 8)

// Использование в Stream API
List<String> reportLines = results.stream()
        .map(toReportLine)  // Function<TestResult, String>
        .collect(Collectors.toList());
```

### Consumer<T> — потребление значения

```java
// Принимает T, ничего не возвращает (void)
// Метод: void accept(T t)

import java.util.function.Consumer;

// Потребители
Consumer<String> print = System.out::println;
Consumer<TestResult> logResult = r ->
        logger.info("Тест: {} — Статус: {}", r.getTestName(), r.getStatus());

Consumer<WebElement> highlight = element -> {
    // Подсветка элемента на странице (для отладки)
    JavascriptExecutor js = (JavascriptExecutor) driver;
    js.executeScript("arguments[0].style.border='3px solid red'", element);
};

// Применение
print.accept("Привет!"); // выводит "Привет!"
logResult.accept(testResult);

// Цепочка Consumer'ов
Consumer<TestResult> logAndSave = logResult
        .andThen(r -> saveToDatabase(r)); // сначала залогировать, потом сохранить

// Использование в forEach
results.forEach(logResult);

// Использование в Selenium — действие над каждым элементом
driver.findElements(By.className("error-message")).forEach(element -> {
    logger.warn("Ошибка на странице: {}", element.getText());
});
```

### Supplier<T> — поставщик значений

```java
// Не принимает аргументов, возвращает T
// Метод: T get()

import java.util.function.Supplier;

// Поставщики
Supplier<String> randomEmail = () -> "user_" + UUID.randomUUID() + "@test.com";
Supplier<WebDriver> driverFactory = () -> new ChromeDriver();
Supplier<LocalDateTime> now = LocalDateTime::now;

// Применение
String email = randomEmail.get(); // "user_a1b2c3d4...@test.com"
WebDriver driver = driverFactory.get();

// Ленивая инициализация
Supplier<ExpensiveResource> lazyResource = () -> {
    System.out.println("Создаём ресурс (только при первом обращении)");
    return new ExpensiveResource();
};

// Supplier в Optional
Optional<String> empty = Optional.empty();
String value = empty.orElseGet(() -> "значение по умолчанию"); // Supplier вызывается только если empty

// Генерация тестовых данных
Supplier<User> testUserFactory = () -> new User(
        "TestUser_" + System.currentTimeMillis(),
        randomEmail.get(),
        "Password123!"
);

List<User> testUsers = Stream.generate(testUserFactory)
        .limit(10)
        .collect(Collectors.toList());
```

### UnaryOperator<T> — преобразование в тот же тип

```java
// Частный случай Function<T, T> — входной и выходной тип совпадают
// Метод: T apply(T t)

import java.util.function.UnaryOperator;

UnaryOperator<String> addPrefix = s -> "TEST_" + s;
UnaryOperator<String> toUpper = String::toUpperCase;
UnaryOperator<Integer> doubleIt = n -> n * 2;

// Применение
String prefixed = addPrefix.apply("login"); // "TEST_login"

// Композиция
UnaryOperator<String> prefixAndUpper = addPrefix.andThen(toUpper)::apply;
// Или проще:
Function<String, String> prefixAndUpperF = addPrefix.andThen(toUpper);

// Использование в List.replaceAll
List<String> names = new ArrayList<>(List.of("login_test", "logout_test"));
names.replaceAll(addPrefix); // ["TEST_login_test", "TEST_logout_test"]
```

### Сводная таблица

| Интерфейс | Метод | Вход | Выход | Пример |
|---|---|---|---|---|
| `Predicate<T>` | `test(T)` | T | boolean | Фильтрация тестов |
| `Function<T,R>` | `apply(T)` | T | R | Маппинг результатов |
| `Consumer<T>` | `accept(T)` | T | void | Логирование |
| `Supplier<T>` | `get()` | — | T | Фабрика данных |
| `UnaryOperator<T>` | `apply(T)` | T | T | Трансформация строк |
| `BiFunction<T,U,R>` | `apply(T,U)` | T, U | R | Сравнение двух объектов |
| `BiPredicate<T,U>` | `test(T,U)` | T, U | boolean | Проверка пары значений |
| `BiConsumer<T,U>` | `accept(T,U)` | T, U | void | Заполнение формы (поле, значение) |

---

## Lambda-выражения — синтаксис

### Базовый синтаксис

```java
// Полная форма
(ТипПараметра параметр1, ТипПараметра параметр2) -> { тело метода; return результат; }

// Примеры от полной к сокращённой:

// 1. Полная форма
Comparator<String> comp1 = (String a, String b) -> { return a.compareTo(b); };

// 2. Без типов параметров (компилятор выведет)
Comparator<String> comp2 = (a, b) -> { return a.compareTo(b); };

// 3. Без фигурных скобок и return (для одного выражения)
Comparator<String> comp3 = (a, b) -> a.compareTo(b);

// 4. Один параметр — можно без скобок
Predicate<String> pred = s -> s.isEmpty();

// 5. Без параметров
Supplier<String> sup = () -> "Hello";

// 6. Многострочная lambda
Function<TestResult, String> formatter = result -> {
    String status = result.getStatus().name();
    String name = result.getTestName();
    long duration = result.getDuration();
    return String.format("[%s] %s — %dms", status, name, duration);
};
```

### Замыкание (Closure) — доступ к внешним переменным

```java
// Lambda может использовать внешние переменные, если они effectively final
String prefix = "TEST"; // effectively final — не изменяется после присваивания
Function<String, String> addPrefix = s -> prefix + "_" + s; // ОК

// String mutable = "TEST";
// mutable = "PROD"; // теперь переменная не effectively final
// Function<String, String> fn = s -> mutable + s; // ОШИБКА КОМПИЛЯЦИИ
```

---

## Method References (ссылки на методы)

Method reference — сокращённая запись lambda, когда lambda просто вызывает существующий метод.

### Четыре типа ссылок на методы

```java
// 1. Ссылка на статический метод: ИмяКласса::статическийМетод
Function<String, Integer> parseInt = Integer::parseInt;
// Эквивалент: s -> Integer.parseInt(s)

// 2. Ссылка на метод экземпляра конкретного объекта: объект::метод
String prefix = "TEST";
Predicate<String> startsWith = prefix::startsWith;
// Эквивалент: s -> prefix.startsWith(s)

// 3. Ссылка на метод экземпляра произвольного объекта: ИмяКласса::метод
Function<String, Integer> length = String::length;
// Эквивалент: s -> s.length()

Predicate<String> isEmpty = String::isEmpty;
// Эквивалент: s -> s.isEmpty()

// 4. Ссылка на конструктор: ИмяКласса::new
Supplier<ArrayList<String>> listFactory = ArrayList::new;
// Эквивалент: () -> new ArrayList<>()

Function<String, User> userFactory = User::new;
// Эквивалент: name -> new User(name)
```

### Примеры из тестирования

```java
// Stream API с method references
List<String> testNames = results.stream()
        .map(TestResult::getTestName)          // Тип 3: метод экземпляра
        .filter(Predicate.not(String::isEmpty)) // Тип 3 + статический метод Predicate
        .sorted(String::compareTo)              // Тип 3: метод экземпляра
        .collect(Collectors.toList());

// Selenium — получение текстов элементов
List<String> texts = driver.findElements(By.className("item")).stream()
        .map(WebElement::getText)   // method reference вместо el -> el.getText()
        .collect(Collectors.toList());

// Логирование
results.forEach(System.out::println); // method reference на метод экземпляра System.out
```

---

## Optional — безопасная работа с null

### Создание Optional

```java
import java.util.Optional;

// 1. Optional.of() — значение НЕ может быть null
Optional<String> name = Optional.of("TestUser");
// Optional.of(null); // NullPointerException!

// 2. Optional.ofNullable() — значение МОЖЕТ быть null
Optional<String> maybeName = Optional.ofNullable(getUserName()); // может быть empty

// 3. Optional.empty() — пустой Optional
Optional<String> empty = Optional.empty();
```

### Основные методы Optional

```java
Optional<String> opt = Optional.ofNullable(getValue());

// isPresent() / isEmpty() (Java 11+) — проверка наличия значения
if (opt.isPresent()) {
    System.out.println(opt.get());
}

// ifPresent() — выполнить действие, если значение есть
opt.ifPresent(value -> logger.info("Значение: {}", value));

// ifPresentOrElse() (Java 9+) — действие для обоих случаев
opt.ifPresentOrElse(
        value -> logger.info("Найдено: {}", value),
        () -> logger.warn("Значение отсутствует")
);

// orElse() — вернуть значение по умолчанию
String result = opt.orElse("default");

// orElseGet() — ленивое вычисление значения по умолчанию (Supplier)
String result2 = opt.orElseGet(() -> computeExpensiveDefault());
// Supplier вызывается ТОЛЬКО если Optional пуст

// orElseThrow() — бросить исключение, если пуст
String result3 = opt.orElseThrow(() ->
        new TestDataException("config", "Обязательное значение отсутствует"));

// orElseThrow() без аргументов (Java 10+) — бросает NoSuchElementException
String result4 = opt.orElseThrow();

// or() (Java 9+) — альтернативный Optional
Optional<String> fallback = opt.or(() -> Optional.of("fallback_value"));
```

### Преобразование и фильтрация

```java
// map() — преобразование значения внутри Optional
Optional<String> upperName = Optional.of("test")
        .map(String::toUpperCase); // Optional["TEST"]

// Цепочка map — безопасная навигация по вложенным объектам
Optional<String> city = Optional.ofNullable(user)
        .map(User::getAddress)
        .map(Address::getCity)
        .map(String::toUpperCase);
// Если любой из промежуточных результатов null — вернётся Optional.empty()

// flatMap() — когда внутренний метод уже возвращает Optional
Optional<String> email = Optional.ofNullable(user)
        .flatMap(User::getEmail); // getEmail() возвращает Optional<String>

// filter() — фильтрация значения
Optional<String> longName = Optional.of("TestAutomation")
        .filter(s -> s.length() > 5); // Optional["TestAutomation"]

Optional<String> filtered = Optional.of("hi")
        .filter(s -> s.length() > 5); // Optional.empty()
```

### Практический пример в тестах

```java
// Безопасный поиск элемента с Optional
public Optional<WebElement> safeFindElement(By locator) {
    try {
        return Optional.of(driver.findElement(locator));
    } catch (NoSuchElementException e) {
        return Optional.empty();
    }
}

// Использование
String buttonText = safeFindElement(By.id("submit"))
        .filter(WebElement::isDisplayed)
        .map(WebElement::getText)
        .orElse("Кнопка не найдена");

// Безопасное получение конфигурации
public String getConfigValue(String key) {
    return Optional.ofNullable(System.getProperty(key))
            .or(() -> Optional.ofNullable(System.getenv(key)))
            .or(() -> Optional.ofNullable(propertiesFile.getProperty(key)))
            .orElseThrow(() -> new TestConfigurationException(key));
}
```

### Антипаттерны Optional

```java
// ПЛОХО: Optional как параметр метода
public void process(Optional<String> name) { } // Не делайте так!

// ХОРОШО: используйте перегрузку или nullable
public void process(String name) { }

// ПЛОХО: Optional.get() без проверки
String value = optional.get(); // NoSuchElementException если пуст!

// ХОРОШО: orElse / orElseThrow / ifPresent
String value = optional.orElse("default");

// ПЛОХО: Optional для полей класса
private Optional<String> name; // Не стоит

// ХОРОШО: nullable поле + Optional в геттере
private String name;
public Optional<String> getName() { return Optional.ofNullable(name); }

// ПЛОХО: isPresent + get
if (optional.isPresent()) {
    doSomething(optional.get());
}

// ХОРОШО: ifPresent
optional.ifPresent(this::doSomething);
```

---

## Применение в тестовой автоматизации

### Selenide — пользовательские условия

```java
// Selenide Condition — это фактически Predicate над WebElement
import com.codeborne.selenide.Condition;

// Кастомное условие через Condition.match
Condition hasMinLength = Condition.match("минимальная длина текста 5",
        el -> el.getText().length() >= 5);

// Использование
$("h1").shouldHave(hasMinLength);

// Кастомный Condition через наследование
public class AttributeContains extends Condition {
    private final String attribute;
    private final String expectedValue;

    public AttributeContains(String attribute, String expectedValue) {
        super("attribute '" + attribute + "' contains '" + expectedValue + "'");
        this.attribute = attribute;
        this.expectedValue = expectedValue;
    }

    @Override
    public boolean apply(Driver driver, WebElement element) {
        String actual = element.getAttribute(attribute);
        return actual != null && actual.contains(expectedValue);
    }
}

// Использование
$("input").shouldHave(new AttributeContains("class", "active"));
```

### Кастомные ожидания с функциональными интерфейсами

```java
// Универсальный waiter на основе Supplier и Predicate
public class SmartWait {

    public static <T> T until(Supplier<T> action, Predicate<T> condition,
                               Duration timeout, Duration pollingInterval,
                               String description) {
        long deadline = System.currentTimeMillis() + timeout.toMillis();
        T lastResult = null;
        Exception lastException = null;

        while (System.currentTimeMillis() < deadline) {
            try {
                lastResult = action.get();
                if (condition.test(lastResult)) {
                    return lastResult;
                }
            } catch (Exception e) {
                lastException = e;
            }
            try {
                Thread.sleep(pollingInterval.toMillis());
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new RuntimeException("Ожидание прервано", e);
            }
        }

        String message = String.format("Таймаут (%s): %s. Последний результат: %s",
                timeout, description, lastResult);
        if (lastException != null) {
            throw new TimeoutException(message, lastException);
        }
        throw new TimeoutException(message);
    }
}

// Использование
// Ждём, пока API вернёт статус "COMPLETED"
String status = SmartWait.until(
        () -> apiClient.getStatus(orderId),          // Supplier — что делать
        "COMPLETED"::equals,                          // Predicate — какое условие ждём
        Duration.ofSeconds(30),                       // таймаут
        Duration.ofMillis(500),                       // интервал опроса
        "Статус заказа станет COMPLETED"              // описание для ошибки
);
```

### REST Assured — матчеры с lambda

```java
// REST Assured с Hamcrest-матчерами и lambda
given()
        .when()
        .get("/api/users")
        .then()
        .statusCode(200)
        .body("users.size()", greaterThan(0))
        .body("users.name", everyItem(not(emptyString())));

// Кастомная валидация ответа с Consumer
Consumer<ValidatableResponse> validateUserList = response -> {
    response.statusCode(200);
    response.body("users", notNullValue());
    response.body("users.size()", greaterThan(0));
};

ValidatableResponse response = given()
        .when()
        .get("/api/users")
        .then();
validateUserList.accept(response);

// Кастомная десериализация с Function
Function<Response, List<User>> extractUsers = resp ->
        resp.jsonPath().getList("users", User.class);

List<User> users = extractUsers.apply(
        given().when().get("/api/users").then().extract().response()
);
```

### Параметризованная фабрика тестовых данных

```java
// Фабрика тестовых данных на основе Supplier
public class TestDataFactory {

    private static final Map<String, Supplier<?>> factories = new HashMap<>();

    public static <T> void register(String key, Supplier<T> factory) {
        factories.put(key, factory);
    }

    @SuppressWarnings("unchecked")
    public static <T> T create(String key) {
        Supplier<?> factory = factories.get(key);
        if (factory == null) {
            throw new TestDataException("factory", "Фабрика не найдена: " + key);
        }
        return (T) factory.get();
    }

    // Инициализация
    static {
        register("admin", () -> new User("admin", "admin@test.com", "Admin123!"));
        register("regular_user", () -> new User(
                "user_" + System.currentTimeMillis(),
                "user_" + UUID.randomUUID() + "@test.com",
                "Pass123!"
        ));
    }
}

// Использование
User admin = TestDataFactory.create("admin");
User user = TestDataFactory.create("regular_user");
```

---

## Связь с тестированием

- **Stream API** — `filter`, `map`, `forEach` принимают функциональные интерфейсы
- **Selenide/Selenium** — условия ожидания, кастомные `Condition`, `ExpectedCondition`
- **REST Assured** — валидация ответов, извлечение данных, кастомные матчеры
- **JUnit 5** — `assertThrows` принимает `Executable` (функциональный интерфейс)
- **AssertJ** — `satisfies`, `allSatisfy`, `extracting` работают с функциональными интерфейсами
- **Retry-механизмы** — `Supplier<T>` для оборачивания повторяемых действий
- **Page Object** — кастомные ожидания и валидации на основе `Predicate`

---

## Типичные ошибки

1. **Слишком сложные lambda** — если lambda занимает более 3 строк, лучше выделить метод
```java
// ПЛОХО — нечитаемо
results.stream().filter(r -> {
    boolean a = r.getStatus() == Status.PASSED;
    boolean b = r.getDuration() < 5000;
    boolean c = r.getModule().equals("payment");
    return a && b && c;
}).collect(Collectors.toList());

// ХОРОШО — именованный предикат
Predicate<TestResult> isQuickPassedPayment = r ->
        r.getStatus() == Status.PASSED
        && r.getDuration() < 5000
        && r.getModule().equals("payment");

results.stream().filter(isQuickPassedPayment).collect(Collectors.toList());
```

2. **Побочные эффекты в lambda** — lambda должны быть по возможности чистыми функциями

3. **Игнорирование Optional** — использование `get()` без проверки

4. **Optional.of(null)** — бросает NPE; используйте `Optional.ofNullable()`

5. **Путаница `orElse` и `orElseGet`** — `orElse` всегда вычисляет значение, `orElseGet` — лениво
```java
// orElse — computeDefault() вызывается ВСЕГДА, даже если Optional не пуст
String val1 = optional.orElse(computeDefault());

// orElseGet — computeDefault() вызывается ТОЛЬКО если Optional пуст
String val2 = optional.orElseGet(() -> computeDefault());
```

6. **Не использовать method reference там, где можно** — `s -> s.length()` вместо `String::length`

7. **Мутирование внешних переменных в lambda** — нарушает функциональный стиль, чревато ошибками в параллельных stream

---

## Вопросы на интервью

### Уровень Junior

- Что такое функциональный интерфейс? Приведите примеры.
- Что такое lambda-выражение? Как оно связано с функциональным интерфейсом?
- Назовите основные функциональные интерфейсы из `java.util.function`.
- Что такое Optional и зачем он нужен?
- В чём разница между `orElse` и `orElseGet`?
- Что такое method reference? Перечислите виды.

### Уровень Middle

- В чём разница между `Predicate`, `Function`, `Consumer` и `Supplier`? Когда что использовать?
- Как комбинировать предикаты (`and`, `or`, `negate`)?
- Как работает замыкание (closure) в lambda? Что такое effectively final?
- Почему `Optional.of(null)` бросает NPE? Когда использовать `of` vs `ofNullable`?
- Как lambda-выражения реализованы в байткоде (invokedynamic)?
- Напишите кастомный функциональный интерфейс для валидации тестовых данных.

### Уровень Senior

- Чем lambda отличается от анонимного класса на уровне реализации?
- Как lambda-выражения влияют на производительность? Что такое `invokedynamic`?
- Как реализовать каррирование (currying) в Java с помощью функциональных интерфейсов?
- Как `Optional` взаимодействует со Stream API (`stream()` в Java 9+)?
- Как спроектировать fluent API на основе функциональных интерфейсов для тестового фреймворка?
- Как реализовать memoization (кеширование результатов) с помощью `Function`?

---

## Практические задания

1. **Predicate Chain:** Создайте набор предикатов для фильтрации тестовых результатов: `byStatus(Status)`, `byModule(String)`, `byMaxDuration(long)`. Скомбинируйте их для получения "быстрых пройденных тестов модуля payment".

2. **Custom Waiter:** Реализуйте метод `pollUntil(Supplier<T> action, Predicate<T> condition, Duration timeout)`, который опрашивает `action` каждые 500мс, пока `condition` не станет `true` или не истечёт `timeout`.

3. **Optional Pipeline:** Напишите метод, который по цепочке ищет конфигурационное значение: System property -> environment variable -> config file -> default value. Используйте `Optional.or()`.

4. **Method References:** Перепишите следующий код с использованием method references:
   ```java
   list.stream()
       .filter(s -> s != null)
       .map(s -> s.trim())
       .filter(s -> !s.isEmpty())
       .map(s -> s.toUpperCase())
       .forEach(s -> System.out.println(s));
   ```

5. **Functional Test Builder:** Создайте builder для тестовых данных, используя `Consumer<T>`:
   ```java
   User user = TestBuilder.build(User::new, u -> {
       u.setName("Test");
       u.setEmail("test@test.com");
   });
   ```

---

## Дополнительные ресурсы

- [Oracle — Lambda Expressions](https://docs.oracle.com/javase/tutorial/java/javaOO/lambdaexpressions.html)
- [Baeldung — Functional Interfaces in Java](https://www.baeldung.com/java-8-functional-interfaces)
- [Baeldung — Guide to Java Optional](https://www.baeldung.com/java-optional)
- [Baeldung — Java Method References](https://www.baeldung.com/java-method-references)
- Книга: "Modern Java in Action" — Manning Publications (главы 2-3, 11)
- [Java 8 in Action — Lambda Quick Reference](https://www.manning.com/books/java-8-in-action)
