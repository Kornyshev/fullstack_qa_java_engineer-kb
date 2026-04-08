# Пошаговые упражнения REST Assured

## Содержание

1. [Подготовка проекта](#подготовка-проекта)
2. [Упражнение 1: GET + проверка статуса и тела](#упражнение-1-get--проверка-статуса-и-тела)
3. [Упражнение 2: POST с JSON и десериализация в POJO](#упражнение-2-post-с-json-и-десериализация-в-pojo)
4. [Упражнение 3: Параметризованные тесты с @MethodSource](#упражнение-3-параметризованные-тесты-с-methodsource)
5. [Упражнение 4: Bearer Token авторизация](#упражнение-4-bearer-token-авторизация)
6. [Упражнение 5: JSON Schema Validation](#упражнение-5-json-schema-validation)
7. [Упражнение 6: RequestSpecification для DRY](#упражнение-6-requestspecification-для-dry)
8. [Итоговое задание](#итоговое-задание)

---

## Подготовка проекта

### Maven-зависимости

Создайте Maven-проект и добавьте в `pom.xml`:

```xml
<properties>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
    <rest-assured.version>5.4.0</rest-assured.version>
    <junit.version>5.10.2</junit.version>
    <jackson.version>2.17.0</jackson.version>
</properties>

<dependencies>
    <!-- REST Assured -->
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>rest-assured</artifactId>
        <version>${rest-assured.version}</version>
        <scope>test</scope>
    </dependency>

    <!-- JSON Schema Validation -->
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>json-schema-validator</artifactId>
        <version>${rest-assured.version}</version>
        <scope>test</scope>
    </dependency>

    <!-- JUnit 5 -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>${junit.version}</version>
        <scope>test</scope>
    </dependency>

    <!-- Jackson для сериализации/десериализации -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>${jackson.version}</version>
    </dependency>
</dependencies>
```

### Структура проекта

```
src/
├── main/java/
│   └── com/qa/models/
│       ├── Post.java
│       └── User.java
└── test/
    ├── java/com/qa/tests/
    │   ├── Exercise01_GetTest.java
    │   ├── Exercise02_PostTest.java
    │   ├── Exercise03_ParameterizedTest.java
    │   ├── Exercise04_AuthTest.java
    │   ├── Exercise05_SchemaTest.java
    │   └── Exercise06_SpecTest.java
    └── resources/
        └── schemas/
            └── post-schema.json
```

---

## Упражнение 1: GET + проверка статуса и тела

### Задание

Напишите тест, который отправляет GET-запрос на `https://jsonplaceholder.typicode.com/posts/1` и проверяет:
- Статус-код 200
- `userId` равен 1
- `id` равен 1
- `title` не пустой
- Заголовок `Content-Type` содержит `application/json`

### Подсказка

Используйте статические импорты REST Assured:
- `given()`, `when()`, `then()` — для BDD-стиля
- `equalTo()`, `notNullValue()` — Hamcrest matchers
- `contentType(ContentType.JSON)` — проверка Content-Type

### Решение

```java
package com.qa.tests;

import io.restassured.http.ContentType;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

public class Exercise01_GetTest {

    private static final String BASE_URL = "https://jsonplaceholder.typicode.com";

    @Test
    void shouldReturnPostById() {
        // Отправляем GET-запрос и проверяем ответ
        given()
            .baseUri(BASE_URL)
        .when()
            .get("/posts/1")
        .then()
            .statusCode(200)                          // Проверяем статус-код
            .contentType(ContentType.JSON)             // Проверяем Content-Type
            .body("userId", equalTo(1))                // Проверяем userId
            .body("id", equalTo(1))                    // Проверяем id
            .body("title", not(emptyOrNullString()));  // Проверяем, что title не пустой
    }

    @Test
    void shouldReturnAllPosts() {
        // Получаем список всех постов и проверяем количество
        given()
            .baseUri(BASE_URL)
        .when()
            .get("/posts")
        .then()
            .statusCode(200)
            .body("size()", equalTo(100))              // 100 постов в массиве
            .body("[0].id", equalTo(1))                // Первый пост имеет id=1
            .body("userId", everyItem(notNullValue())); // У каждого поста есть userId
    }

    @Test
    void shouldReturnEmptyObjectForNonExistentPost() {
        // Проверяем поведение при запросе несуществующего ресурса
        given()
            .baseUri(BASE_URL)
        .when()
            .get("/posts/9999")
        .then()
            .statusCode(404);
    }

    @Test
    void shouldFilterCommentsByPostId() {
        // Фильтрация с помощью query-параметра
        given()
            .baseUri(BASE_URL)
            .queryParam("postId", 1)
        .when()
            .get("/comments")
        .then()
            .statusCode(200)
            .body("size()", equalTo(5))                     // 5 комментариев у поста 1
            .body("postId", everyItem(equalTo(1)))          // Все принадлежат посту 1
            .body("email", everyItem(containsString("@"))); // У всех есть email
    }
}
```

---

## Упражнение 2: POST с JSON и десериализация в POJO

### Задание

1. Создайте POJO-класс `Post` с полями `userId`, `id`, `title`, `body`
2. Отправьте POST-запрос с объектом `Post` в теле
3. Десериализуйте ответ обратно в `Post`
4. Проверьте значения полей через assertions JUnit

### Подсказка

- Для сериализации используйте `.body(postObject)`
- Для десериализации: `.extract().as(Post.class)`
- Не забудьте указать `ContentType.JSON`

### Решение

#### POJO-модель

```java
package com.qa.models;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

// Игнорируем неизвестные поля при десериализации
@JsonIgnoreProperties(ignoreUnknown = true)
public class Post {

    private Integer userId;
    private Integer id;
    private String title;
    private String body;

    // Пустой конструктор нужен для Jackson
    public Post() {}

    // Конструктор для удобного создания объектов в тестах
    public Post(Integer userId, String title, String body) {
        this.userId = userId;
        this.title = title;
        this.body = body;
    }

    // Геттеры и сеттеры
    public Integer getUserId() { return userId; }
    public void setUserId(Integer userId) { this.userId = userId; }

    public Integer getId() { return id; }
    public void setId(Integer id) { this.id = id; }

    public String getTitle() { return title; }
    public void setTitle(String title) { this.title = title; }

    public String getBody() { return body; }
    public void setBody(String body) { this.body = body; }

    @Override
    public String toString() {
        return "Post{userId=" + userId + ", id=" + id +
               ", title='" + title + "', body='" + body + "'}";
    }
}
```

#### Тестовый класс

```java
package com.qa.tests;

import com.qa.models.Post;
import io.restassured.http.ContentType;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static org.junit.jupiter.api.Assertions.*;

public class Exercise02_PostTest {

    private static final String BASE_URL = "https://jsonplaceholder.typicode.com";

    @Test
    void shouldCreatePostAndDeserializeResponse() {
        // Создаём объект для отправки
        Post newPost = new Post(1, "Заголовок теста", "Тело тестового поста");

        // Отправляем POST и десериализуем ответ
        Post createdPost = given()
            .baseUri(BASE_URL)
            .contentType(ContentType.JSON)
            .body(newPost)
        .when()
            .post("/posts")
        .then()
            .statusCode(201)
            .extract()
            .as(Post.class);

        // Проверяем через JUnit assertions
        assertAll("Проверка созданного поста",
            () -> assertNotNull(createdPost.getId(), "id не должен быть null"),
            () -> assertEquals(1, createdPost.getUserId(), "userId должен совпадать"),
            () -> assertEquals("Заголовок теста", createdPost.getTitle(), "title должен совпадать"),
            () -> assertEquals("Тело тестового поста", createdPost.getBody(), "body должен совпадать")
        );
    }

    @Test
    void shouldDeserializeGetResponseToPojo() {
        // Десериализация GET-ответа
        Post post = given()
            .baseUri(BASE_URL)
        .when()
            .get("/posts/1")
        .then()
            .statusCode(200)
            .extract()
            .as(Post.class);

        assertAll("Проверка поста #1",
            () -> assertEquals(1, post.getId()),
            () -> assertEquals(1, post.getUserId()),
            () -> assertFalse(post.getTitle().isEmpty(), "Заголовок не должен быть пустым"),
            () -> assertFalse(post.getBody().isEmpty(), "Тело не должно быть пустым")
        );
    }

    @Test
    void shouldDeserializeListOfPosts() {
        // Десериализация массива объектов
        Post[] posts = given()
            .baseUri(BASE_URL)
            .queryParam("userId", 1)
        .when()
            .get("/posts")
        .then()
            .statusCode(200)
            .extract()
            .as(Post[].class);

        // Проверяем массив
        assertTrue(posts.length > 0, "Должен быть хотя бы один пост");
        for (Post post : posts) {
            assertEquals(1, post.getUserId(), "Все посты должны принадлежать userId=1");
        }
    }
}
```

---

## Упражнение 3: Параметризованные тесты с @MethodSource

### Задание

Напишите параметризованный тест, который проверяет получение пользователей по ID. Для каждого пользователя проверьте `id`, `name`, `email`. Используйте `@MethodSource` для поставки данных.

### Подсказка

- `@ParameterizedTest` вместо `@Test`
- `@MethodSource("provideUserData")` — ссылка на статический метод
- `Stream<Arguments>` — возвращаемый тип метода-источника

### Решение

```java
package com.qa.tests;

import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.Arguments;
import org.junit.jupiter.params.provider.MethodSource;
import org.junit.jupiter.params.provider.CsvSource;

import java.util.stream.Stream;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

public class Exercise03_ParameterizedTest {

    private static final String BASE_URL = "https://jsonplaceholder.typicode.com";

    // Источник данных: id, имя, email
    static Stream<Arguments> provideUserData() {
        return Stream.of(
            Arguments.of(1, "Leanne Graham", "Sincere@april.biz"),
            Arguments.of(2, "Ervin Howell", "Shanna@melissa.tv"),
            Arguments.of(3, "Clementine Bauch", "Nathan@yesenia.net"),
            Arguments.of(4, "Patricia Lebsack", "Julianne.OConner@kory.org"),
            Arguments.of(5, "Chelsey Dietrich", "Lucio_Hettinger@annie.ca")
        );
    }

    @ParameterizedTest(name = "Пользователь #{0}: {1}")
    @MethodSource("provideUserData")
    void shouldReturnCorrectUserData(int id, String name, String email) {
        // Проверяем данные каждого пользователя
        given()
            .baseUri(BASE_URL)
        .when()
            .get("/users/" + id)
        .then()
            .statusCode(200)
            .body("id", equalTo(id))
            .body("name", equalTo(name))
            .body("email", equalTo(email));
    }

    // Альтернативный вариант: @CsvSource для простых данных
    @ParameterizedTest(name = "Пост #{0} принадлежит пользователю #{1}")
    @CsvSource({
        "1,  1",
        "11, 2",
        "21, 3",
        "31, 4",
        "41, 5"
    })
    void shouldReturnPostWithCorrectUserId(int postId, int expectedUserId) {
        given()
            .baseUri(BASE_URL)
        .when()
            .get("/posts/" + postId)
        .then()
            .statusCode(200)
            .body("id", equalTo(postId))
            .body("userId", equalTo(expectedUserId));
    }

    // Параметризованный негативный тест
    static Stream<Arguments> provideInvalidEndpoints() {
        return Stream.of(
            Arguments.of("/posts/0", 404, "Пост с id=0"),
            Arguments.of("/posts/-1", 404, "Пост с отрицательным id"),
            Arguments.of("/posts/abc", 404, "Пост с нечисловым id"),
            Arguments.of("/invalid", 404, "Несуществующий эндпоинт")
        );
    }

    @ParameterizedTest(name = "Негативный тест: {2}")
    @MethodSource("provideInvalidEndpoints")
    void shouldReturn404ForInvalidEndpoints(String endpoint, int expectedStatus, String description) {
        given()
            .baseUri(BASE_URL)
        .when()
            .get(endpoint)
        .then()
            .statusCode(expectedStatus);
    }
}
```

---

## Упражнение 4: Bearer Token авторизация

### Задание

Используя Reqres API (`https://reqres.in`):
1. Отправьте POST на `/api/login` для получения токена
2. Используйте полученный токен в заголовке Authorization
3. Проверьте работу с невалидным токеном

### Подсказка

- `.header("Authorization", "Bearer " + token)` — добавление заголовка
- `.auth().oauth2(token)` — встроенный метод REST Assured
- Сначала извлеките `token` из ответа на `/api/login`

### Решение

```java
package com.qa.tests;

import io.restassured.http.ContentType;
import io.restassured.response.Response;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.Order;
import org.junit.jupiter.api.TestMethodOrder;
import org.junit.jupiter.api.MethodOrderer;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;
import static org.junit.jupiter.api.Assertions.*;

@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
public class Exercise04_AuthTest {

    private static final String BASE_URL = "https://reqres.in";
    private static String authToken;

    @Test
    @Order(1)
    void shouldLoginAndReceiveToken() {
        // Логинимся и получаем токен
        String loginBody = """
            {
                "email": "eve.holt@reqres.in",
                "password": "cityslicka"
            }
            """;

        Response response = given()
            .baseUri(BASE_URL)
            .contentType(ContentType.JSON)
            .body(loginBody)
        .when()
            .post("/api/login")
        .then()
            .statusCode(200)
            .body("token", notNullValue())
            .extract()
            .response();

        // Сохраняем токен для последующих запросов
        authToken = response.jsonPath().getString("token");
        assertFalse(authToken.isEmpty(), "Токен не должен быть пустым");
        System.out.println("Получен токен: " + authToken);
    }

    @Test
    @Order(2)
    void shouldAccessProtectedResourceWithToken() {
        // Используем токен для авторизации (через header)
        given()
            .baseUri(BASE_URL)
            .header("Authorization", "Bearer " + authToken)
        .when()
            .get("/api/users/2")
        .then()
            .statusCode(200)
            .body("data.id", equalTo(2))
            .body("data.email", notNullValue());
    }

    @Test
    @Order(3)
    void shouldAccessWithOAuth2Method() {
        // Альтернативный способ: встроенный метод auth().oauth2()
        given()
            .baseUri(BASE_URL)
            .auth().oauth2(authToken)
        .when()
            .get("/api/users?page=1")
        .then()
            .statusCode(200)
            .body("data.size()", greaterThan(0));
    }

    @Test
    void shouldFailLoginWithoutPassword() {
        // Негативный тест: логин без пароля
        String body = """
            {
                "email": "peter@klaven"
            }
            """;

        given()
            .baseUri(BASE_URL)
            .contentType(ContentType.JSON)
            .body(body)
        .when()
            .post("/api/login")
        .then()
            .statusCode(400)
            .body("error", equalTo("Missing password"));
    }

    @Test
    void shouldRegisterSuccessfully() {
        // Успешная регистрация
        String body = """
            {
                "email": "eve.holt@reqres.in",
                "password": "pistol"
            }
            """;

        given()
            .baseUri(BASE_URL)
            .contentType(ContentType.JSON)
            .body(body)
        .when()
            .post("/api/register")
        .then()
            .statusCode(200)
            .body("id", notNullValue())
            .body("token", notNullValue());
    }

    @Test
    void shouldFailRegisterWithoutPassword() {
        // Негативный тест: регистрация без пароля
        String body = """
            {
                "email": "sydney@fife"
            }
            """;

        given()
            .baseUri(BASE_URL)
            .contentType(ContentType.JSON)
            .body(body)
        .when()
            .post("/api/register")
        .then()
            .statusCode(400)
            .body("error", equalTo("Missing password"));
    }
}
```

---

## Упражнение 5: JSON Schema Validation

### Задание

1. Создайте JSON Schema для объекта `Post`
2. Напишите тест, который валидирует ответ API по этой схеме
3. Создайте невалидную схему и убедитесь, что тест падает

### Подсказка

- Файл схемы размещается в `src/test/resources/schemas/`
- `matchesJsonSchemaInClasspath("schemas/post-schema.json")` — матчер
- Используйте `"required"` для обязательных полей

### Решение

#### JSON Schema: `src/test/resources/schemas/post-schema.json`

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Post",
  "description": "Схема объекта поста JSONPlaceholder",
  "type": "object",
  "properties": {
    "userId": {
      "type": "integer",
      "minimum": 1,
      "description": "Идентификатор автора"
    },
    "id": {
      "type": "integer",
      "minimum": 1,
      "description": "Уникальный идентификатор поста"
    },
    "title": {
      "type": "string",
      "minLength": 1,
      "description": "Заголовок поста"
    },
    "body": {
      "type": "string",
      "minLength": 1,
      "description": "Содержимое поста"
    }
  },
  "required": ["userId", "id", "title", "body"],
  "additionalProperties": false
}
```

#### JSON Schema для массива постов: `src/test/resources/schemas/posts-array-schema.json`

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Posts Array",
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "userId": { "type": "integer", "minimum": 1 },
      "id": { "type": "integer", "minimum": 1 },
      "title": { "type": "string", "minLength": 1 },
      "body": { "type": "string", "minLength": 1 }
    },
    "required": ["userId", "id", "title", "body"]
  },
  "minItems": 1
}
```

#### Тестовый класс

```java
package com.qa.tests;

import io.restassured.http.ContentType;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static io.restassured.module.jsv.JsonSchemaValidator.matchesJsonSchemaInClasspath;

public class Exercise05_SchemaTest {

    private static final String BASE_URL = "https://jsonplaceholder.typicode.com";

    @Test
    void shouldMatchPostSchema() {
        // Валидация одного поста по JSON Schema
        given()
            .baseUri(BASE_URL)
        .when()
            .get("/posts/1")
        .then()
            .statusCode(200)
            .body(matchesJsonSchemaInClasspath("schemas/post-schema.json"));
    }

    @Test
    void shouldMatchPostsArraySchema() {
        // Валидация массива постов
        given()
            .baseUri(BASE_URL)
        .when()
            .get("/posts")
        .then()
            .statusCode(200)
            .body(matchesJsonSchemaInClasspath("schemas/posts-array-schema.json"));
    }

    @Test
    void shouldValidatePostCreationResponse() {
        // При создании поста ответ тоже должен соответствовать схеме
        // Примечание: JSONPlaceholder возвращает id=101 для нового поста
        String body = """
            {
                "title": "Тестовый пост",
                "body": "Содержимое поста",
                "userId": 1
            }
            """;

        given()
            .baseUri(BASE_URL)
            .contentType(ContentType.JSON)
            .body(body)
        .when()
            .post("/posts")
        .then()
            .statusCode(201)
            .body(matchesJsonSchemaInClasspath("schemas/post-schema.json"));
    }
}
```

---

## Упражнение 6: RequestSpecification для DRY

### Задание

Вынесите повторяющуюся конфигурацию (baseUri, contentType, logging) в `RequestSpecification` и `ResponseSpecification`. Перепишите тесты из предыдущих упражнений, используя спецификации.

### Подсказка

- `RequestSpecBuilder` — для построения спецификации запроса
- `ResponseSpecBuilder` — для спецификации ответа
- `RestAssured.requestSpecification` — глобальная установка
- `@BeforeAll` / `@BeforeEach` — для инициализации

### Решение

```java
package com.qa.tests;

import io.restassured.RestAssured;
import io.restassured.builder.RequestSpecBuilder;
import io.restassured.builder.ResponseSpecBuilder;
import io.restassured.filter.log.LogDetail;
import io.restassured.http.ContentType;
import io.restassured.specification.RequestSpecification;
import io.restassured.specification.ResponseSpecification;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

public class Exercise06_SpecTest {

    // Спецификация для запросов на чтение
    private static RequestSpecification getRequestSpec;

    // Спецификация для запросов на создание/обновление
    private static RequestSpecification postRequestSpec;

    // Спецификация для успешных ответов
    private static ResponseSpecification successResponseSpec;

    // Спецификация для ответа с созданным ресурсом
    private static ResponseSpecification createdResponseSpec;

    @BeforeAll
    static void setupSpecs() {
        // Базовая спецификация для GET-запросов
        getRequestSpec = new RequestSpecBuilder()
            .setBaseUri("https://jsonplaceholder.typicode.com")
            .log(LogDetail.URI)       // Логируем URI при отправке
            .build();

        // Спецификация для POST/PUT-запросов (с Content-Type)
        postRequestSpec = new RequestSpecBuilder()
            .setBaseUri("https://jsonplaceholder.typicode.com")
            .setContentType(ContentType.JSON)
            .log(LogDetail.ALL)       // Логируем всё: URI, headers, body
            .build();

        // Ожидаемый ответ: 200, JSON, время < 2 секунд
        successResponseSpec = new ResponseSpecBuilder()
            .expectStatusCode(200)
            .expectContentType(ContentType.JSON)
            .expectResponseTime(lessThan(2000L))
            .build();

        // Ожидаемый ответ: 201 Created
        createdResponseSpec = new ResponseSpecBuilder()
            .expectStatusCode(201)
            .expectContentType(ContentType.JSON)
            .build();
    }

    @Test
    void shouldGetAllPostsWithSpec() {
        // Используем спецификации — код чистый и читаемый
        given()
            .spec(getRequestSpec)
        .when()
            .get("/posts")
        .then()
            .spec(successResponseSpec)
            .body("size()", equalTo(100));
    }

    @Test
    void shouldGetSinglePostWithSpec() {
        given()
            .spec(getRequestSpec)
        .when()
            .get("/posts/1")
        .then()
            .spec(successResponseSpec)
            .body("id", equalTo(1))
            .body("userId", equalTo(1));
    }

    @Test
    void shouldCreatePostWithSpec() {
        String body = """
            {
                "title": "Тестовый пост со спецификацией",
                "body": "Используем RequestSpecification",
                "userId": 5
            }
            """;

        given()
            .spec(postRequestSpec)
            .body(body)
        .when()
            .post("/posts")
        .then()
            .spec(createdResponseSpec)
            .body("id", notNullValue())
            .body("userId", equalTo(5));
    }

    @Test
    void shouldFilterWithQueryParamsUsingSpec() {
        given()
            .spec(getRequestSpec)
            .queryParam("userId", 1)
        .when()
            .get("/posts")
        .then()
            .spec(successResponseSpec)
            .body("size()", equalTo(10))
            .body("userId", everyItem(equalTo(1)));
    }

    @Test
    void shouldUpdatePostWithSpec() {
        String body = """
            {
                "id": 1,
                "title": "Обновлённый заголовок",
                "body": "Обновлённое содержимое",
                "userId": 1
            }
            """;

        given()
            .spec(postRequestSpec)
            .body(body)
        .when()
            .put("/posts/1")
        .then()
            .spec(successResponseSpec)
            .body("title", equalTo("Обновлённый заголовок"));
    }
}
```

---

## Итоговое задание

Объедините все навыки из упражнений 1-6 и создайте комплексный тестовый класс:

### Требования

1. Используйте `RequestSpecification` для конфигурации (Упражнение 6)
2. Создайте POJO-модель `User` с полями: `id`, `name`, `username`, `email` (Упражнение 2)
3. Напишите параметризованный тест для 5 пользователей (Упражнение 3)
4. Добавьте JSON Schema для объекта User (Упражнение 5)
5. Покройте негативные сценарии: несуществующий пользователь, невалидный путь

### Критерии выполнения

| Критерий | Описание |
|----------|----------|
| Компиляция | Проект собирается без ошибок |
| Все тесты зелёные | `mvn test` — 100% pass rate |
| POJO используется | Десериализация в объект, проверка через assertEquals |
| Параметризация | Минимум 5 тестовых наборов через `@MethodSource` |
| Спецификации | `RequestSpecification` / `ResponseSpecification` вынесены |
| Схема | JSON Schema валидирует `/users/{id}` |
| Негативные тесты | Минимум 2 негативных сценария |

### Ожидаемый результат запуска

```
Tests run: 12, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

---

## Дополнительные ресурсы

- REST Assured документация: [https://rest-assured.io](https://rest-assured.io)
- REST Assured GitHub Wiki: [https://github.com/rest-assured/rest-assured/wiki/Usage](https://github.com/rest-assured/rest-assured/wiki/Usage)
- Hamcrest Matchers: [http://hamcrest.org/JavaHamcrest/javadoc/](http://hamcrest.org/JavaHamcrest/javadoc/)
- JSON Schema: [https://json-schema.org](https://json-schema.org)
