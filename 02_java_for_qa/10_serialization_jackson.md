# Сериализация и Jackson

## Обзор

Сериализация — это преобразование Java-объекта в формат, пригодный для передачи или хранения (JSON, XML, байты), а десериализация — обратный процесс. Для QA-инженера, работающего с API-тестами, Jackson — это основной инструмент: он автоматически преобразует POJO-объекты в JSON (для отправки запросов) и JSON-ответы обратно в Java-объекты (для проверки данных). Без уверенного владения Jackson невозможно писать чистые, поддерживаемые API-тесты. REST Assured использует Jackson «под капотом» при вызовах `.body(object)` и `.as(Class)`.

---

## Что такое сериализация и десериализация

```
Сериализация (Serialization):
Java Object  ──→  JSON / XML / bytes

Десериализация (Deserialization):
JSON / XML / bytes  ──→  Java Object
```

### Зачем QA-инженеру

| Операция | Контекст в тестировании |
|----------|------------------------|
| Сериализация | Формирование тела запроса (`POST`, `PUT`, `PATCH`) из Java-объекта |
| Десериализация | Парсинг тела ответа в Java-объект для assertions |
| Оба направления | Работа с тестовыми данными из JSON-файлов, конфигурациями |

### Пример из жизни QA

Без Jackson (строковый JSON — хрупко и неудобно):
```java
String body = "{\"name\": \"Иван\", \"email\": \"ivan@test.com\", \"age\": 25}";
```

С Jackson (типобезопасный POJO):
```java
UserRequest user = new UserRequest("Иван", "ivan@test.com", 25);
String json = objectMapper.writeValueAsString(user);
// Результат: {"name":"Иван","email":"ivan@test.com","age":25}
```

---

## Jackson ObjectMapper

`ObjectMapper` — главный класс библиотеки Jackson. Через него выполняются все операции сериализации и десериализации.

### Подключение зависимости

```xml
<!-- Maven -->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.17.0</version>
</dependency>
```

```groovy
// Gradle
implementation 'com.fasterxml.jackson.core:jackson-databind:2.17.0'
```

### Базовые операции

```java
import com.fasterxml.jackson.databind.ObjectMapper;

// Создаём ObjectMapper — обычно один экземпляр на весь проект
ObjectMapper objectMapper = new ObjectMapper();

// ============ СЕРИАЛИЗАЦИЯ ============

// Объект → JSON строка
UserRequest user = new UserRequest("Иван", "ivan@test.com", 25);
String json = objectMapper.writeValueAsString(user);
// Результат: {"name":"Иван","email":"ivan@test.com","age":25}

// Объект → JSON файл
objectMapper.writeValue(new File("user.json"), user);

// Объект → byte[]
byte[] bytes = objectMapper.writeValueAsBytes(user);

// ============ ДЕСЕРИАЛИЗАЦИЯ ============

// JSON строка → объект
String responseJson = """
    {"name": "Иван", "email": "ivan@test.com", "age": 25}
    """;
UserResponse user = objectMapper.readValue(responseJson, UserResponse.class);

// JSON файл → объект
UserResponse userFromFile = objectMapper.readValue(
    new File("testdata/user.json"), UserResponse.class
);

// JSON строка → List объектов
List<UserResponse> users = objectMapper.readValue(
    jsonArray,
    new TypeReference<List<UserResponse>>() {}
);

// JSON строка → Map
Map<String, Object> map = objectMapper.readValue(
    json,
    new TypeReference<Map<String, Object>>() {}
);
```

### Настройка ObjectMapper для тестов

```java
public class JacksonConfig {

    // Настроенный ObjectMapper для всего тестового проекта
    public static final ObjectMapper MAPPER = new ObjectMapper()
        // Не падать, если в JSON есть поля, которых нет в POJO
        .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
        // Красивый вывод JSON (удобно для логирования)
        .enable(SerializationFeature.INDENT_OUTPUT)
        // Поддержка Java 8 дат (LocalDate, LocalDateTime)
        .registerModule(new JavaTimeModule())
        // Даты как строки, а не timestamps
        .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
}
```

**Важно:** `FAIL_ON_UNKNOWN_PROPERTIES = false` — критическая настройка для API-тестов. API может вернуть дополнительные поля, которых нет в вашем POJO. Без этой настройки тест упадёт с `UnrecognizedPropertyException`.

---

## POJO для API-тестирования

