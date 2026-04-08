# Generics в Java

## Обзор

Generics (обобщения) — механизм параметризации типов, появившийся в Java 5. Он позволяет создавать
классы, интерфейсы и методы, которые работают с разными типами данных, сохраняя при этом
типобезопасность на этапе компиляции. Для QA-инженера понимание generics необходимо для работы
с тестовыми фреймворками: `List<WebElement>`, `Page<T>` в Page Object'ах, `Response<T>`
в REST-клиентах, обобщённые утилиты для тестовых данных — всё это основано на generics.

Без generics мы бы работали с "сырыми" типами (raw types), теряя проверку типов компилятором
и рискуя получить `ClassCastException` во время выполнения.

---

## Зачем нужны Generics

### Проблема без generics

```java
// Без generics — используем "сырой" тип (raw type)
List testResults = new ArrayList();
testResults.add(new TestResult("login_test", Status.PASSED));
testResults.add("случайная строка"); // компилятор НЕ ругается!

// При извлечении — нужен явный каст, риск ClassCastException
TestResult result = (TestResult) testResults.get(0); // ОК
TestResult oops = (TestResult) testResults.get(1);   // ClassCastException в рантайме!
```

### Решение с generics

```java
// С generics — типобезопасность на этапе компиляции
List<TestResult> testResults = new ArrayList<>();
testResults.add(new TestResult("login_test", Status.PASSED));
// testResults.add("случайная строка"); // ОШИБКА КОМПИЛЯЦИИ!

// При извлечении — каст не нужен
TestResult result = testResults.get(0); // безопасно, тип гарантирован
```

### Преимущества

| Без Generics | С Generics |
|---|---|
| Ошибки в рантайме (`ClassCastException`) | Ошибки на этапе компиляции |
| Явное приведение типов (casting) | Автоматическое определение типа |
| Нет документирования ожидаемого типа | Тип виден в сигнатуре |
| Код менее читаемый | Код самодокументирующийся |

---

## Generic-классы

### Объявление generic-класса

```java
// T — параметр типа (type parameter)
// Общепринятые имена: T (Type), E (Element), K (Key), V (Value), R (Result)
public class TestDataContainer<T> {
    private final T data;
    private final String description;

    public TestDataContainer(T data, String description) {
        this.data = data;
        this.description = description;
    }

    public T getData() {
        return data;
    }

    public String getDescription() {
        return description;
    }
}

// Использование с разными типами
TestDataContainer<String> stringData = new TestDataContainer<>("admin", "Логин администратора");
TestDataContainer<Integer> intData = new TestDataContainer<>(42, "Код ответа");
TestDataContainer<User> userData = new TestDataContainer<>(new User("John"), "Тестовый пользователь");

// Извлечение — без кастинга, тип определён
String login = stringData.getData();    // String
Integer code = intData.getData();       // Integer
User user = userData.getData();         // User
```

### Generic-класс с несколькими параметрами

```java
// Пара "ключ-значение" для хранения тестовых параметров
public class TestParameter<K, V> {
    private final K key;
    private final V value;

    public TestParameter(K key, V value) {
        this.key = key;
        this.value = value;
    }

    public K getKey() { return key; }
    public V getValue() { return value; }

    @Override
    public String toString() {
        return key + " = " + value;
    }
}

// Использование
TestParameter<String, Integer> timeout = new TestParameter<>("timeout", 30);
TestParameter<String, Boolean> headless = new TestParameter<>("headless", true);
```

### Пример: обобщённый Page Object

