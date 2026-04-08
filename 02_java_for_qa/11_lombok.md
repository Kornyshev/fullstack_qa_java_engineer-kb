# Lombok для тестового кода

## Обзор

Lombok — это библиотека для Java, которая автоматически генерирует boilerplate-код (конструкторы, getters, setters, `toString()`, `equals()`, `hashCode()`) на этапе компиляции с помощью аннотаций. Для QA-инженера Lombok — незаменимый инструмент: он позволяет создавать лаконичные POJO для API-тестов, удобно генерировать тестовые данные через `@Builder` и добавлять логирование через `@Slf4j`. Вместо 50 строк шаблонного кода вы пишете 5 строк с аннотациями — и тестовый код остаётся чистым и читаемым.

---

## Подключение Lombok

### Maven

```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.32</version>
    <scope>provided</scope> <!-- Нужен только при компиляции -->
</dependency>
```

### Gradle

```groovy
compileOnly 'org.projectlombok:lombok:1.18.32'
annotationProcessor 'org.projectlombok:lombok:1.18.32'

// Для тестового кода
testCompileOnly 'org.projectlombok:lombok:1.18.32'
testAnnotationProcessor 'org.projectlombok:lombok:1.18.32'
```

### Настройка IDE (IntelliJ IDEA)

1. **Установите плагин Lombok:** `Settings → Plugins → Marketplace → "Lombok" → Install`
2. **Включите annotation processing:** `Settings → Build, Execution, Deployment → Compiler → Annotation Processors → Enable annotation processing`
3. **Перезапустите IDE**

**Без этих шагов** IDE не будет видеть сгенерированные методы, код будет подсвечен красным, автодополнение не будет работать. Это самая частая проблема при первом использовании Lombok.

---

## Основные аннотации

### @Getter и @Setter

Генерируют getter и setter для полей класса.

```java
import lombok.Getter;
import lombok.Setter;

@Getter
@Setter
public class UserRequest {
    private String name;
    private String email;
    private int age;
}

// Эквивалент без Lombok — 20+ строк:
// public String getName() { return name; }
// public void setName(String name) { this.name = name; }
// public String getEmail() { return email; }
// ... и так для каждого поля
```

Можно применять к отдельным полям:

```java
public class UserResponse {
    @Getter
    private Long id;         // Только getter (id не должен меняться)

    @Getter @Setter
    private String name;     // И getter, и setter

    @Getter
    @Setter(AccessLevel.PROTECTED)
    private String internal; // Setter с ограниченным доступом
}
```

### @ToString

Генерирует метод `toString()` — полезно для логирования в тестах.

```java
import lombok.ToString;

@ToString
public class UserResponse {
    private Long id;
    private String name;
    private String email;
}

// Результат: UserResponse(id=1, name=Иван, email=ivan@test.com)

// Исключить поле из toString (например, пароль)
@ToString(exclude = "password")
public class UserRequest {
    private String name;
    private String password; // Не попадёт в toString
}

// Включить только определённые поля
@ToString(onlyExplicitlyIncluded = true)
public class UserResponse {
    @ToString.Include
    private Long id;
    @ToString.Include
    private String name;
    private String internalData; // Не попадёт в toString
}
```

### @EqualsAndHashCode

Генерирует `equals()` и `hashCode()` — важно для сравнения объектов в assertions.

```java
import lombok.EqualsAndHashCode;

@EqualsAndHashCode
public class UserResponse {
    private Long id;
    private String name;
    private String email;
}

// Теперь можно сравнивать объекты
UserResponse expected = new UserResponse(1L, "Иван", "ivan@test.com");
UserResponse actual = objectMapper.readValue(json, UserResponse.class);
assertThat(actual).isEqualTo(expected); // Сравнение по всем полям

// Исключить поле из сравнения (например, id генерируется сервером)
@EqualsAndHashCode(exclude = "id")
public class UserResponse {
    private Long id;       // Не участвует в equals/hashCode
    private String name;   // Участвует
    private String email;  // Участвует
}
```

### @Data