POJO (Plain Old Java Object) — это простой Java-класс, описывающий структуру данных запроса или ответа.

### Пример: создание Request и Response POJO

```java
// POJO для тела запроса на создание пользователя
public class CreateUserRequest {
    private String name;
    private String email;
    private int age;
    private String role;

    // Конструктор по умолчанию — обязателен для Jackson
    public CreateUserRequest() {}

    // Конструктор со всеми полями — удобен для создания в тестах
    public CreateUserRequest(String name, String email, int age, String role) {
        this.name = name;
        this.email = email;
        this.age = age;
        this.role = role;
    }

    // Getters и Setters — Jackson использует их для сериализации/десериализации
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }

    public int getAge() { return age; }
    public void setAge(int age) { this.age = age; }

    public String getRole() { return role; }
    public void setRole(String role) { this.role = role; }
}
```

```java
// POJO для тела ответа — может содержать дополнительные поля
public class UserResponse {
    private Long id;
    private String name;
    private String email;
    private int age;
    private String role;
    private String createdAt;

    // Конструктор по умолчанию
    public UserResponse() {}

    // Getters и Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }

    public int getAge() { return age; }
    public void setAge(int age) { this.age = age; }

    public String getRole() { return role; }
    public void setRole(String role) { this.role = role; }

    public String getCreatedAt() { return createdAt; }
    public void setCreatedAt(String createdAt) { this.createdAt = createdAt; }
}
```

### Использование POJO в REST Assured тестах

```java
@Test
@DisplayName("Создание пользователя через API возвращает 201 и корректные данные")
void shouldCreateUser() {
    // Формируем тело запроса через POJO
    CreateUserRequest request = new CreateUserRequest(
        "Иван Петров", "ivan@test.com", 30, "admin"
    );

    // REST Assured автоматически сериализует POJO в JSON через Jackson
    UserResponse response = given()
        .contentType(ContentType.JSON)
        .body(request)  // Сериализация: POJO → JSON
    .when()
        .post("/api/users")
    .then()
        .statusCode(201)
        .extract()
        .as(UserResponse.class);  // Десериализация: JSON → POJO

    // Проверяем через Java-объект — типобезопасно
    assertThat(response.getName()).isEqualTo("Иван Петров");
    assertThat(response.getEmail()).isEqualTo("ivan@test.com");
    assertThat(response.getId()).isNotNull();
    assertThat(response.getCreatedAt()).isNotNull();
}
```

---

## Аннотации Jackson

Аннотации Jackson позволяют тонко настраивать процесс сериализации и десериализации.

### @JsonProperty

Связывает Java-поле с именем в JSON, когда они различаются (snake_case в API, camelCase в Java).

```java
public class OrderRequest {

    @JsonProperty("order_id")         // В JSON будет "order_id"
    private String orderId;

    @JsonProperty("customer_name")    // В JSON будет "customer_name"
    private String customerName;

    @JsonProperty("total_amount")
    private double totalAmount;

    @JsonProperty("is_paid")
    private boolean isPaid;

    // Getters / Setters
}
```

**Альтернатива для всего класса** — стратегия именования:
```java
@JsonNaming(PropertyNamingStrategies.SnakeCaseStrategy.class)
public class OrderRequest {
    private String orderId;       // Автоматически станет "order_id"
    private String customerName;  // Автоматически станет "customer_name"
    private double totalAmount;   // Автоматически станет "total_amount"
}
```

### @JsonIgnore

Исключает поле из сериализации и десериализации.

```java
public class UserRequest {
    private String name;
    private String email;

    @JsonIgnore
    private String internalTestNote; // Не попадёт в JSON — используется только в тестах
}
```

### @JsonIgnoreProperties

Исключает несколько полей на уровне класса или игнорирует неизвестные поля.

```java
// Игнорируем конкретные поля при сериализации
@JsonIgnoreProperties({"createdAt", "updatedAt"})
public class UserResponse {
    private Long id;
    private String name;
    private String createdAt;  // Не будет сериализовано
    private String updatedAt;  // Не будет сериализовано
}

// Игнорируем любые неизвестные поля при десериализации
@JsonIgnoreProperties(ignoreUnknown = true)
public class UserResponse {
    private Long id;
    private String name;
    // Если в JSON есть поля, которых здесь нет — ошибки не будет
}
```

### @JsonInclude

Управляет включением null-значений и пустых коллекций.