```java
// Базовый Page Object с возвращением типа наследника
public abstract class BasePage<T extends BasePage<T>> {
    protected WebDriver driver;

    public BasePage(WebDriver driver) {
        this.driver = driver;
    }

    // Возвращает this с правильным типом — для цепочки вызовов
    @SuppressWarnings("unchecked")
    protected T self() {
        return (T) this;
    }

    public T waitForPageLoad() {
        // Ожидание загрузки страницы
        new WebDriverWait(driver, Duration.ofSeconds(10))
                .until(d -> ((JavascriptExecutor) d)
                        .executeScript("return document.readyState").equals("complete"));
        return self();
    }
}

// Конкретная страница
public class LoginPage extends BasePage<LoginPage> {

    public LoginPage(WebDriver driver) {
        super(driver);
    }

    public LoginPage enterUsername(String username) {
        driver.findElement(By.id("username")).sendKeys(username);
        return self(); // возвращает LoginPage, а не BasePage
    }

    public LoginPage enterPassword(String password) {
        driver.findElement(By.id("password")).sendKeys(password);
        return self();
    }
}

// Цепочка вызовов с правильными типами
LoginPage loginPage = new LoginPage(driver)
        .waitForPageLoad()      // возвращает LoginPage, не BasePage
        .enterUsername("admin")
        .enterPassword("pass");
```

---

## Generic-методы

```java
public class TestUtils {

    // Generic-метод — параметр типа объявляется ПЕРЕД возвращаемым типом
    public static <T> T getRandomElement(List<T> list) {
        if (list == null || list.isEmpty()) {
            throw new IllegalArgumentException("Список не может быть пустым");
        }
        Random random = new Random();
        return list.get(random.nextInt(list.size()));
    }

    // Generic-метод с несколькими параметрами
    public static <T, R> R transform(T input, Function<T, R> transformer) {
        return transformer.apply(input);
    }

    // Использование
    public static void main(String[] args) {
        List<String> browsers = List.of("Chrome", "Firefox", "Safari");
        String randomBrowser = getRandomElement(browsers); // T определён как String

        List<Integer> ports = List.of(8080, 8081, 8082);
        Integer randomPort = getRandomElement(ports); // T определён как Integer

        // Преобразование строки в длину
        Integer length = transform("Hello", String::length); // T=String, R=Integer
    }
}
```

---

## Type Erasure (стирание типов)

Generics в Java существуют **только на этапе компиляции**. Во время выполнения вся информация
о типовых параметрах стирается — это называется **type erasure**.

```java
// На этапе компиляции:
List<String> strings = new ArrayList<>();
List<Integer> integers = new ArrayList<>();

// После стирания типов (в байткоде):
// List strings = new ArrayList();
// List integers = new ArrayList();

// Поэтому:
System.out.println(strings.getClass() == integers.getClass()); // true!
// Оба — java.util.ArrayList

// Нельзя использовать instanceof с параметром типа
// if (obj instanceof List<String>) {} // ОШИБКА КОМПИЛЯЦИИ
if (obj instanceof List<?>) {} // Можно — wildcard

// Нельзя создать массив generic-типа
// T[] array = new T[10]; // ОШИБКА КОМПИЛЯЦИИ

// Нельзя использовать параметр типа в static-контексте
// private static T value; // ОШИБКА КОМПИЛЯЦИИ
```

### Последствия для QA

```java
// Type erasure влияет на десериализацию в тестах API
// Проблема: информация о типе теряется
public <T> T parseJson(String json, Class<T> clazz) {
    // Правильно: передаём Class<T> как параметр
    return new ObjectMapper().readValue(json, clazz);
}

// Использование
User user = parseJson(jsonString, User.class);

// Для generic-типов нужен TypeReference (Jackson)
List<User> users = new ObjectMapper().readValue(
        jsonString,
        new TypeReference<List<User>>() {} // анонимный класс сохраняет информацию о типе
);
```

---

## Ограниченные типы (Bounded Types)

### Upper Bounded — `extends`

```java
// T может быть Number или любым его наследником (Integer, Double, Long...)
public static <T extends Number> double sum(List<T> numbers) {
    return numbers.stream()
            .mapToDouble(Number::doubleValue)
            .sum();
}

// Использование
sum(List.of(1, 2, 3));           // Integer extends Number — ОК
sum(List.of(1.5, 2.5, 3.5));    // Double extends Number — ОК
// sum(List.of("a", "b", "c")); // String не extends Number — ОШИБКА

// Несколько ограничений (класс + интерфейсы)
// Класс должен быть первым, интерфейсы — после &
public static <T extends Comparable<T> & Serializable> T findMax(List<T> list) {
    return list.stream()
            .max(Comparable::compareTo)
            .orElseThrow();
}
```

