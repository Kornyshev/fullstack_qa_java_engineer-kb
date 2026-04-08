# Мини-проект: покрытие JSONPlaceholder API

## Содержание

1. [Описание проекта](#описание-проекта)
2. [Структура проекта](#структура-проекта)
3. [Maven-конфигурация](#maven-конфигурация)
4. [Конфигурация проекта](#конфигурация-проекта)
5. [POJO-модели](#pojo-модели)
6. [Базовый класс тестов](#базовый-класс-тестов)
7. [Тесты для /posts](#тесты-для-posts)
8. [Тесты для /users](#тесты-для-users)
9. [Тесты для /comments](#тесты-для-comments)
10. [Негативные сценарии](#негативные-сценарии)
11. [Allure-отчётность](#allure-отчётность)
12. [Запуск и анализ](#запуск-и-анализ)

---

## Описание проекта

Цель — создать полноценный API-тестовый проект с нуля, который покрывает основные эндпоинты JSONPlaceholder API. Проект можно использовать как шаблон для портфолио.

**Покрываемые эндпоинты:**
- `GET /posts`, `GET /posts/{id}`, `POST /posts`, `PUT /posts/{id}`, `DELETE /posts/{id}`
- `GET /users`, `GET /users/{id}`
- `GET /comments`, `GET /comments?postId={id}`

**Технологический стек:**
- Java 17
- Maven
- REST Assured 5.4.0
- JUnit 5
- Jackson (сериализация)
- Allure (отчёты)
- Owner (конфигурация)

---

## Структура проекта

```
jsonplaceholder-api-tests/
├── pom.xml
├── src/
│   ├── main/java/com/qa/
│   │   ├── config/
│   │   │   └── ApiConfig.java
│   │   └── models/
│   │       ├── Post.java
│   │       ├── User.java
│   │       ├── Comment.java
│   │       ├── Address.java
│   │       ├── Geo.java
│   │       └── Company.java
│   └── test/
│       ├── java/com/qa/tests/
│       │   ├── BaseTest.java
│       │   ├── PostsTest.java
│       │   ├── UsersTest.java
│       │   ├── CommentsTest.java
│       │   └── NegativeTest.java
│       └── resources/
│           ├── api.properties
│           └── schemas/
│               ├── post-schema.json
│               ├── user-schema.json
│               └── comment-schema.json
├── .gitignore
└── README.md
```

---

## Maven-конфигурация

### pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.qa</groupId>
    <artifactId>jsonplaceholder-api-tests</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>

        <rest-assured.version>5.4.0</rest-assured.version>
        <junit.version>5.10.2</junit.version>
        <jackson.version>2.17.0</jackson.version>
        <allure.version>2.25.0</allure.version>
        <aspectj.version>1.9.21</aspectj.version>
        <owner.version>1.0.12</owner.version>
    </properties>

    <dependencies>
        <!-- REST Assured -->
        <dependency>
            <groupId>io.rest-assured</groupId>
            <artifactId>rest-assured</artifactId>
            <version>${rest-assured.version}</version>
            <scope>test</scope>
        </dependency>
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

        <!-- Jackson -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>${jackson.version}</version>
        </dependency>

        <!-- Allure -->
        <dependency>
            <groupId>io.qameta.allure</groupId>
            <artifactId>allure-junit5</artifactId>
            <version>${allure.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.qameta.allure</groupId>
            <artifactId>allure-rest-assured</artifactId>
            <version>${allure.version}</version>
            <scope>test</scope>
        </dependency>

        <!-- Owner — конфигурация -->
        <dependency>
            <groupId>org.aeonbits.owner</groupId>
            <artifactId>owner</artifactId>
            <version>${owner.version}</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.2.5</version>
                <configuration>
                    <argLine>
                        -javaagent:"${settings.localRepository}/org/aspectj/aspectjweaver/${aspectj.version}/aspectjweaver-${aspectj.version}.jar"
                    </argLine>
                </configuration>
                <dependencies>
                    <dependency>
                        <groupId>org.aspectj</groupId>
                        <artifactId>aspectjweaver</artifactId>
                        <version>${aspectj.version}</version>
                    </dependency>
                </dependencies>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## Конфигурация проекта

### api.properties

```properties
# Базовый URL API
base.url=https://jsonplaceholder.typicode.com
# Таймаут подключения (миллисекунды)
connection.timeout=5000
# Таймаут чтения (миллисекунды)
read.timeout=5000
```

### ApiConfig.java

```java
package com.qa.config;

import org.aeonbits.owner.Config;
import org.aeonbits.owner.ConfigFactory;

@Config.Sources("classpath:api.properties")
public interface ApiConfig extends Config {

    @Key("base.url")
    String baseUrl();

    @Key("connection.timeout")
    @DefaultValue("5000")
    int connectionTimeout();

    @Key("read.timeout")
    @DefaultValue("5000")
    int readTimeout();

    // Фабричный метод для создания экземпляра конфигурации
    static ApiConfig getInstance() {
        return ConfigFactory.create(ApiConfig.class, System.getProperties());
    }
}
```

---

## POJO-модели

### Post.java

```java
package com.qa.models;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

@JsonIgnoreProperties(ignoreUnknown = true)
public class Post {

    private Integer userId;
    private Integer id;
    private String title;
    private String body;

    public Post() {}

    public Post(Integer userId, String title, String body) {
        this.userId = userId;
        this.title = title;
        this.body = body;
    }

    // Геттеры
    public Integer getUserId() { return userId; }
    public Integer getId() { return id; }
    public String getTitle() { return title; }
    public String getBody() { return body; }

    // Сеттеры
    public void setUserId(Integer userId) { this.userId = userId; }
    public void setId(Integer id) { this.id = id; }
    public void setTitle(String title) { this.title = title; }
    public void setBody(String body) { this.body = body; }
}
```

### User.java

```java
package com.qa.models;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

@JsonIgnoreProperties(ignoreUnknown = true)
public class User {

    private Integer id;
    private String name;
    private String username;
    private String email;
    private Address address;
    private String phone;
    private String website;
    private Company company;

    public User() {}

    // Геттеры
    public Integer getId() { return id; }
    public String getName() { return name; }
    public String getUsername() { return username; }
    public String getEmail() { return email; }
    public Address getAddress() { return address; }
    public String getPhone() { return phone; }
    public String getWebsite() { return website; }
    public Company getCompany() { return company; }

    // Сеттеры
    public void setId(Integer id) { this.id = id; }
    public void setName(String name) { this.name = name; }
    public void setUsername(String username) { this.username = username; }
    public void setEmail(String email) { this.email = email; }
    public void setAddress(Address address) { this.address = address; }
    public void setPhone(String phone) { this.phone = phone; }
    public void setWebsite(String website) { this.website = website; }
    public void setCompany(Company company) { this.company = company; }
}
```

### Comment.java

```java
package com.qa.models;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

@JsonIgnoreProperties(ignoreUnknown = true)
public class Comment {

    private Integer postId;
    private Integer id;
    private String name;
    private String email;
    private String body;

    public Comment() {}

    // Геттеры
    public Integer getPostId() { return postId; }
    public Integer getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    public String getBody() { return body; }

    // Сеттеры
    public void setPostId(Integer postId) { this.postId = postId; }
    public void setId(Integer id) { this.id = id; }
    public void setName(String name) { this.name = name; }
    public void setEmail(String email) { this.email = email; }
    public void setBody(String body) { this.body = body; }
}
```

### Address.java и вспомогательные классы

```java
package com.qa.models;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

@JsonIgnoreProperties(ignoreUnknown = true)
public class Address {
    private String street;
    private String suite;
    private String city;
    private String zipcode;
    private Geo geo;

    // Геттеры и сеттеры
    public String getStreet() { return street; }
    public void setStreet(String street) { this.street = street; }
    public String getSuite() { return suite; }
    public void setSuite(String suite) { this.suite = suite; }
    public String getCity() { return city; }
    public void setCity(String city) { this.city = city; }
    public String getZipcode() { return zipcode; }
    public void setZipcode(String zipcode) { this.zipcode = zipcode; }
    public Geo getGeo() { return geo; }
    public void setGeo(Geo geo) { this.geo = geo; }
}
```

```java
package com.qa.models;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

@JsonIgnoreProperties(ignoreUnknown = true)
public class Geo {
    private String lat;
    private String lng;

    public String getLat() { return lat; }
    public void setLat(String lat) { this.lat = lat; }
    public String getLng() { return lng; }
    public void setLng(String lng) { this.lng = lng; }
}
```

```java
package com.qa.models;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

@JsonIgnoreProperties(ignoreUnknown = true)
public class Company {
    private String name;
    private String catchPhrase;
    private String bs;

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getCatchPhrase() { return catchPhrase; }
    public void setCatchPhrase(String catchPhrase) { this.catchPhrase = catchPhrase; }
    public String getBs() { return bs; }
    public void setBs(String bs) { this.bs = bs; }
}
```

---

## Базовый класс тестов

### BaseTest.java

```java
package com.qa.tests;

import com.qa.config.ApiConfig;
import io.qameta.allure.restassured.AllureRestAssured;
import io.restassured.builder.RequestSpecBuilder;
import io.restassured.builder.ResponseSpecBuilder;
import io.restassured.filter.log.LogDetail;
import io.restassured.http.ContentType;
import io.restassured.specification.RequestSpecification;
import io.restassured.specification.ResponseSpecification;
import org.junit.jupiter.api.BeforeAll;

import static org.hamcrest.Matchers.lessThan;

public abstract class BaseTest {

    protected static ApiConfig config;
    protected static RequestSpecification baseRequestSpec;
    protected static RequestSpecification jsonRequestSpec;
    protected static ResponseSpecification okResponseSpec;
    protected static ResponseSpecification createdResponseSpec;

    @BeforeAll
    static void setupBase() {
        // Загружаем конфигурацию
        config = ApiConfig.getInstance();

        // Спецификация для GET-запросов
        baseRequestSpec = new RequestSpecBuilder()
            .setBaseUri(config.baseUrl())
            .addFilter(new AllureRestAssured())  // Логирование в Allure
            .log(LogDetail.URI)
            .build();

        // Спецификация для POST/PUT-запросов с JSON-телом
        jsonRequestSpec = new RequestSpecBuilder()
            .setBaseUri(config.baseUrl())
            .setContentType(ContentType.JSON)
            .addFilter(new AllureRestAssured())
            .log(LogDetail.ALL)
            .build();

        // Ожидаемый ответ: 200 OK
        okResponseSpec = new ResponseSpecBuilder()
            .expectStatusCode(200)
            .expectContentType(ContentType.JSON)
            .expectResponseTime(lessThan(5000L))
            .build();

        // Ожидаемый ответ: 201 Created
        createdResponseSpec = new ResponseSpecBuilder()
            .expectStatusCode(201)
            .expectContentType(ContentType.JSON)
            .build();
    }
}
```

---

## Тесты для /posts

### PostsTest.java

```java
package com.qa.tests;

import com.qa.models.Post;
import io.qameta.allure.*;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;

import static io.restassured.RestAssured.given;
import static io.restassured.module.jsv.JsonSchemaValidator.matchesJsonSchemaInClasspath;
import static org.hamcrest.Matchers.*;
import static org.junit.jupiter.api.Assertions.*;

@Epic("API-тесты JSONPlaceholder")
@Feature("Posts")
public class PostsTest extends BaseTest {

    @Test
    @Story("Получение списка постов")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("GET /posts — получить все посты")
    void shouldReturnAllPosts() {
        Post[] posts = given()
            .spec(baseRequestSpec)
        .when()
            .get("/posts")
        .then()
            .spec(okResponseSpec)
            .body("size()", equalTo(100))
            .extract()
            .as(Post[].class);

        assertEquals(100, posts.length, "Должно быть 100 постов");
    }

    @ParameterizedTest(name = "GET /posts/{0}")
    @ValueSource(ints = {1, 50, 100})
    @Story("Получение поста по ID")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("GET /posts/{id} — получить пост по ID")
    void shouldReturnPostById(int postId) {
        Post post = given()
            .spec(baseRequestSpec)
        .when()
            .get("/posts/" + postId)
        .then()
            .spec(okResponseSpec)
            .body(matchesJsonSchemaInClasspath("schemas/post-schema.json"))
            .extract()
            .as(Post.class);

        assertAll("Проверка поста #" + postId,
            () -> assertEquals(postId, post.getId()),
            () -> assertNotNull(post.getUserId()),
            () -> assertFalse(post.getTitle().isEmpty()),
            () -> assertFalse(post.getBody().isEmpty())
        );
    }

    @Test
    @Story("Создание поста")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("POST /posts — создать новый пост")
    void shouldCreatePost() {
        Post newPost = new Post(1, "Заголовок нового поста", "Тело нового поста");

        Post created = given()
            .spec(jsonRequestSpec)
            .body(newPost)
        .when()
            .post("/posts")
        .then()
            .spec(createdResponseSpec)
            .extract()
            .as(Post.class);

        assertAll("Проверка созданного поста",
            () -> assertNotNull(created.getId(), "Должен вернуться id"),
            () -> assertEquals(newPost.getUserId(), created.getUserId()),
            () -> assertEquals(newPost.getTitle(), created.getTitle()),
            () -> assertEquals(newPost.getBody(), created.getBody())
        );
    }

    @Test
    @Story("Обновление поста")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("PUT /posts/1 — полное обновление поста")
    void shouldUpdatePost() {
        Post updated = new Post(1, "Обновлённый заголовок", "Обновлённое тело");
        updated.setId(1);

        given()
            .spec(jsonRequestSpec)
            .body(updated)
        .when()
            .put("/posts/1")
        .then()
            .spec(okResponseSpec)
            .body("title", equalTo("Обновлённый заголовок"))
            .body("body", equalTo("Обновлённое тело"));
    }

    @Test
    @Story("Частичное обновление поста")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("PATCH /posts/1 — частичное обновление")
    void shouldPatchPost() {
        String patchBody = """
            {
                "title": "Только заголовок изменён"
            }
            """;

        given()
            .spec(jsonRequestSpec)
            .body(patchBody)
        .when()
            .patch("/posts/1")
        .then()
            .spec(okResponseSpec)
            .body("title", equalTo("Только заголовок изменён"))
            .body("id", equalTo(1));
    }

    @Test
    @Story("Удаление поста")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("DELETE /posts/1 — удалить пост")
    void shouldDeletePost() {
        given()
            .spec(baseRequestSpec)
        .when()
            .delete("/posts/1")
        .then()
            .statusCode(200);
    }

    @Test
    @Story("Фильтрация постов")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("GET /posts?userId=1 — фильтрация по userId")
    void shouldFilterPostsByUserId() {
        Post[] posts = given()
            .spec(baseRequestSpec)
            .queryParam("userId", 1)
        .when()
            .get("/posts")
        .then()
            .spec(okResponseSpec)
            .body("size()", equalTo(10))
            .body("userId", everyItem(equalTo(1)))
            .extract()
            .as(Post[].class);

        for (Post post : posts) {
            assertEquals(1, post.getUserId(), "Все посты должны принадлежать userId=1");
        }
    }
}
```

---

## Тесты для /users

### UsersTest.java

```java
package com.qa.tests;

import com.qa.models.User;
import io.qameta.allure.*;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.Arguments;
import org.junit.jupiter.params.provider.MethodSource;

import java.util.stream.Stream;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;
import static org.junit.jupiter.api.Assertions.*;

@Epic("API-тесты JSONPlaceholder")
@Feature("Users")
public class UsersTest extends BaseTest {

    static Stream<Arguments> provideUsers() {
        return Stream.of(
            Arguments.of(1, "Leanne Graham", "Bret", "Sincere@april.biz"),
            Arguments.of(2, "Ervin Howell", "Antonette", "Shanna@melissa.tv"),
            Arguments.of(3, "Clementine Bauch", "Samantha", "Nathan@yesenia.net"),
            Arguments.of(5, "Chelsey Dietrich", "Kamren", "Lucio_Hettinger@annie.ca"),
            Arguments.of(10, "Clementina DuBuque", "Moriah.Stanton", "Rey.Padberg@karina.biz")
        );
    }

    @Test
    @Story("Получение списка пользователей")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("GET /users — получить всех пользователей")
    void shouldReturnAllUsers() {
        User[] users = given()
            .spec(baseRequestSpec)
        .when()
            .get("/users")
        .then()
            .spec(okResponseSpec)
            .body("size()", equalTo(10))
            .extract()
            .as(User[].class);

        assertEquals(10, users.length);

        // Проверяем, что у всех пользователей заполнены обязательные поля
        for (User user : users) {
            assertAll("Проверка пользователя #" + user.getId(),
                () -> assertNotNull(user.getId()),
                () -> assertNotNull(user.getName()),
                () -> assertNotNull(user.getEmail()),
                () -> assertTrue(user.getEmail().contains("@"), "Email должен содержать @")
            );
        }
    }

    @ParameterizedTest(name = "Пользователь #{0}: {1}")
    @MethodSource("provideUsers")
    @Story("Получение пользователя по ID")
    @Severity(SeverityLevel.CRITICAL)
    void shouldReturnUserById(int id, String name, String username, String email) {
        User user = given()
            .spec(baseRequestSpec)
        .when()
            .get("/users/" + id)
        .then()
            .spec(okResponseSpec)
            .extract()
            .as(User.class);

        assertAll("Проверка пользователя #" + id,
            () -> assertEquals(id, user.getId()),
            () -> assertEquals(name, user.getName()),
            () -> assertEquals(username, user.getUsername()),
            () -> assertEquals(email, user.getEmail())
        );
    }

    @Test
    @Story("Вложенные объекты пользователя")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("GET /users/1 — проверка адреса и компании")
    void shouldReturnUserWithNestedObjects() {
        given()
            .spec(baseRequestSpec)
        .when()
            .get("/users/1")
        .then()
            .spec(okResponseSpec)
            .body("address.city", not(emptyString()))
            .body("address.geo.lat", notNullValue())
            .body("address.geo.lng", notNullValue())
            .body("company.name", not(emptyString()))
            .body("company.catchPhrase", notNullValue());
    }
}
```

---

## Тесты для /comments

### CommentsTest.java

```java
package com.qa.tests;

import com.qa.models.Comment;
import io.qameta.allure.*;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;
import static org.junit.jupiter.api.Assertions.*;

@Epic("API-тесты JSONPlaceholder")
@Feature("Comments")
public class CommentsTest extends BaseTest {

    @Test
    @Story("Получение комментариев")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("GET /comments — получить все комментарии")
    void shouldReturnAllComments() {
        given()
            .spec(baseRequestSpec)
        .when()
            .get("/comments")
        .then()
            .spec(okResponseSpec)
            .body("size()", equalTo(500));
    }

    @Test
    @Story("Фильтрация комментариев")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("GET /comments?postId=1 — комментарии к посту")
    void shouldReturnCommentsByPostId() {
        Comment[] comments = given()
            .spec(baseRequestSpec)
            .queryParam("postId", 1)
        .when()
            .get("/comments")
        .then()
            .spec(okResponseSpec)
            .body("size()", equalTo(5))
            .body("postId", everyItem(equalTo(1)))
            .extract()
            .as(Comment[].class);

        for (Comment comment : comments) {
            assertAll("Комментарий #" + comment.getId(),
                () -> assertEquals(1, comment.getPostId()),
                () -> assertNotNull(comment.getEmail()),
                () -> assertTrue(comment.getEmail().contains("@")),
                () -> assertFalse(comment.getBody().isEmpty())
            );
        }
    }

    @Test
    @Story("Вложенные ресурсы")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("GET /posts/1/comments — комментарии через вложенный ресурс")
    void shouldReturnCommentsAsNestedResource() {
        // Оба способа должны возвращать одинаковые данные
        Comment[] viaQuery = given()
            .spec(baseRequestSpec)
            .queryParam("postId", 1)
        .when()
            .get("/comments")
        .then()
            .extract()
            .as(Comment[].class);

        Comment[] viaNested = given()
            .spec(baseRequestSpec)
        .when()
            .get("/posts/1/comments")
        .then()
            .extract()
            .as(Comment[].class);

        assertEquals(viaQuery.length, viaNested.length,
            "Оба способа должны возвращать одинаковое количество комментариев");
    }

    @Test
    @Story("Валидация email")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("Все комментарии содержат валидный email")
    void allCommentsShouldHaveValidEmail() {
        given()
            .spec(baseRequestSpec)
        .when()
            .get("/comments")
        .then()
            .spec(okResponseSpec)
            .body("email", everyItem(containsString("@")));
    }
}
```

---

## Негативные сценарии

### NegativeTest.java

```java
package com.qa.tests;

import io.qameta.allure.*;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@Epic("API-тесты JSONPlaceholder")
@Feature("Негативные сценарии")
public class NegativeTest extends BaseTest {

    @ParameterizedTest(name = "GET /posts/{0} — несуществующий пост")
    @ValueSource(ints = {0, -1, 999, 10000})
    @Story("404 для несуществующих ресурсов")
    @Severity(SeverityLevel.NORMAL)
    void shouldReturn404ForNonExistentPost(int postId) {
        given()
            .spec(baseRequestSpec)
        .when()
            .get("/posts/" + postId)
        .then()
            .statusCode(404);
    }

    @Test
    @Story("404 для несуществующих ресурсов")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("GET /users/999 — несуществующий пользователь")
    void shouldReturn404ForNonExistentUser() {
        given()
            .spec(baseRequestSpec)
        .when()
            .get("/users/999")
        .then()
            .statusCode(404);
    }

    @Test
    @Story("Невалидные данные")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("POST /posts — пустое тело запроса")
    void shouldHandleEmptyBody() {
        given()
            .spec(jsonRequestSpec)
            .body("{}")
        .when()
            .post("/posts")
        .then()
            .statusCode(201)   // JSONPlaceholder принимает пустое тело
            .body("id", notNullValue());
    }

    @Test
    @Story("Невалидный эндпоинт")
    @Severity(SeverityLevel.MINOR)
    @DisplayName("GET /nonexistent — несуществующий путь")
    void shouldReturn404ForInvalidEndpoint() {
        given()
            .spec(baseRequestSpec)
        .when()
            .get("/nonexistent")
        .then()
            .statusCode(404);
    }

    @Test
    @Story("Фильтрация несуществующих данных")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("GET /posts?userId=999 — пустой результат фильтрации")
    void shouldReturnEmptyArrayForNonExistentUserId() {
        given()
            .spec(baseRequestSpec)
            .queryParam("userId", 999)
        .when()
            .get("/posts")
        .then()
            .statusCode(200)
            .body("size()", equalTo(0));  // Пустой массив
    }

    @Test
    @Story("Невалидный Content-Type")
    @Severity(SeverityLevel.MINOR)
    @DisplayName("POST /posts — отправка данных в формате plain text")
    void shouldHandlePlainTextBody() {
        given()
            .spec(baseRequestSpec)
            .contentType("text/plain")
            .body("Это простой текст, не JSON")
        .when()
            .post("/posts")
        .then()
            .statusCode(anyOf(is(200), is(201)));
    }
}
```

---

## Allure-отчётность

### Генерация отчёта

```bash
# Запуск тестов
mvn clean test

# Генерация Allure-отчёта
mvn allure:serve
```

### Ожидаемая структура отчёта

В Allure-отчёте вы увидите:

1. **Overview** — общая статистика: сколько тестов прошло, упало, пропущено
2. **Suites** — группировка по тестовым классам
3. **Behaviors** — группировка по Epic → Feature → Story (из аннотаций)
4. **Graphs** — графики: продолжительность, статусы, severity

Каждый тест будет содержать:
- Полный запрос (URL, headers, body) — благодаря `AllureRestAssured` фильтру
- Полный ответ (status, headers, body)
- Severity и описание из аннотаций

---

## Запуск и анализ

### Команды запуска

```bash
# Все тесты
mvn clean test

# Только Posts-тесты
mvn clean test -Dtest=PostsTest

# Только негативные тесты
mvn clean test -Dtest=NegativeTest

# С генерацией Allure-отчёта
mvn clean test && mvn allure:serve
```

### Ожидаемый результат

```
[INFO] Tests run: 25, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

### .gitignore

```
target/
.idea/
*.iml
allure-results/
allure-report/
.DS_Store
```

### README.md для репозитория

Включите в README следующие разделы:
- Описание проекта и стека технологий
- Инструкция по запуску: `mvn clean test`
- Как посмотреть отчёт: `mvn allure:serve`
- Структура проекта
- Список покрытых эндпоинтов и сценариев
- Скриншот Allure-отчёта

---

## Дополнительные задания

1. **Расширение покрытия**: Добавьте тесты для эндпоинтов `/albums`, `/photos`, `/todos`
2. **POJO для всех моделей**: Создайте POJO-классы для Album, Photo, Todo
3. **JSON Schema**: Создайте JSON Schema для каждого ресурса и валидируйте ответы
4. **CI/CD**: Добавьте GitHub Actions workflow для запуска тестов при push
5. **Retry**: Добавьте механизм повторного запуска упавших тестов