Комбинированная аннотация — заменяет `@Getter`, `@Setter`, `@ToString`, `@EqualsAndHashCode` и `@RequiredArgsConstructor`. Самая популярная аннотация для POJO в тестах.

```java
import lombok.Data;

@Data
public class CreateUserRequest {
    private String name;
    private String email;
    private int age;
    private String role;
}

// Эквивалент: @Getter + @Setter + @ToString + @EqualsAndHashCode
// + @RequiredArgsConstructor (конструктор для final-полей)
// Это примерно 80 строк сгенерированного кода!
```

Пример использования в тесте:

```java
@Test
void shouldCreateUser() {
    // Создаём объект и заполняем через setters
    var request = new CreateUserRequest();
    request.setName("Иван");
    request.setEmail("ivan@test.com");
    request.setAge(30);
    request.setRole("admin");

    UserResponse response = given()
        .body(request) // toString() в логах покажет все поля
    .when()
        .post("/api/users")
    .then()
        .statusCode(201)
        .extract()
        .as(UserResponse.class);

    assertThat(response.getName()).isEqualTo(request.getName());
}
```

### @NoArgsConstructor и @AllArgsConstructor

```java
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;

@Data
@NoArgsConstructor          // Конструктор без аргументов (нужен для Jackson)
@AllArgsConstructor         // Конструктор со всеми аргументами (удобен для тестов)
public class UserRequest {
    private String name;
    private String email;
    private int age;
}

// Теперь доступны оба варианта:
var user1 = new UserRequest();                         // No-args
var user2 = new UserRequest("Иван", "ivan@test.com", 30); // All-args
```

### @RequiredArgsConstructor

Генерирует конструктор только для `final`-полей и полей с `@NonNull`.

```java
import lombok.RequiredArgsConstructor;

@RequiredArgsConstructor
public class ApiClient {
    private final String baseUrl;      // Попадёт в конструктор
    private final RestTemplate client;  // Попадёт в конструктор
    private int timeout = 30;           // НЕ попадёт — не final
}

// Сгенерированный конструктор:
// public ApiClient(String baseUrl, RestTemplate client) { ... }
```

---

## @Builder — главная аннотация для тестовых данных

`@Builder` реализует паттерн Builder — самый удобный способ создания тестовых объектов.

### Базовое использование

```java
import lombok.Builder;
import lombok.Data;

@Data
@Builder
public class CreateUserRequest {
    private String name;
    private String email;
    private int age;
    private String role;
    private String department;
    private boolean active;
}

// Создание объекта через Builder — чистый, читаемый код
var request = CreateUserRequest.builder()
    .name("Иван Петров")
    .email("ivan@test.com")
    .age(30)
    .role("admin")
    .department("QA")
    .active(true)
    .build();
```

### @Builder.Default — значения по умолчанию

```java
@Data
@Builder
public class CreateUserRequest {
    private String name;
    private String email;

    @Builder.Default
    private int age = 25;              // По умолчанию 25

    @Builder.Default
    private String role = "user";      // По умолчанию "user"

    @Builder.Default
    private boolean active = true;     // По умолчанию активен

    @Builder.Default
    private String department = "QA";  // По умолчанию QA
}

// Теперь можно создавать объекты, указывая только нужные поля
var minimalRequest = CreateUserRequest.builder()
    .name("Иван")
    .email("ivan@test.com")
    .build();
// age=25, role="user", active=true, department="QA" — подставятся автоматически
```

### Builder для тестовых сценариев

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class OrderRequest {
    private String customerId;
    private String productId;

    @Builder.Default
    private int quantity = 1;

    @Builder.Default
    private String currency = "RUB";

    private String promoCode;
    private String deliveryAddress;
}

class OrderApiTest {

    @Test
    @DisplayName("Создание заказа — минимальный набор полей")
    void shouldCreateOrderWithMinimalFields() {
        var request = OrderRequest.builder()
            .customerId("CUST-001")
            .productId("PROD-100")
            .build();
        // quantity=1, currency="RUB" — подставятся из @Builder.Default

        given().body(request)
            .when().post("/api/orders")
            .then().statusCode(201);
    }