### Пример из тестирования: ограниченный тип для страниц

```java
// Метод принимает только наследников BasePage
public static <T extends BasePage<?>> T openPage(WebDriver driver, Class<T> pageClass) {
    try {
        T page = pageClass.getDeclaredConstructor(WebDriver.class).newInstance(driver);
        return page.waitForPageLoad();
    } catch (ReflectiveOperationException e) {
        throw new TestFrameworkException("Не удалось создать страницу: " + pageClass.getName(), e);
    }
}

// Использование
LoginPage loginPage = openPage(driver, LoginPage.class);
DashboardPage dashboard = openPage(driver, DashboardPage.class);
```

---

## Wildcards (подстановочные знаки)

### Unbounded Wildcard — `?`

```java
// Принимает список ЛЮБОГО типа
public static void printAll(List<?> list) {
    for (Object item : list) {
        System.out.println(item);
    }
}

// Можно передать любой список
printAll(List.of("a", "b", "c"));
printAll(List.of(1, 2, 3));
printAll(List.of(new User("John")));
```

### Upper Bounded Wildcard — `? extends T`

```java
// Читать можно, записывать — нет (producer)
// "Всё, что является T или его потомком"

public static double calculateAvgDuration(List<? extends TestResult> results) {
    // Можно ЧИТАТЬ как TestResult
    return results.stream()
            .mapToLong(TestResult::getDuration)
            .average()
            .orElse(0.0);

    // results.add(new TestResult(...)); // ОШИБКА КОМПИЛЯЦИИ — нельзя добавлять
}

// Можно передать List<TestResult> или List<ExtendedTestResult>
List<TestResult> basic = getResults();
List<ExtendedTestResult> extended = getExtendedResults(); // ExtendedTestResult extends TestResult

calculateAvgDuration(basic);    // ОК
calculateAvgDuration(extended); // ОК
```

### Lower Bounded Wildcard — `? super T`

```java
// Записывать можно, читать как конкретный тип — нет (consumer)
// "Всё, что является T или его предком"

public static void addDefaults(List<? super TestResult> list) {
    // Можно ЗАПИСЫВАТЬ TestResult и его потомков
    list.add(new TestResult("default_test", Status.PASSED));
    list.add(new ExtendedTestResult("extended_test", Status.PASSED, "module"));

    // Object item = list.get(0); // Можно прочитать только как Object
}

// Можно передать List<TestResult> или List<Object>
List<TestResult> results = new ArrayList<>();
List<Object> objects = new ArrayList<>();

addDefaults(results); // ОК
addDefaults(objects);  // ОК
```

---

## Принцип PECS (Producer Extends, Consumer Super)

Мнемоническое правило для выбора между `? extends` и `? super`:

- **Producer** (источник данных) — используйте `? extends T` — из коллекции **читают**
- **Consumer** (приёмник данных) — используйте `? super T` — в коллекцию **пишут**

```java
// Копирование элементов из source в destination
public static <T> void copy(List<? extends T> source, List<? super T> destination) {
    // source — Producer (читаем из него) → extends
    // destination — Consumer (пишем в него) → super
    for (T item : source) {
        destination.add(item);
    }
}

// Пример из реальных тестов: передача тестовых данных в обработчик
public static void processResults(
        List<? extends TestResult> input,   // Producer — читаем результаты
        List<? super ReportEntry> output     // Consumer — записываем отчёты
) {
    for (TestResult result : input) {
        output.add(new ReportEntry(result.getTestName(), result.getStatus()));
    }
}
```

### Шпаргалка PECS

| Ситуация | Wildcard | Операция | Пример |
|---|---|---|---|
| Только чтение | `? extends T` | get | `List<? extends Number>` |
| Только запись | `? super T` | add | `List<? super Integer>` |
| Чтение и запись | Без wildcard | get + add | `List<T>` |
| Неважно какой тип | `?` | Только Object-методы | `List<?>` |

