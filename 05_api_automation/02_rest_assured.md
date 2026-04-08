# REST Assured

## Обзор

REST Assured — это Java-библиотека для тестирования REST API, которая предоставляет удобный DSL (Domain Specific Language) в стиле BDD (given/when/then). Она является стандартом де-факто для автоматизации API-тестов в Java-проектах. Для QA-инженера владение REST Assured — один из ключевых навыков при автоматизации тестирования.

REST Assured позволяет:
- Отправлять HTTP-запросы любого типа
- Валидировать статус-коды, заголовки, тело ответа
- Извлекать данные из JSON/XML-ответов
- Работать с аутентификацией
- Сериализовать/десериализовать объекты
- Валидировать JSON Schema

---

## Настройка проекта (Maven)

### Зависимости в pom.xml

```xml
<dependencies>
    <!-- REST Assured -->
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>rest-assured</artifactId>
        <version>5.4.0</version>
        <scope>test</scope>
    </dependency>

    <!-- JSON Schema Validation -->
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>json-schema-validator</artifactId>
        <version>5.4.0</version>
        <scope>test</scope>
    </dependency>

    <!-- Jackson для сериализации -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.17.0</version>
    </dependency>

    <!-- JUnit 5 -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.10.2</version>
        <scope>test</scope>
    </dependency>

    <!-- Hamcrest для матчеров -->
    <dependency>
        <groupId>org.hamcrest</groupId>
        <artifactId>hamcrest</artifactId>
        <version>2.2</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Статические импорты

```java
import static io.restassured.RestAssured.*;
import static io.restassured.matcher.RestAssuredMatchers.*;
import static org.hamcrest.Matchers.*;
```

---

## Паттерн Given / When / Then

REST Assured использует BDD-подход, разбивая тест на три логические части:

| Блок | Назначение | Примеры |
|------|-----------|---------|
| `given()` | Предусловия: заголовки, параметры, тело запроса, авторизация | headers, params, body, auth |
| `when()` | Действие: HTTP-метод и URL | get(), post(), put(), delete() |
| `then()` | Проверки: валидация ответа | statusCode(), body(), header() |

```java
// Базовый пример — получение списка пользователей
given()
    .baseUri("https://jsonplaceholder.typicode.com")
    .header("Accept", "application/json")
.when()
    .get("/users")
.then()
    .statusCode(200)
    .contentType(ContentType.JSON)
    .body("size()", greaterThan(0));
```

---

## Основные HTTP-запросы

### GET-запрос

```java
@Test
void shouldGetUserById() {
    given()
        .baseUri("https://jsonplaceholder.typicode.com")
    .when()
        .get("/users/1")
    .then()
        .statusCode(200)
        .body("id", equalTo(1))
        .body("name", notNullValue())
        .body("email", containsString("@"));
}
```

### POST-запрос

```java
@Test
void shouldCreateNewPost() {
    // Тело запроса в формате JSON-строки
    String requestBody = """
        {
            "title": "Тестовый пост",
            "body": "Содержимое поста",
            "userId": 1
        }
        """;

    given()
        .baseUri("https://jsonplaceholder.typicode.com")
        .contentType(ContentType.JSON)
        .body(requestBody)
    .when()
        .post("/posts")
    .then()
        .statusCode(201)
        .body("id", notNullValue())
        .body("title", equalTo("Тестовый пост"));
}
```

### PUT-запрос

```java
@Test
void shouldUpdatePost() {
    String updatedBody = """
        {
            "id": 1,
            "title": "Обновлённый заголовок",
            "body": "Обновлённое содержимое",
            "userId": 1
        }
        """;

    given()
        .baseUri("https://jsonplaceholder.typicode.com")
        .contentType(ContentType.JSON)
        .body(updatedBody)
    .when()
        .put("/posts/1")
    .then()
        .statusCode(200)
        .body("title", equalTo("Обновлённый заголовок"));
}
```

### PATCH-запрос

```java
@Test
void shouldPartiallyUpdatePost() {
    // Только обновляемое поле
    String patchBody = """
        {
            "title": "Частично обновлённый заголовок"
        }
        """;

    given()
        .baseUri("https://jsonplaceholder.typicode.com")
        .contentType(ContentType.JSON)
        .body(patchBody)
    .when()
        .patch("/posts/1")
    .then()
        .statusCode(200)
        .body("title", equalTo("Частично обновлённый заголовок"))
        .body("body", notNullValue()); // Остальные поля не затронуты
}
```

### DELETE-запрос

```java
@Test
void shouldDeletePost() {
    given()
        .baseUri("https://jsonplaceholder.typicode.com")
    .when()
        .delete("/posts/1")
    .then()
        .statusCode(200); // Или 204 в зависимости от API
}
```

---

## Заголовки и Query Parameters

### Работа с заголовками

```java
given()
    .header("Authorization", "Bearer eyJhbGciOi...")
    .header("Accept-Language", "ru")
    .headers(
        "X-Custom-Header", "value1",
        "X-Another-Header", "value2"
    )