```java
// Не включать null-поля в JSON
@JsonInclude(JsonInclude.Include.NON_NULL)
public class UpdateUserRequest {
    private String name;   // Если null — не попадёт в JSON
    private String email;  // Если null — не попадёт в JSON
    private Integer age;   // Если null — не попадёт в JSON
}
```

Это особенно полезно для PATCH-запросов, где нужно отправить только изменённые поля:

```java
@Test
@DisplayName("Частичное обновление — меняем только email")
void shouldPartiallyUpdateUser() {
    UpdateUserRequest request = new UpdateUserRequest();
    request.setEmail("new_email@test.com");
    // name и age — null, не попадут в JSON

    given()
        .contentType(ContentType.JSON)
        .body(request)
        // Тело запроса: {"email": "new_email@test.com"}
    .when()
        .patch("/api/users/1")
    .then()
        .statusCode(200);
}
```

### @JsonFormat

Задаёт формат сериализации, особенно полезен для дат.

```java
import com.fasterxml.jackson.annotation.JsonFormat;
import java.time.LocalDateTime;
import java.time.LocalDate;

public class EventResponse {

    @JsonFormat(pattern = "yyyy-MM-dd")
    private LocalDate eventDate;

    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss", timezone = "Europe/Moscow")
    private LocalDateTime createdAt;

    @JsonFormat(shape = JsonFormat.Shape.STRING) // Число как строка
    private Long eventId;
}
```

### Сводная таблица аннотаций

| Аннотация | Назначение | Типичный контекст в тестах |
|-----------|-----------|---------------------------|
| `@JsonProperty` | Маппинг имени поля в JSON | snake_case API ↔ camelCase Java |
| `@JsonIgnore` | Исключить поле | Внутренние тестовые поля |
| `@JsonIgnoreProperties` | Исключить несколько полей / неизвестные | Частичный POJO для ответа |
| `@JsonInclude` | Включать только непустые поля | PATCH-запросы |
| `@JsonFormat` | Формат даты/числа | Работа с датами в API |
| `@JsonNaming` | Стратегия именования для класса | Весь POJO в snake_case |
| `@JsonCreator` | Пометить конструктор для десериализации | Immutable POJO |
| `@JsonAlias` | Альтернативные имена для десериализации | API с нестабильными именами |

---

## Работа с вложенными объектами и коллекциями

### Вложенные объекты

```java
// JSON ответа:
// {
//   "id": 1,
//   "name": "Иван",
//   "address": {
//     "city": "Москва",
//     "street": "Ленина",
//     "zip": "101000"
//   }
// }

public class Address {
    private String city;
    private String street;
    private String zip;
    // Getters / Setters / Конструктор по умолчанию
}

public class UserWithAddress {
    private Long id;
    private String name;
    private Address address; // Вложенный объект — Jackson десериализует автоматически
    // Getters / Setters / Конструктор по умолчанию
}

// Десериализация
UserWithAddress user = objectMapper.readValue(json, UserWithAddress.class);
assertThat(user.getAddress().getCity()).isEqualTo("Москва");
```

### Списки и массивы

```java
// JSON: [{"id":1,"name":"Иван"}, {"id":2,"name":"Пётр"}]

// Способ 1: TypeReference (рекомендуемый)
List<UserResponse> users = objectMapper.readValue(
    json,
    new TypeReference<List<UserResponse>>() {}
);

// Способ 2: Через массив
UserResponse[] usersArray = objectMapper.readValue(json, UserResponse[].class);
List<UserResponse> users = Arrays.asList(usersArray);
```

### Сложная вложенная структура

```java
// JSON:
// {
//   "total": 2,
//   "page": 1,
//   "data": [
//     {"id": 1, "name": "Иван", "orders": [{"orderId": "A1"}, {"orderId": "A2"}]},
//     {"id": 2, "name": "Пётр", "orders": []}
//   ]
// }

// Обёртка для пагинированного ответа
public class PaginatedResponse<T> {
    private int total;
    private int page;
    private List<T> data;
    // Getters / Setters
}

public class Order {
    private String orderId;
    // Getters / Setters
}

public class UserWithOrders {
    private Long id;
    private String name;
    private List<Order> orders; // Список вложенных объектов
    // Getters / Setters
}

// Десериализация с TypeReference
PaginatedResponse<UserWithOrders> response = objectMapper.readValue(
    json,
    new TypeReference<PaginatedResponse<UserWithOrders>>() {}
);

assertThat(response.getTotal()).isEqualTo(2);
assertThat(response.getData().get(0).getOrders()).hasSize(2);
```