    @Test
    @DisplayName("Создание заказа — полный набор полей")
    void shouldCreateOrderWithAllFields() {
        var request = OrderRequest.builder()
            .customerId("CUST-001")
            .productId("PROD-100")
            .quantity(5)
            .currency("USD")
            .promoCode("SALE20")
            .deliveryAddress("Москва, ул. Тверская, д. 1")
            .build();

        given().body(request)
            .when().post("/api/orders")
            .then().statusCode(201);
    }

    @Test
    @DisplayName("Валидация — заказ с нулевым количеством")
    void shouldRejectOrderWithZeroQuantity() {
        var request = OrderRequest.builder()
            .customerId("CUST-001")
            .productId("PROD-100")
            .quantity(0) // Невалидное значение
            .build();

        given().body(request)
            .when().post("/api/orders")
            .then().statusCode(400);
    }
}
```

### @Builder + Jackson — идеальная пара для API-тестов

```java
import com.fasterxml.jackson.annotation.JsonInclude;
import com.fasterxml.jackson.annotation.JsonProperty;
import com.fasterxml.jackson.databind.PropertyNamingStrategies;
import com.fasterxml.jackson.databind.annotation.JsonNaming;
import lombok.*;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
@JsonNaming(PropertyNamingStrategies.SnakeCaseStrategy.class)
public class CreateUserRequest {
    private String firstName;
    private String lastName;
    private String email;

    @Builder.Default
    private String role = "user";

    @Builder.Default
    private boolean emailVerified = false;
}

// В тесте:
var request = CreateUserRequest.builder()
    .firstName("Иван")
    .lastName("Петров")
    .email("ivan@test.com")
    .build();

// JSON: {"first_name":"Иван","last_name":"Петров",
//        "email":"ivan@test.com","role":"user","email_verified":false}
```

**Важно:** при использовании `@Builder` вместе с Jackson необходимо также добавить `@NoArgsConstructor` и `@AllArgsConstructor`, иначе Jackson не сможет десериализовать JSON обратно в объект.

---

## @Slf4j — логирование в тестах

`@Slf4j` автоматически создаёт логгер для класса. Незаменим для отладки тестов.

```java
import lombok.extern.slf4j.Slf4j;

@Slf4j
public class UserApiTest extends BaseApiTest {

    @Test
    void shouldCreateUser() {
        var request = CreateUserRequest.builder()
            .name("Иван")
            .email("ivan@test.com")
            .build();

        log.info("Отправляем запрос на создание пользователя: {}", request);

        var response = given()
            .body(request)
        .when()
            .post("/api/users")
        .then()
            .statusCode(201)
            .extract()
            .as(UserResponse.class);

        log.info("Получен ответ: {}", response);
        log.debug("ID созданного пользователя: {}", response.getId());

        assertThat(response.getName()).isEqualTo("Иван");
    }
}

// Эквивалент без Lombok:
// private static final Logger log = LoggerFactory.getLogger(UserApiTest.class);
```

### Уровни логирования

| Уровень | Когда использовать | Пример |
|---------|-------------------|--------|
| `log.error()` | Критические ошибки | `log.error("API вернул 500: {}", responseBody)` |
| `log.warn()` | Предупреждения | `log.warn("Тест занял {} мс (лимит: 5000)", duration)` |
| `log.info()` | Основная информация | `log.info("Создан пользователь: {}", userId)` |
| `log.debug()` | Детальная информация | `log.debug("Тело запроса: {}", requestBody)` |
| `log.trace()` | Максимальная детализация | `log.trace("Куки: {}", cookies)` |

### @Slf4j в базовом тестовом классе

```java
@Slf4j
public abstract class BaseApiTest {

    @BeforeEach
    void logTestStart(TestInfo testInfo) {
        log.info("═══ Начало теста: {} ═══", testInfo.getDisplayName());
    }

    @AfterEach
    void logTestEnd(TestInfo testInfo) {
        log.info("═══ Конец теста: {} ═══", testInfo.getDisplayName());
    }

    // Утилитный метод для логирования запросов
    protected void logRequest(String method, String url, Object body) {
        log.info("→ {} {}", method, url);
        if (body != null) {
            log.debug("→ Тело запроса: {}", body);
        }
    }