.when()
    .get("/api/resource");
```

### Query Parameters

```java
// Одиночные параметры
given()
    .queryParam("page", 1)
    .queryParam("size", 20)
    .queryParam("sort", "name,asc")
.when()
    .get("/api/users");
// Результат: GET /api/users?page=1&size=20&sort=name,asc

// Path Parameters
given()
    .pathParam("userId", 42)
.when()
    .get("/api/users/{userId}");
// Результат: GET /api/users/42
```

---

## Сериализация и десериализация (POJO)

### Модель данных

```java
// POJO-класс для пользователя
public class User {
    private Integer id;
    private String name;
    private String email;
    private String phone;

    // Конструктор без аргументов нужен для десериализации
    public User() {}

    public User(String name, String email, String phone) {
        this.name = name;
        this.email = email;
        this.phone = phone;
    }

    // Геттеры и сеттеры
    public Integer getId() { return id; }
    public void setId(Integer id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public String getPhone() { return phone; }
    public void setPhone(String phone) { this.phone = phone; }
}
```

### POJO → JSON (сериализация)

```java
@Test
void shouldCreateUserWithPojo() {
    User newUser = new User("Тест Тестов", "test@example.com", "+7-999-123-45-67");

    // REST Assured автоматически сериализует POJO в JSON
    given()
        .baseUri("https://api.example.com")
        .contentType(ContentType.JSON)
        .body(newUser)
    .when()
        .post("/api/users")
    .then()
        .statusCode(201)
        .body("name", equalTo("Тест Тестов"));
}
```

### JSON → POJO (десериализация)

```java
@Test
void shouldDeserializeResponseToPojo() {
    User user = given()
        .baseUri("https://jsonplaceholder.typicode.com")
    .when()
        .get("/users/1")
    .then()
        .statusCode(200)
        .extract()
        .as(User.class); // Десериализация JSON → POJO

    // Проверки через стандартные assertions
    assertNotNull(user.getId());
    assertEquals("Leanne Graham", user.getName());
    assertTrue(user.getEmail().contains("@"));
}
```

### Извлечение списка объектов

```java
@Test
void shouldDeserializeListOfUsers() {
    List<User> users = given()
        .baseUri("https://jsonplaceholder.typicode.com")
    .when()
        .get("/users")
    .then()
        .statusCode(200)
        .extract()
        .jsonPath()
        .getList(".", User.class);

    assertEquals(10, users.size());
    assertTrue(users.stream().allMatch(u -> u.getEmail() != null));
}
```

---

## Извлечение данных из JSON (JsonPath)

### Основные выражения

```java
Response response = given()
    .baseUri("https://jsonplaceholder.typicode.com")
.when()
    .get("/users/1");

// Извлечение одиночных значений
String name = response.jsonPath().getString("name");
int id = response.jsonPath().getInt("id");

// Извлечение вложенных объектов
String city = response.jsonPath().getString("address.city");
String lat = response.jsonPath().getString("address.geo.lat");

// Извлечение из массивов
Response postsResponse = get("https://jsonplaceholder.typicode.com/posts");
String firstTitle = postsResponse.jsonPath().getString("[0].title");
List<Integer> allUserIds = postsResponse.jsonPath().getList("userId", Integer.class);
```

### Inline-валидация через body()

```java
@Test
void shouldValidateJsonStructure() {
    given()
        .baseUri("https://jsonplaceholder.typicode.com")
    .when()
        .get("/users")
    .then()
        .body("size()", equalTo(10))                          // Размер массива
        .body("[0].name", equalTo("Leanne Graham"))           // Первый элемент
        .body("name", hasItems("Leanne Graham", "Ervin Howell")) // Содержит элементы
        .body("email", everyItem(containsString("@")))        // Каждый email содержит @
        .body("findAll { it.id > 5 }.size()", equalTo(5));    // GPath-выражение
}
```

---

## JSON Schema Validation

JSON Schema позволяет валидировать структуру ответа: типы полей, обязательные поля, формат.

### Файл схемы: `src/test/resources/schemas/user-schema.json`

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["id", "name", "email"],
  "properties": {
    "id": { "type": "integer" },
    "name": { "type": "string", "minLength": 1 },
    "email": { "type": "string", "format": "email" },
    "phone": { "type": "string" },
    "website": { "type": "string" },
    "address": {
      "type": "object",
      "properties": {
        "city": { "type": "string" },
        "zipcode": { "type": "string" }
      }
    }
  },
  "additionalProperties": true
}
```

### Тест с валидацией схемы

```java
import static io.restassured.module.jsv.JsonSchemaValidator.matchesJsonSchemaInClasspath;

@Test
void shouldMatchUserJsonSchema() {
    given()
        .baseUri("https://jsonplaceholder.typicode.com")
    .when()
        .get("/users/1")
    .then()
        .statusCode(200)
        .body(matchesJsonSchemaInClasspath("schemas/user-schema.json"));
}
```

---

## Request и Response Specifications

Спецификации позволяют вынести общую конфигурацию, чтобы не дублировать код.

### RequestSpecification

```java
public class ApiSpecs {

    // Базовая спецификация запроса
    public static RequestSpecification baseRequestSpec() {
        return new RequestSpecBuilder()
            .setBaseUri("https://api.example.com")
            .setBasePath("/api/v1")
            .setContentType(ContentType.JSON)
            .addHeader("Accept", "application/json")
            .addFilter(new AllureRestAssured()) // Интеграция с Allure
            .log(LogDetail.ALL)
            .build();
    }

    // Спецификация с авторизацией
    public static RequestSpecification authRequestSpec(String token) {
        return new RequestSpecBuilder()
            .addRequestSpecification(baseRequestSpec())
            .addHeader("Authorization", "Bearer " + token)
            .build();
    }
}
```

### ResponseSpecification

```java
public class ApiSpecs {

    // Спецификация успешного ответа
    public static ResponseSpecification successResponseSpec() {
        return new ResponseSpecBuilder()
            .expectStatusCode(200)
            .expectContentType(ContentType.JSON)
            .build();
    }

    // Спецификация ответа при создании ресурса
    public static ResponseSpecification createdResponseSpec() {
        return new ResponseSpecBuilder()
            .expectStatusCode(201)
            .expectContentType(ContentType.JSON)
            .build();
    }
}
```

### Использование спецификаций в тестах

```java
@Test
void shouldGetUsersWithSpecs() {
    given()
        .spec(ApiSpecs.authRequestSpec("my-token"))
    .when()
        .get("/users")
    .then()
        .spec(ApiSpecs.successResponseSpec())
        .body("size()", greaterThan(0));
}
```

---

## Логирование

### Логирование запросов и ответов

```java
// Логировать всё
given()
    .log().all()      // Логирование всего запроса
.when()
    .get("/users")
.then()
    .log().all();      // Логирование всего ответа

// Логировать только при ошибке
given()
    .log().ifValidationFails()
.when()
    .get("/users")
.then()
    .log().ifError()              // Логировать при HTTP-ошибке (4xx, 5xx)
    .log().ifValidationFails()    // Логировать при ошибке валидации
    .statusCode(200);

// Логировать конкретные части
given()
    .log().headers()    // Только заголовки
    .log().body()       // Только тело
    .log().params()     // Только параметры
.when()
    .get("/users");
```

### Глобальная конфигурация логирования

```java
@BeforeAll
static void setup() {
    // Включить логирование при ошибках для всех тестов
    RestAssured.enableLoggingOfRequestAndResponseIfValidationFails();

    // Или через фильтры
    RestAssured.filters(new RequestLoggingFilter(), new ResponseLoggingFilter());
}
```

---

## Фильтры

Фильтры позволяют перехватывать и модифицировать запросы/ответы.

```java
// Кастомный фильтр для добавления correlation id
public class CorrelationIdFilter implements Filter {
    @Override
    public Response filter(FilterableRequestSpecification requestSpec,
                           FilterableResponseSpecification responseSpec,
                           FilterContext ctx) {
        // Добавляем заголовок перед отправкой запроса
        requestSpec.header("X-Correlation-Id", UUID.randomUUID().toString());
        return ctx.next(requestSpec, responseSpec);
    }
}

// Использование
given()
    .filter(new CorrelationIdFilter())
.when()
    .get("/api/resource");
```

---

## Полный пример CRUD-тестов

```java
class UserApiTest {

    private static final String BASE_URI = "https://api.example.com";
    private static RequestSpecification requestSpec;

    @BeforeAll
    static void setup() {
        requestSpec = new RequestSpecBuilder()
            .setBaseUri(BASE_URI)
            .setBasePath("/api/v1")
            .setContentType(ContentType.JSON)
            .addHeader("Authorization", "Bearer test-token")
            .build();
    }

    @Test
    @DisplayName("Получение списка пользователей")
    void shouldGetAllUsers() {
        given()
            .spec(requestSpec)
            .queryParam("page", 0)
            .queryParam("size", 10)
        .when()
            .get("/users")
        .then()
            .statusCode(200)
            .body("content.size()", lessThanOrEqualTo(10))
            .body("totalElements", greaterThanOrEqualTo(0));
    }

    @Test
    @DisplayName("Создание нового пользователя")
    void shouldCreateUser() {
        User newUser = new User("Тест Тестов", "test@example.com", "+7-999-000-00-00");

        int userId = given()
            .spec(requestSpec)
            .body(newUser)
        .when()
            .post("/users")
        .then()
            .statusCode(201)
            .body("name", equalTo("Тест Тестов"))
            .body("id", notNullValue())
            .extract()
            .path("id");

        // Проверяем, что пользователь действительно создан
        given()
            .spec(requestSpec)
        .when()
            .get("/users/" + userId)
        .then()
            .statusCode(200)
            .body("name", equalTo("Тест Тестов"));
    }

    @Test
    @DisplayName("Обновление пользователя")
    void shouldUpdateUser() {
        User updatedUser = new User("Обновлённое Имя", "updated@example.com", "+7-999-111-11-11");

        given()
            .spec(requestSpec)
            .body(updatedUser)
        .when()
            .put("/users/1")
        .then()
            .statusCode(200)
            .body("name", equalTo("Обновлённое Имя"))
            .body("email", equalTo("updated@example.com"));
    }

    @Test
    @DisplayName("Удаление пользователя")
    void shouldDeleteUser() {
        given()
            .spec(requestSpec)
        .when()
            .delete("/users/1")
        .then()
            .statusCode(204);

        // Проверяем, что пользователь удалён
        given()
            .spec(requestSpec)
        .when()
            .get("/users/1")
        .then()
            .statusCode(404);
    }

    @Test
    @DisplayName("Попытка создания пользователя без обязательного поля")
    void shouldReturn400WhenNameMissing() {
        String invalidBody = """
            {
                "email": "test@example.com"
            }
            """;

        given()
            .spec(requestSpec)
            .body(invalidBody)
        .when()
            .post("/users")
        .then()
            .statusCode(400)
            .body("errors", hasSize(greaterThan(0)));
    }

    @Test
    @DisplayName("Попытка получения несуществующего пользователя")
    void shouldReturn404ForNonExistentUser() {
        given()
            .spec(requestSpec)
        .when()
            .get("/users/99999")
        .then()
            .statusCode(404);
    }
}
```

---

## Связь с тестированием

REST Assured — основной инструмент QA-инженера для:
- **Smoke-тестов API** — быстрая проверка доступности и базовой функциональности
- **Регрессионных тестов** — автоматизированная проверка после изменений
- **Контрактных тестов** — валидация JSON Schema
- **Интеграционных тестов** — проверка взаимодействия между сервисами
- **Data-driven тестов** — параметризация с помощью JUnit 5 `@ParameterizedTest`

---

## Типичные ошибки

1. **Отсутствие `contentType(ContentType.JSON)` при POST/PUT** — сервер может вернуть 415 Unsupported Media Type
2. **Хардкод baseUri в каждом тесте** — используйте спецификации или `RestAssured.baseURI`
3. **Игнорирование порядка выполнения тестов** — тесты должны быть независимыми; не полагайтесь на данные, созданные другим тестом
4. **Не используют `extract()`** — вместо этого делают повторный запрос для получения данных
5. **Отсутствие логирования** — при падении теста невозможно понять, что было в запросе/ответе
6. **Проверка только statusCode** — нужно также проверять тело, заголовки, структуру
7. **Не используют спецификации** — много дублирования кода между тестами
8. **Забывают про clean-up** — после создания тестовых данных нужно их удалять

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Что такое REST Assured? Для чего используется?
2. Объясните паттерн given/when/then в REST Assured.
3. Как отправить GET-запрос и проверить статус-код?
4. Как передать заголовки в запрос?
5. Как передать тело запроса в формате JSON?

### 🟡 Средний уровень
6. Как извлечь значение из JSON-ответа? Приведите примеры JsonPath.
7. Что такое RequestSpecification и ResponseSpecification? Зачем они нужны?
8. Как выполнить JSON Schema validation?
9. Как сериализовать POJO в JSON и обратно?
10. Как настроить логирование в REST Assured?
11. Как параметризовать тесты REST Assured с JUnit 5?

### 🔴 Продвинутый уровень
12. Как реализовать кастомный фильтр для REST Assured?
13. Как организовать архитектуру API-тестов для большого проекта?
14. Как интегрировать REST Assured с Allure для отчётности?
15. Как тестировать multipart/form-data и загрузку файлов?
16. Как обрабатывать динамические данные (timestamps, UUIDs) при валидации?

---

## Практические задания

### Задание 1: Базовые CRUD-тесты
Используя JSONPlaceholder (`https://jsonplaceholder.typicode.com`), напишите тесты для CRUD-операций над ресурсом `/posts`. Используйте спецификации для общей конфигурации.

### Задание 2: Валидация JSON Schema
Создайте JSON Schema для ресурса `/users` из JSONPlaceholder. Напишите тест, который валидирует ответ против этой схемы.

### Задание 3: Сериализация с POJO
Создайте POJO-классы для ресурсов `/posts` и `/comments`. Напишите тесты, которые создают ресурсы через POJO и десериализуют ответы.

### Задание 4: Параметризация
Напишите параметризованный тест (`@ParameterizedTest`), который проверяет получение пользователей по ID от 1 до 10.

---

## Дополнительные ресурсы

- [REST Assured Official Documentation](https://rest-assured.io)
- [REST Assured GitHub](https://github.com/rest-assured/rest-assured)
- [REST Assured Wiki](https://github.com/rest-assured/rest-assured/wiki/Usage)
- [Hamcrest Matchers Reference](http://hamcrest.org/JavaHamcrest/javadoc/2.2/)
- [JSONPlaceholder — бесплатный API для практики](https://jsonplaceholder.typicode.com)
- [JSON Schema Specification](https://json-schema.org)