---

## Кастомные десериализаторы

Иногда API возвращает данные в нестандартном формате, и стандартной десериализации недостаточно.

```java
// API возвращает статус как число: {"status": 1} (1=active, 0=inactive)
// В Java хотим enum
public enum UserStatus {
    ACTIVE, INACTIVE;
}

// Кастомный десериализатор
public class UserStatusDeserializer extends JsonDeserializer<UserStatus> {
    @Override
    public UserStatus deserialize(JsonParser p, DeserializationContext ctx)
            throws IOException {
        int value = p.getIntValue();
        return value == 1 ? UserStatus.ACTIVE : UserStatus.INACTIVE;
    }
}

// Использование
public class UserResponse {
    private String name;

    @JsonDeserialize(using = UserStatusDeserializer.class)
    private UserStatus status;

    // Getters / Setters
}
```

---

## Сравнение Jackson и Gson

| Критерий | Jackson | Gson |
|----------|---------|------|
| Производительность | Быстрее (особенно на больших объёмах) | Медленнее |
| Аннотации | Богатый набор (`@JsonProperty`, `@JsonFormat`, ...) | Минимальный (`@SerializedName`, `@Expose`) |
| Поддержка Java 8+ | Через `jackson-datatype-jsr310` | Требует ручных адаптеров |
| REST Assured | Используется по умолчанию | Можно настроить |
| Spring Boot | Используется по умолчанию | Нужна дополнительная настройка |
| Гибкость | Очень гибкий (миксины, модули, фабрики) | Проще, но менее гибкий |
| Размер библиотеки | Больше (3 модуля: core, annotations, databind) | Один JAR |
| Кастомная (де)сериализация | `JsonSerializer` / `JsonDeserializer` | `TypeAdapter` / `JsonSerializer` |

```java
// Пример: одно и то же с Jackson и Gson

// ===== Jackson =====
ObjectMapper mapper = new ObjectMapper();
String json = mapper.writeValueAsString(user);
UserResponse parsed = mapper.readValue(json, UserResponse.class);

// ===== Gson =====
Gson gson = new Gson();
String json = gson.toJson(user);
UserResponse parsed = gson.fromJson(json, UserResponse.class);
```

**Рекомендация для QA:** используйте Jackson — он стандарт де-факто в Java-экосистеме, REST Assured и Spring Boot работают с ним по умолчанию.

---

## Практический пример: полный API-тест с POJO

```java
// Полный пример API-теста с использованием Jackson и REST Assured

@JsonInclude(JsonInclude.Include.NON_NULL)
@JsonNaming(PropertyNamingStrategies.SnakeCaseStrategy.class)
class CreateBookRequest {
    private String title;
    private String author;
    private Integer publishYear;
    private String isbn;

    // Конструктор, Getters, Setters
    public CreateBookRequest() {}

    public CreateBookRequest(String title, String author, Integer publishYear, String isbn) {
        this.title = title;
        this.author = author;
        this.publishYear = publishYear;
        this.isbn = isbn;
    }

    // Getters / Setters ...
    public String getTitle() { return title; }
    public void setTitle(String title) { this.title = title; }
    public String getAuthor() { return author; }
    public void setAuthor(String author) { this.author = author; }
    public Integer getPublishYear() { return publishYear; }
    public void setPublishYear(Integer publishYear) { this.publishYear = publishYear; }
    public String getIsbn() { return isbn; }
    public void setIsbn(String isbn) { this.isbn = isbn; }
}

@JsonIgnoreProperties(ignoreUnknown = true)
@JsonNaming(PropertyNamingStrategies.SnakeCaseStrategy.class)
class BookResponse {
    private Long id;
    private String title;
    private String author;
    private Integer publishYear;
    private String isbn;

    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss")
    private LocalDateTime createdAt;

    // Getters / Setters ...
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getTitle() { return title; }
    public void setTitle(String title) { this.title = title; }
    public String getAuthor() { return author; }
    public void setAuthor(String author) { this.author = author; }
    public Integer getPublishYear() { return publishYear; }
    public void setPublishYear(Integer publishYear) { this.publishYear = publishYear; }
    public String getIsbn() { return isbn; }
    public void setIsbn(String isbn) { this.isbn = isbn; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }
}

class BookApiTest extends BaseApiTest {

    @Test
    @DisplayName("Создание книги — успешный сценарий")
    void shouldCreateBook() {
        // Подготовка тестовых данных
        var request = new CreateBookRequest(
            "Война и мир", "Лев Толстой", 1869, "978-5-17-074647-1"
        );

        // Отправка запроса и получение ответа как POJO
        BookResponse response = given()
            .contentType(ContentType.JSON)
            .body(request)
        .when()
            .post("/api/books")
        .then()
            .statusCode(201)
            .extract()
            .as(BookResponse.class);

        // Проверки через типобезопасные assertions
        assertThat(response.getId()).isPositive();
        assertThat(response.getTitle()).isEqualTo("Война и мир");
        assertThat(response.getAuthor()).isEqualTo("Лев Толстой");
        assertThat(response.getPublishYear()).isEqualTo(1869);
        assertThat(response.getCreatedAt()).isNotNull();
    }

    @Test
    @DisplayName("Получение списка книг — десериализация массива")
    void shouldGetBooksList() {
        // Десериализация списка объектов
        List<BookResponse> books = given()
        .when()
            .get("/api/books")
        .then()
            .statusCode(200)
            .extract()
            .jsonPath()
            .getList(".", BookResponse.class);

        assertThat(books).isNotEmpty();
        assertThat(books).allSatisfy(book -> {
            assertThat(book.getId()).isPositive();
            assertThat(book.getTitle()).isNotBlank();
        });
    }
}
```