    // Утилитный метод для логирования ответов
    protected void logResponse(int statusCode, String body) {
        log.info("← Статус: {}", statusCode);
        log.debug("← Тело ответа: {}", body);
    }
}
```

---

## Другие полезные аннотации

### @Value — иммутабельный объект

Аналог `@Data`, но все поля `private final`, класс `final`, нет setters. Подходит для объектов, которые не должны меняться после создания.

```java
import lombok.Value;

@Value
public class ApiConfig {
    String baseUrl;
    String apiKey;
    int timeout;
}

// Использование
var config = new ApiConfig("https://api.test.com", "key123", 30);
// config.setBaseUrl(...) — НЕ СУЩЕСТВУЕТ, объект иммутабельный
```

### @With — создание копии с изменённым полем

```java
import lombok.With;
import lombok.Value;

@Value
public class UserRequest {
    @With String name;
    @With String email;
    int age;
}

var baseUser = new UserRequest("Иван", "ivan@test.com", 30);
var modified = baseUser.withName("Пётр").withEmail("petr@test.com");
// Создана копия с изменёнными полями, оригинал не изменён
```

### @SneakyThrows — для checked exceptions

```java
import lombok.SneakyThrows;

public class TestUtils {

    @SneakyThrows // Не нужно объявлять throws IOException
    public static String readTestData(String path) {
        return Files.readString(Path.of(path));
    }

    @SneakyThrows
    public static <T> T fromJson(String json, Class<T> clazz) {
        return new ObjectMapper().readValue(json, clazz);
    }
}
```

**Осторожно:** `@SneakyThrows` скрывает checked exceptions. Используйте осознанно — только в утилитном тестовом коде.

### @Cleanup — автоматическое закрытие ресурсов

```java
import lombok.Cleanup;

public void processTestData() {
    @Cleanup InputStream in = new FileInputStream("testdata.json");
    // in.close() вызовется автоматически
}
```

В большинстве случаев лучше использовать `try-with-resources`, но `@Cleanup` удобен для быстрого прототипирования.

---

## Когда НЕ использовать Lombok

| Ситуация | Причина | Альтернатива |
|----------|---------|-------------|
| Production-код приложения (не тесты) | Команда разработки может не одобрить | Java records (Java 16+), обычные классы |
| Сложная логика в getters/setters | Lombok не поддерживает кастомную логику | Написать вручную |
| Наследование с `@EqualsAndHashCode` | Проблемы с `equals()` в иерархии классов | `@EqualsAndHashCode(callSuper = true)` или вручную |
| `@Data` на entity-классах | `toString()` может вызвать lazy loading | `@Getter` + `@Setter` отдельно |
| Проект без поддержки annotation processing | Не скомпилируется | Обычные Java-классы, IDE-генерация |
| Очень простые классы (1-2 поля) | Lombok избыточен | Java records |

### Java Records vs Lombok

Начиная с Java 16 доступны Records — встроенный механизм для иммутабельных данных:

```java
// Java Record — без внешних зависимостей
public record UserResponse(Long id, String name, String email) {}