---

## Где QA встречает Generics

### 1. Коллекции WebElement

```java
// List<WebElement> — самый частый generic в Selenium
List<WebElement> buttons = driver.findElements(By.tagName("button"));

// Stream API с generic-коллекциями
List<String> buttonTexts = buttons.stream()
        .filter(WebElement::isDisplayed)
        .map(WebElement::getText)
        .collect(Collectors.toList());
```

### 2. REST API — Response с типом

```java
// REST Assured с generic-ответом (через TypeRef)
List<User> users = given()
        .when()
        .get("/api/users")
        .then()
        .statusCode(200)
        .extract()
        .as(new TypeRef<List<User>>() {});

// Свой generic-класс для API-ответа
public class ApiResponse<T> {
    private int statusCode;
    private T data;
    private String message;

    // Геттеры и сеттеры
    public T getData() { return data; }
    public int getStatusCode() { return statusCode; }
}

// Использование
ApiResponse<User> userResponse = apiClient.get("/users/1", User.class);
User user = userResponse.getData(); // типобезопасно
```

### 3. Обобщённые утилиты для тестов

```java
// Утилита ожидания с generic-результатом
public static <T> T waitFor(Supplier<T> action, Predicate<T> condition,
                             Duration timeout, String description) {
    long deadline = System.currentTimeMillis() + timeout.toMillis();
    T result = null;
    while (System.currentTimeMillis() < deadline) {
        result = action.get();
        if (condition.test(result)) {
            return result;
        }
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        }
    }
    throw new TimeoutException("Не дождались выполнения условия: " + description);
}

// Использование
String status = waitFor(
        () -> apiClient.getOrderStatus(orderId),    // действие
        s -> s.equals("COMPLETED"),                  // условие
        Duration.ofSeconds(30),                       // таймаут
        "Статус заказа = COMPLETED"                  // описание
);
```

### 4. Data Provider с generics

```java
// Обобщённый загрузчик тестовых данных
public class TestDataLoader {

    private static final ObjectMapper mapper = new ObjectMapper();

    // Загрузка одного объекта
    public static <T> T loadFromJson(String filePath, Class<T> type) {
        try (InputStream is = TestDataLoader.class.getResourceAsStream(filePath)) {
            return mapper.readValue(is, type);
        } catch (IOException e) {
            throw new TestDataException(filePath, "Не удалось загрузить данные");
        }
    }

    // Загрузка списка объектов
    public static <T> List<T> loadListFromJson(String filePath, Class<T> elementType) {
        try (InputStream is = TestDataLoader.class.getResourceAsStream(filePath)) {
            JavaType listType = mapper.getTypeFactory()
                    .constructCollectionType(List.class, elementType);
            return mapper.readValue(is, listType);
        } catch (IOException e) {
            throw new TestDataException(filePath, "Не удалось загрузить список данных");
        }
    }
}

// Использование
User admin = TestDataLoader.loadFromJson("/data/admin.json", User.class);
List<Product> products = TestDataLoader.loadListFromJson("/data/products.json", Product.class);
```

---

## Связь с тестированием

- **Selenium** — `List<WebElement>`, `WebDriverWait` с generic-ожиданиями
- **REST Assured** — `TypeRef<T>` для десериализации ответов в нужный тип
- **Page Object** — generic-базовый класс для цепочки вызовов (`BasePage<T extends BasePage<T>>`)
- **TestNG/JUnit** — `@DataProvider` возвращает `Object[][]`, но generic-утилиты помогают типизировать данные
- **Jackson/Gson** — `TypeReference<T>` для работы с generic-типами при десериализации
- **Assertion-библиотеки** — AssertJ: `assertThat(list).extracting(...)` работает с generics

---

## Типичные ошибки

1. **Raw types** — использование `List` вместо `List<String>` — теряется типобезопасность
```java
List rawList = new ArrayList(); // предупреждение компилятора, нет проверки типов
```