---

## Связь с тестированием

1. **Каждый API-тест** использует сериализацию (формирование запроса) и десериализацию (проверка ответа). Без POJO тесты превращаются в хрупкие строковые проверки.

2. **Типобезопасность:** ошибка в имени поля (`getName()` vs `getname()`) ловится компилятором, а не падением теста в рантайме.

3. **Тестовые данные из файлов:** Jackson позволяет хранить тестовые данные в JSON-файлах и десериализовать их в POJO, что упрощает управление большими наборами тестовых данных.

4. **Логирование:** `objectMapper.writerWithDefaultPrettyPrinter().writeValueAsString(object)` позволяет красиво выводить объекты в логи и отчёты.

5. **Contract testing:** POJO-классы фактически описывают контракт API. Если API изменит структуру ответа, тест упадёт при десериализации — это раннее обнаружение регрессии.

---

## Типичные ошибки

1. **Отсутствие конструктора по умолчанию.** Jackson требует no-arg конструктор для десериализации. Если его нет — `InvalidDefinitionException`. Часто забывают добавить его при написании POJO вручную.

2. **Не настроен `FAIL_ON_UNKNOWN_PROPERTIES = false`.** API возвращает 20 полей, а в POJO описаны только 5 — тест падает с `UnrecognizedPropertyException`. Это самая частая ошибка начинающих QA.

3. **Несовпадение имён полей.** API использует `user_name` (snake_case), а в Java — `userName` (camelCase). Без `@JsonProperty` или `@JsonNaming` Jackson не может выполнить маппинг, и поле остаётся `null`.

4. **Забывают подключить модуль для Java 8 дат.** При десериализации `LocalDateTime` без `JavaTimeModule` получают `InvalidDefinitionException: Java 8 date/time type not supported by default`.

---

## Вопросы на интервью

- 🟢 **Q:** Что такое сериализация и десериализация? Приведите пример в контексте API-тестов.
- **A:** Сериализация — преобразование Java-объекта в JSON (для тела запроса). Десериализация — преобразование JSON-ответа в Java-объект (для assertions). В REST Assured: `.body(pojo)` — сериализация, `.as(Class)` — десериализация.

- 🟢 **Q:** Какие основные методы ObjectMapper вы используете?
- **A:** `writeValueAsString(object)` — объект в JSON-строку, `readValue(json, Class)` — JSON в объект, `readValue(json, TypeReference)` — JSON в коллекцию объектов.

- 🟢 **Q:** Зачем нужна аннотация `@JsonProperty`?
- **A:** Для маппинга, когда имя поля в Java отличается от имени в JSON. Например, `@JsonProperty("user_name")` для поля `userName`.