// Lombok @Value — аналогичный результат, но через библиотеку
@Value
public class UserResponse {
    Long id;
    String name;
    String email;
}
```

| Критерий | Java Record | Lombok @Value |
|----------|-------------|---------------|
| Внешние зависимости | Нет | Да (lombok) |
| Мутабельность | Только immutable | Только immutable |
| Builder | Нет (нужен вручную) | `@Builder` |
| Default-значения | Нет | `@Builder.Default` |
| Наследование | Нет (records are final) | Нет (Value classes are final) |
| Совместимость с Jackson | Да (с Java 16+) | Да |

**Рекомендация для QA:** используйте Records для простых immutable-объектов, Lombok `@Data` + `@Builder` для сложных POJO с Builder-паттерном.

---

## Связь с тестированием

1. **POJO для API-тестов** — `@Data` + `@Builder` + `@NoArgsConstructor` + `@AllArgsConstructor` — стандартный набор аннотаций для request/response классов. Экономит десятки строк boilerplate-кода.

2. **Создание тестовых данных** — `@Builder` с `@Builder.Default` позволяет создавать объекты, указывая только отличающиеся от дефолтных значения. Это ускоряет написание тестов и делает их читаемее.

3. **Логирование тестов** — `@Slf4j` — стандартный способ добавить логирование в тестовый класс. Без него приходится вручную создавать Logger в каждом классе.

4. **Page Object Model** — `@Getter` + `@RequiredArgsConstructor` удобны для создания page-объектов с WebDriver:

```java
@Getter
@RequiredArgsConstructor
public class LoginPage {
    private final WebDriver driver;
    // driver передаётся через конструктор автоматически
}
```

5. **Config-классы** — `@Value` или `@Data` для конфигурационных объектов тестового фреймворка, хранящих URL, таймауты, credentials.

---

## Типичные ошибки

1. **Не установлен плагин Lombok в IDE.** Код компилируется через Maven/Gradle, но IDE показывает ошибки. Решение: установить плагин и включить annotation processing.

2. **Забывают `@NoArgsConstructor` при использовании `@Builder`.** Jackson требует конструктор без аргументов для десериализации. Без `@NoArgsConstructor` тесты, использующие Jackson для парсинга ответов, упадут.

3. **Используют `@Data` на классах с наследованием.** `@EqualsAndHashCode`, включённый в `@Data`, не учитывает поля родительского класса. Решение: добавить `@EqualsAndHashCode(callSuper = true)`.

4. **`@Builder.Default` не работает как ожидается.** Если использовать `@Builder` без `@Builder.Default`, значения по умолчанию, указанные при объявлении полей (`private int age = 25`), будут проигнорированы Builder-ом. Нужно явно пометить `@Builder.Default`.

---

## Вопросы на интервью

- 🟢 **Q:** Что такое Lombok и зачем он нужен в тестовом проекте?
- **A:** Lombok — библиотека, генерирующая boilerplate-код (getters, setters, конструкторы, toString, equals/hashCode) во время компиляции через аннотации. В тестах сокращает объём кода для POJO (request/response), упрощает создание тестовых данных через `@Builder`, добавляет логирование через `@Slf4j`.

- 🟢 **Q:** Что делает аннотация `@Data`?
- **A:** `@Data` = `@Getter` + `@Setter` + `@ToString` + `@EqualsAndHashCode` + `@RequiredArgsConstructor`. Генерирует все стандартные методы POJO за одну аннотацию.

- 🟢 **Q:** Как использовать `@Builder` для создания тестовых данных?
- **A:** Ставим `@Builder` на класс, создаём объекты через `ClassName.builder().field1(value1).field2(value2).build()`. Для значений по умолчанию используем `@Builder.Default`. Это паттерн Builder, реализованный автоматически.

- 🟡 **Q:** Какие аннотации Lombok нужны для POJO, используемого в REST Assured с Jackson?
- **A:** Минимальный набор: `@Data` + `@NoArgsConstructor` + `@AllArgsConstructor`. Для удобства: `@Builder`. `@NoArgsConstructor` обязателен для десериализации Jackson.

- 🟡 **Q:** Что такое `@Slf4j` и чем это лучше `System.out.println`?
- **A:** `@Slf4j` создаёт логгер SLF4J. В отличие от `System.out.println`: поддерживает уровни логирования (debug/info/warn/error), работает с logging-фреймворками (Logback, Log4j), выводит timestamp и имя класса, можно настроить вывод в файл.

- 🟡 **Q:** Чем `@Value` отличается от `@Data`?
- **A:** `@Value` создаёт immutable-класс: все поля `private final`, нет setters, класс `final`. `@Data` — mutable: есть setters, поля не final. Для конфигов и неизменяемых объектов — `@Value`, для POJO с Jackson — `@Data`.

- 🟡 **Q:** Нужно ли устанавливать Lombok в IDE? Почему?
- **A:** Да. Lombok работает на этапе компиляции через annotation processing. IDE не видит сгенерированные методы без плагина и показывает ошибки. В IntelliJ: установить плагин + включить annotation processing.

- 🔴 **Q:** Когда НЕ стоит использовать Lombok?
- **A:** Когда команда не одобряет зависимость, при сложной логике в getters/setters, при проблемах с наследованием и `equals()`, в production-коде, если команда предпочитает Java Records. Также `@SneakyThrows` может скрыть важные исключения.

- 🔴 **Q:** Сравните Java Records и Lombok. Когда что использовать?
- **A:** Records — встроенный механизм Java 16+, immutable, без зависимостей, но без Builder и default-значений. Lombok — внешняя библиотека, но гибче: `@Builder`, `@Builder.Default`, mutable и immutable варианты. Для простых immutable POJO — Records, для сложных тестовых данных с Builder — Lombok.

---

## Практические задания

### Задание 1 (базовое): POJO с Lombok для API-теста

Создайте `CreateBookRequest` и `BookResponse` с использованием Lombok:
- `@Data`, `@Builder`, `@NoArgsConstructor`, `@AllArgsConstructor`
- `@Builder.Default` для значений по умолчанию (`genre = "fiction"`, `available = true`)
- Напишите тест, создающий книгу через Builder и проверяющий ответ

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class CreateBookRequest {
    private String title;
    private String author;

    @Builder.Default
    private String genre = "fiction";

    private Integer year;

    @Builder.Default
    private boolean available = true;
}

// Использование в тесте:
var book = CreateBookRequest.builder()
    .title("Мастер и Маргарита")
    .author("Михаил Булгаков")
    .year(1967)
    .build();
// genre="fiction", available=true — из @Builder.Default
```