2. **Смешение generics и массивов** — массивы ковариантны, generics — инвариантны
```java
Object[] array = new String[10]; // Компилируется (ковариантность массивов)
array[0] = 42; // ArrayStoreException в рантайме!

// List<Object> list = new ArrayList<String>(); // ОШИБКА КОМПИЛЯЦИИ (инвариантность generics)
// Это ПРАВИЛЬНОЕ поведение — защищает от ошибок
```

3. **Забыть указать тип при создании** — diamond operator `<>` помогает
```java
// Java 7+: diamond operator выводит тип автоматически
List<Map<String, List<TestResult>>> data = new ArrayList<>(); // <> вместо повторения типов
```

4. **Попытка создать экземпляр типового параметра**
```java
// Нельзя: new T() — из-за type erasure
// Решение: передать Class<T> и использовать рефлексию или Supplier<T>
```

5. **Неправильное использование wildcard** — `List<?>` вместо `List<? extends T>` или `List<T>`

6. **Игнорирование предупреждений компилятора** — `unchecked` предупреждения — потенциальные баги

---

## Вопросы на интервью

### Уровень Junior

- Что такое generics и зачем они нужны?
- Что означает `<T>` в объявлении класса или метода?
- Чем `List<String>` отличается от просто `List` (raw type)?
- Что такое diamond operator `<>`?
- Приведите пример generic-класса или метода.

### Уровень Middle

- Что такое type erasure? Какие у него последствия?
- В чём разница между `<? extends T>` и `<? super T>`?
- Объясните принцип PECS. Приведите пример.
- Можно ли создать массив generic-типа? Почему?
- Чем отличается `List<Object>` от `List<?>`?
- Как работает `TypeReference` в Jackson и зачем он нужен?

### Уровень Senior

- Как реализовать generic-метод, который работает с несколькими ограничениями типов (intersection types)?
- Как обойти ограничения type erasure при десериализации JSON в generic-тип?
- В чём разница между ковариантностью массивов и инвариантностью generics? Почему это важно?
- Как generics взаимодействуют с наследованием? Приведите пример подводного камня.
- Как реализовать типобезопасный heterogeneous container (typesafe heterogeneous container pattern)?
- Что такое рекурсивные ограничения типов (recursive type bounds)? Пример: `<T extends Comparable<T>>`.

---

## Практические задания

1. **Generic Container:** Создайте `TestDataPair<F, S>` — пару из двух элементов разных типов (например, `TestDataPair<String, Integer>` для хранения имени параметра и его значения). Добавьте метод `swap()`, возвращающий `TestDataPair<S, F>`.

2. **Generic Utility:** Напишите generic-метод `findAll(List<T> list, Predicate<T> condition)`, возвращающий `List<T>` элементов, удовлетворяющих условию. Протестируйте с разными типами.

3. **Bounded Type:** Создайте метод `findMaxByProperty(List<T> list, Function<T, R> extractor)`, где `R extends Comparable<R>`. Метод должен находить элемент с максимальным значением указанного свойства.

4. **API Response Wrapper:** Реализуйте generic-класс `ApiResponse<T>` с полями `statusCode`, `data`, `errors`. Добавьте методы `isSuccess()`, `getDataOrThrow()`. Используйте в тестах API.

5. **PECS в действии:** Напишите метод `mergeResults`, который принимает `List<? extends TestResult>` (источник) и `List<? super TestResult>` (приёмник), фильтрует только `FAILED`-результаты и добавляет их в приёмник.

---

## Дополнительные ресурсы

- [Oracle — Generics Tutorial](https://docs.oracle.com/javase/tutorial/java/generics/)
- [Baeldung — Guide to Java Generics](https://www.baeldung.com/java-generics)
- [Effective Java — Joshua Bloch (глава 5: Generics)](https://www.oreilly.com/library/view/effective-java/9780134686097/)
- [Baeldung — Type Erasure in Java](https://www.baeldung.com/java-type-erasure)
- [Java Generics FAQ — Angelika Langer](http://www.angelikalanger.com/GenericsFAQ/JavaGenericsFAQ.html)