- 🟡 **Q:** Чем `@JsonIgnore` отличается от `@JsonIgnoreProperties`?
- **A:** `@JsonIgnore` — на уровне одного поля, исключает его из (де)сериализации. `@JsonIgnoreProperties` — на уровне класса, может исключить несколько полей и имеет параметр `ignoreUnknown = true` для игнорирования неизвестных полей.

- 🟡 **Q:** Как десериализовать JSON-массив в `List<MyClass>`?
- **A:** Используя `TypeReference`: `objectMapper.readValue(json, new TypeReference<List<MyClass>>() {})`. Нельзя просто передать `List<MyClass>.class` из-за type erasure в Java.

- 🟡 **Q:** Зачем нужна настройка `FAIL_ON_UNKNOWN_PROPERTIES = false`?
- **A:** Чтобы Jackson не падал при встрече неизвестных полей в JSON. В API-тестах это критично: API может вернуть больше полей, чем описано в POJO. Альтернатива — аннотация `@JsonIgnoreProperties(ignoreUnknown = true)` на классе.

- 🟡 **Q:** Как работать с вложенными объектами в Jackson?
- **A:** Jackson автоматически десериализует вложенные объекты, если для них есть соответствующие POJO с конструктором по умолчанию. Для вложенных списков тоже — `List<Order>` в поле `UserResponse` десериализуется автоматически.

- 🔴 **Q:** Как создать кастомный десериализатор и когда он нужен?
- **A:** Наследоваться от `JsonDeserializer<T>`, переопределить метод `deserialize()`, применить через `@JsonDeserialize(using = ...)`. Нужен, когда API возвращает данные в нестандартном формате (числовые статусы вместо строковых, нестандартные форматы дат).

- 🔴 **Q:** Сравните Jackson и Gson. Когда какой выбрать?
- **A:** Jackson — стандарт в Spring/REST Assured экосистеме, богаче аннотациями, быстрее. Gson — проще, один JAR, хорош для Android. Для QA Automation на Java — однозначно Jackson, так как он уже интегрирован в основные инструменты.

---

## Практические задания

### Задание 1 (базовое): POJO и ObjectMapper

Создайте POJO-классы `CreateUserRequest` и `UserResponse` для REST API пользователей. Напишите тест, который:
- Сериализует `CreateUserRequest` в JSON и проверяет, что все поля присутствуют
- Десериализует JSON-строку ответа в `UserResponse` и проверяет значения полей

### Задание 2 (среднее): Работа с аннотациями Jackson

Создайте POJO для API, которое использует snake_case. Используйте `@JsonProperty` или `@JsonNaming`. Добавьте `@JsonInclude(NON_NULL)` и напишите тест для PATCH-запроса, где передаются только изменённые поля. Добавьте `@JsonFormat` для поля с датой.

### Задание 3 (продвинутое): Тестовые данные из JSON-файлов

Создайте JSON-файл с массивом тестовых данных (5+ пользователей с разными ролями и статусами). Напишите утилитный класс `TestDataLoader`, который десериализует данные из файла. Используйте `@ParameterizedTest` + `@MethodSource` для запуска тестов с данными из файла.

```java
class TestDataLoader {
    private static final ObjectMapper MAPPER = JacksonConfig.MAPPER;

    // Загрузка тестовых данных из JSON-файла
    public static <T> List<T> loadList(String filePath, Class<T> type) {
        try (var stream = TestDataLoader.class.getResourceAsStream(filePath)) {
            return MAPPER.readValue(stream, MAPPER.getTypeFactory()
                .constructCollectionType(List.class, type));
        } catch (IOException e) {
            throw new RuntimeException("Ошибка загрузки тестовых данных: " + filePath, e);
        }
    }
}
```

---

## Дополнительные ресурсы

- [Baeldung — Jackson ObjectMapper](https://www.baeldung.com/jackson-object-mapper-tutorial) — полный гайд по ObjectMapper
- [Baeldung — Jackson Annotations](https://www.baeldung.com/jackson-annotations) — все аннотации Jackson
- [Baeldung — Jackson Date](https://www.baeldung.com/jackson-serialize-dates) — работа с датами
- [Jackson Documentation](https://github.com/FasterXML/jackson-databind) — официальная документация
- [REST Assured — Serialization](https://github.com/rest-assured/rest-assured/wiki/Usage#serialization) — сериализация в REST Assured
- [Baeldung — Jackson vs Gson](https://www.baeldung.com/jackson-vs-gson) — сравнение библиотек