### Задание 2 (среднее): Фабрика тестовых данных

Создайте класс `TestDataFactory`, использующий `@Builder` для генерации типичных тестовых объектов:
- Метод `validUser()` — пользователь со всеми валидными полями
- Метод `userWithoutEmail()` — пользователь без email (для негативного теста)
- Метод `adminUser()` — пользователь с ролью admin
- Используйте `@Slf4j` для логирования создания каждого объекта

```java
@Slf4j
public class TestDataFactory {

    public static CreateUserRequest validUser() {
        var user = CreateUserRequest.builder()
            .name("Тестовый Пользователь")
            .email("test_" + System.currentTimeMillis() + "@test.com")
            .age(25)
            .role("user")
            .build();
        log.info("Создан валидный пользователь: {}", user);
        return user;
    }

    public static CreateUserRequest userWithoutEmail() {
        var user = CreateUserRequest.builder()
            .name("Пользователь без email")
            .age(30)
            .role("user")
            .build();
        log.info("Создан пользователь без email: {}", user);
        return user;
    }

    public static CreateUserRequest adminUser() {
        var user = CreateUserRequest.builder()
            .name("Администратор")
            .email("admin@test.com")
            .age(35)
            .role("admin")
            .build();
        log.info("Создан администратор: {}", user);
        return user;
    }
}
```

### Задание 3 (продвинутое): Полная модель с наследованием

Создайте иерархию POJO для интернет-магазина:
- Базовый класс `BaseRequest` с полями `requestId`, `timestamp` (с `@Builder.Default`)
- `CreateOrderRequest extends BaseRequest` с полями для заказа
- `OrderResponse` с дополнительными полями от сервера
- Используйте `@SuperBuilder` (Lombok) для Builder в иерархии наследования
- Добавьте Jackson-аннотации для snake_case
- Напишите 3 теста с разными сценариями

---

## Дополнительные ресурсы

- [Project Lombok — Official](https://projectlombok.org/) — официальный сайт с документацией
- [Baeldung — Introduction to Project Lombok](https://www.baeldung.com/intro-to-project-lombok) — подробный гайд
- [Baeldung — Lombok Builder](https://www.baeldung.com/lombok-builder) — всё о `@Builder`
- [Baeldung — Lombok @Slf4j](https://www.baeldung.com/lombok-slf4j) — логирование с Lombok
- [IntelliJ IDEA — Lombok Plugin](https://plugins.jetbrains.com/plugin/6317-lombok) — плагин для IDE
- [Baeldung — Java Records vs Lombok](https://www.baeldung.com/java-record-vs-lombok) — сравнение Records и Lombok
