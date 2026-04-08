# Этап 2: API-автоматизация Conduit

## Обзор

На предыдущем этапе мы составили тест-план и тест-кейсы вручную. Теперь переходим
к автоматизации API-тестов. REST Assured позволяет отправлять HTTP-запросы, валидировать
ответы и интегрироваться с Allure для красивой отчётности. На выходе вы получите полноценный
Java-проект с покрытием всех API-эндпоинтов Conduit.

> **Цель:** Создать comprehensive набор API-тестов для Conduit REST API, используя REST Assured,
> POJO-модели, JSON Schema валидацию и Allure-аннотации.

---

## Предварительные требования

Перед началом убедитесь, что:
- Conduit запущен локально (API доступен на `http://localhost:3000/api`)
- Установлены JDK 17+, Maven 3.9+
- Создан Maven-проект с базовой структурой из Этапа 0 (01_project_overview.md)
- Выполнен этап ручного тестирования (тест-кейсы готовы)

---

## Часть 1: Структура проекта для API-тестов

### 1.1 Целевая структура пакетов

```
src/test/java/
├── api/
│   ├── client/                    # HTTP-клиенты для каждой группы эндпоинтов
│   │   ├── ApiClient.java         # Базовый клиент с общей конфигурацией
│   │   ├── AuthApi.java           # Эндпоинты аутентификации
│   │   ├── ArticleApi.java        # Эндпоинты статей
│   │   ├── CommentApi.java        # Эндпоинты комментариев
│   │   ├── ProfileApi.java        # Эндпоинты профилей
│   │   ├── TagApi.java            # Эндпоинты тегов
│   │   └── FavoriteApi.java       # Эндпоинты избранного
│   ├── models/                    # POJO-модели
│   │   ├── request/               # Тела запросов
│   │   │   ├── LoginRequest.java
│   │   │   ├── RegisterRequest.java
│   │   │   ├── ArticleRequest.java
│   │   │   ├── CommentRequest.java
│   │   │   └── UpdateUserRequest.java
│   │   └── response/              # Тела ответов
│   │       ├── UserResponse.java
│   │       ├── ArticleResponse.java
│   │       ├── ArticlesListResponse.java
│   │       ├── CommentResponse.java
│   │       ├── ProfileResponse.java
│   │       ├── TagsResponse.java
│   │       └── ErrorResponse.java
│   ├── specs/                     # RequestSpecification / ResponseSpecification
│   │   ├── RequestSpecs.java
│   │   └── ResponseSpecs.java
│   └── tests/                     # Тестовые классы
│       ├── BaseApiTest.java
│       ├── AuthApiTest.java
│       ├── ArticleApiTest.java
│       ├── CommentApiTest.java
│       ├── ProfileApiTest.java
│       ├── TagApiTest.java
│       └── FavoriteApiTest.java
├── config/
│   └── ConfigReader.java          # Чтение конфигурации
└── utils/
    └── DataGenerator.java         # Генерация тестовых данных

src/test/resources/
├── config.properties              # Базовый URL, креды
├── json-schemas/                  # JSON Schema файлы
│   ├── user-response.json
│   ├── article-response.json
│   ├── articles-list-response.json
│   ├── comment-response.json
│   └── error-response.json
└── allure.properties              # Настройки Allure
```

### 1.2 Зависимости в pom.xml

```xml
<dependencies>
    <!-- REST Assured: HTTP-клиент для тестирования API -->
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>rest-assured</artifactId>
        <version>5.4.0</version>
        <scope>test</scope>
    </dependency>

    <!-- REST Assured: валидация JSON Schema -->
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>json-schema-validator</artifactId>
        <version>5.4.0</version>
        <scope>test</scope>
    </dependency>

    <!-- JUnit 5 -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.10.2</version>
        <scope>test</scope>
    </dependency>

    <!-- AssertJ: читаемые assertion'ы -->
    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <version>3.25.3</version>
        <scope>test</scope>
    </dependency>

    <!-- Jackson: сериализация/десериализация JSON <-> POJO -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.17.0</version>
    </dependency>

    <!-- JavaFaker: генерация тестовых данных -->
    <dependency>
        <groupId>net.datafaker</groupId>
        <artifactId>datafaker</artifactId>
        <version>2.1.0</version>
        <scope>test</scope>
    </dependency>

    <!-- Allure JUnit 5 -->
    <dependency>
        <groupId>io.qameta.allure</groupId>
        <artifactId>allure-junit5</artifactId>
        <version>2.25.0</version>
        <scope>test</scope>
    </dependency>

    <!-- Allure REST Assured: автоматическое логирование запросов/ответов -->
    <dependency>
        <groupId>io.qameta.allure</groupId>
        <artifactId>allure-rest-assured</artifactId>
        <version>2.25.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

## Часть 2: Конфигурация

### 2.1 Файл config.properties

```properties
# Базовый URL API
base.url=http://localhost:3000/api

# Тестовый пользователь (создаётся заранее или в setup-методе)
test.user.email=testuser@example.com
test.user.password=TestPass123!
test.user.username=testuser
```

### 2.2 Класс ConfigReader

```java
package config;

import java.io.IOException;
import java.io.InputStream;
import java.util.Properties;

/**
 * Читает конфигурацию из properties-файлов.
 * Поддерживает переопределение через переменные окружения (для CI/CD).
 */
public class ConfigReader {

    private static final Properties properties = new Properties();

    static {
        // Загружаем properties-файл при инициализации класса
        try (InputStream input = ConfigReader.class.getClassLoader()
                .getResourceAsStream("config.properties")) {
            if (input != null) {
                properties.load(input);
            }
        } catch (IOException e) {
            throw new RuntimeException("Не удалось загрузить config.properties", e);
        }
    }

    /**
     * Возвращает значение по ключу.
     * Приоритет: переменная окружения -> системное свойство -> properties-файл.
     */
    public static String get(String key) {
        // Сначала проверяем переменную окружения (для CI)
        String envValue = System.getenv(key.replace(".", "_").toUpperCase());
        if (envValue != null) return envValue;

        // Затем системное свойство (для -D параметров Maven)
        String sysValue = System.getProperty(key);
        if (sysValue != null) return sysValue;

        // Наконец, значение из properties-файла
        return properties.getProperty(key);
    }

    public static String getBaseUrl() {
        return get("base.url");
    }
}
```

### 2.3 Генератор тестовых данных

```java
package utils;

import net.datafaker.Faker;

/**
 * Генерирует уникальные тестовые данные с помощью Faker.
 * Каждый вызов возвращает новые значения, что обеспечивает
 * изоляцию между тестами.
 */
public class DataGenerator {

    private static final Faker faker = new Faker();

    public static String randomUsername() {
        // Добавляем timestamp для гарантии уникальности
        return faker.internet().username() + System.currentTimeMillis();
    }

    public static String randomEmail() {
        return "test_" + System.currentTimeMillis() + "@example.com";
    }

    public static String randomPassword() {
        return faker.internet().password(8, 20, true, true);
    }

    public static String randomTitle() {
        return faker.lorem().sentence(5);
    }

    public static String randomDescription() {
        return faker.lorem().sentence(10);
    }

    public static String randomBody() {
        return faker.lorem().paragraph(3);
    }

    public static String randomTag() {
        return faker.programmingLanguage().name().toLowerCase().replaceAll("\\s+", "-");
    }

    public static String randomComment() {
        return faker.lorem().sentence(8);
    }
}
```

---

## Часть 3: POJO-модели

### 3.1 Модели запросов (request)

```java
package api.models.request;

import com.fasterxml.jackson.annotation.JsonInclude;

/**
 * Обёртка для запроса регистрации.
 * JSON-формат: {"user": {"username": "...", "email": "...", "password": "..."}}
 */
@JsonInclude(JsonInclude.Include.NON_NULL)
public class RegisterRequest {

    private User user;

    public RegisterRequest(String username, String email, String password) {
        this.user = new User(username, email, password);
    }

    // Getter и setter
    public User getUser() { return user; }
    public void setUser(User user) { this.user = user; }

    @JsonInclude(JsonInclude.Include.NON_NULL)
    public static class User {
        private String username;
        private String email;
        private String password;

        public User(String username, String email, String password) {
            this.username = username;
            this.email = email;
            this.password = password;
        }

        // Getters и setters
        public String getUsername() { return username; }
        public void setUsername(String username) { this.username = username; }
        public String getEmail() { return email; }
        public void setEmail(String email) { this.email = email; }
        public String getPassword() { return password; }
        public void setPassword(String password) { this.password = password; }
    }
}
```

```java
package api.models.request;

import com.fasterxml.jackson.annotation.JsonInclude;

/**
 * Запрос на авторизацию.
 * JSON: {"user": {"email": "...", "password": "..."}}
 */
@JsonInclude(JsonInclude.Include.NON_NULL)
public class LoginRequest {

    private User user;

    public LoginRequest(String email, String password) {
        this.user = new User(email, password);
    }

    public User getUser() { return user; }
    public void setUser(User user) { this.user = user; }

    public static class User {
        private String email;
        private String password;

        public User(String email, String password) {
            this.email = email;
            this.password = password;
        }

        public String getEmail() { return email; }
        public void setEmail(String email) { this.email = email; }
        public String getPassword() { return password; }
        public void setPassword(String password) { this.password = password; }
    }
}
```

```java
package api.models.request;

import com.fasterxml.jackson.annotation.JsonInclude;
import java.util.List;

/**
 * Запрос на создание/обновление статьи.
 * JSON: {"article": {"title": "...", "description": "...", "body": "...", "tagList": [...]}}
 */
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ArticleRequest {

    private Article article;

    public ArticleRequest(String title, String description, String body, List<String> tagList) {
        this.article = new Article(title, description, body, tagList);
    }

    public Article getArticle() { return article; }
    public void setArticle(Article article) { this.article = article; }

    @JsonInclude(JsonInclude.Include.NON_NULL)
    public static class Article {
        private String title;
        private String description;
        private String body;
        private List<String> tagList;

        public Article(String title, String description, String body, List<String> tagList) {
            this.title = title;
            this.description = description;
            this.body = body;
            this.tagList = tagList;
        }

        // Getters и setters
        public String getTitle() { return title; }
        public void setTitle(String title) { this.title = title; }
        public String getDescription() { return description; }
        public void setDescription(String description) { this.description = description; }
        public String getBody() { return body; }
        public void setBody(String body) { this.body = body; }
        public List<String> getTagList() { return tagList; }
        public void setTagList(List<String> tagList) { this.tagList = tagList; }
    }
}
```

```java
package api.models.request;

import com.fasterxml.jackson.annotation.JsonInclude;

/**
 * Запрос на добавление комментария.
 * JSON: {"comment": {"body": "..."}}
 */
@JsonInclude(JsonInclude.Include.NON_NULL)
public class CommentRequest {

    private Comment comment;

    public CommentRequest(String body) {
        this.comment = new Comment(body);
    }

    public Comment getComment() { return comment; }
    public void setComment(Comment comment) { this.comment = comment; }

    public static class Comment {
        private String body;

        public Comment(String body) { this.body = body; }

        public String getBody() { return body; }
        public void setBody(String body) { this.body = body; }
    }
}
```

### 3.2 Модели ответов (response)

```java
package api.models.response;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

/**
 * Ответ с данными пользователя.
 * JSON: {"user": {"email": "...", "token": "...", "username": "...", "bio": ..., "image": ...}}
 */
@JsonIgnoreProperties(ignoreUnknown = true)
public class UserResponse {

    private User user;

    public User getUser() { return user; }
    public void setUser(User user) { this.user = user; }

    @JsonIgnoreProperties(ignoreUnknown = true)
    public static class User {
        private String email;
        private String token;
        private String username;
        private String bio;
        private String image;

        // Getters и setters
        public String getEmail() { return email; }
        public void setEmail(String email) { this.email = email; }
        public String getToken() { return token; }
        public void setToken(String token) { this.token = token; }
        public String getUsername() { return username; }
        public void setUsername(String username) { this.username = username; }
        public String getBio() { return bio; }
        public void setBio(String bio) { this.bio = bio; }
        public String getImage() { return image; }
        public void setImage(String image) { this.image = image; }
    }
}
```

```java
package api.models.response;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import java.util.List;

/**
 * Ответ с данными статьи.
 * JSON: {"article": {"slug": "...", "title": "...", ...}}
 */
@JsonIgnoreProperties(ignoreUnknown = true)
public class ArticleResponse {

    private Article article;

    public Article getArticle() { return article; }
    public void setArticle(Article article) { this.article = article; }

    @JsonIgnoreProperties(ignoreUnknown = true)
    public static class Article {
        private String slug;
        private String title;
        private String description;
        private String body;
        private List<String> tagList;
        private String createdAt;
        private String updatedAt;
        private boolean favorited;
        private int favoritesCount;
        private Author author;

        // Getters (для краткости — только getters, setters аналогичны)
        public String getSlug() { return slug; }
        public String getTitle() { return title; }
        public String getDescription() { return description; }
        public String getBody() { return body; }
        public List<String> getTagList() { return tagList; }
        public String getCreatedAt() { return createdAt; }
        public String getUpdatedAt() { return updatedAt; }
        public boolean isFavorited() { return favorited; }
        public int getFavoritesCount() { return favoritesCount; }
        public Author getAuthor() { return author; }

        // Setters
        public void setSlug(String slug) { this.slug = slug; }
        public void setTitle(String title) { this.title = title; }
        public void setDescription(String description) { this.description = description; }
        public void setBody(String body) { this.body = body; }
        public void setTagList(List<String> tagList) { this.tagList = tagList; }
        public void setCreatedAt(String createdAt) { this.createdAt = createdAt; }
        public void setUpdatedAt(String updatedAt) { this.updatedAt = updatedAt; }
        public void setFavorited(boolean favorited) { this.favorited = favorited; }
        public void setFavoritesCount(int favoritesCount) { this.favoritesCount = favoritesCount; }
        public void setAuthor(Author author) { this.author = author; }
    }

    @JsonIgnoreProperties(ignoreUnknown = true)
    public static class Author {
        private String username;
        private String bio;
        private String image;
        private boolean following;

        public String getUsername() { return username; }
        public void setUsername(String username) { this.username = username; }
        public String getBio() { return bio; }
        public void setBio(String bio) { this.bio = bio; }
        public String getImage() { return image; }
        public void setImage(String image) { this.image = image; }
        public boolean isFollowing() { return following; }
        public void setFollowing(boolean following) { this.following = following; }
    }
}
```

```java
package api.models.response;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import java.util.List;
import java.util.Map;

/**
 * Ответ с ошибками валидации.
 * JSON: {"errors": {"body": ["can't be blank"], "title": ["can't be blank"]}}
 */
@JsonIgnoreProperties(ignoreUnknown = true)
public class ErrorResponse {

    private Map<String, List<String>> errors;

    public Map<String, List<String>> getErrors() { return errors; }
    public void setErrors(Map<String, List<String>> errors) { this.errors = errors; }
}
```

---

## Часть 4: Спецификации REST Assured (specs)

### 4.1 Request Specifications

```java
package api.specs;

import config.ConfigReader;
import io.qameta.allure.restassured.AllureRestAssured;
import io.restassured.builder.RequestSpecBuilder;
import io.restassured.http.ContentType;
import io.restassured.specification.RequestSpecification;

/**
 * Централизованные RequestSpecification для всех API-тестов.
 * Содержат базовый URL, content type и Allure-фильтр.
 */
public class RequestSpecs {

    /**
     * Базовая спецификация без авторизации.
     * Используется для эндпоинтов регистрации и логина.
     */
    public static RequestSpecification unauthenticated() {
        return new RequestSpecBuilder()
                .setBaseUri(ConfigReader.getBaseUrl())
                .setContentType(ContentType.JSON)
                .addFilter(new AllureRestAssured())   // Логирует запросы/ответы в Allure
                .build();
    }

    /**
     * Спецификация с JWT-токеном.
     * Используется для всех защищённых эндпоинтов.
     */
    public static RequestSpecification authenticated(String token) {
        return new RequestSpecBuilder()
                .setBaseUri(ConfigReader.getBaseUrl())
                .setContentType(ContentType.JSON)
                .addHeader("Authorization", "Token " + token)
                .addFilter(new AllureRestAssured())
                .build();
    }
}
```

### 4.2 Response Specifications

```java
package api.specs;

import io.restassured.builder.ResponseSpecBuilder;
import io.restassured.specification.ResponseSpecification;

import static org.hamcrest.Matchers.lessThan;

/**
 * Общие ResponseSpecification — переиспользуемые проверки ответов.
 */
public class ResponseSpecs {

    /** Успешный ответ: статус 200, время ответа < 5 секунд */
    public static ResponseSpecification success() {
        return new ResponseSpecBuilder()
                .expectStatusCode(200)
                .expectResponseTime(lessThan(5000L))
                .build();
    }

    /** Ответ о создании ресурса: статус 200 (Conduit возвращает 200, не 201) */
    public static ResponseSpecification created() {
        return new ResponseSpecBuilder()
                .expectStatusCode(200)
                .build();
    }

    /** Ответ с ошибкой валидации */
    public static ResponseSpecification validationError() {
        return new ResponseSpecBuilder()
                .expectStatusCode(422)
                .build();
    }

    /** Ответ "не авторизован" */
    public static ResponseSpecification unauthorized() {
        return new ResponseSpecBuilder()
                .expectStatusCode(401)
                .build();
    }

    /** Ответ "не найдено" */
    public static ResponseSpecification notFound() {
        return new ResponseSpecBuilder()
                .expectStatusCode(404)
                .build();
    }
}
```

---

## Часть 5: API-клиенты

### 5.1 AuthApi -- аутентификация

```java
package api.client;

import api.models.request.LoginRequest;
import api.models.request.RegisterRequest;
import api.models.response.UserResponse;
import api.specs.RequestSpecs;
import io.qameta.allure.Step;
import io.restassured.response.Response;

import static io.restassured.RestAssured.given;

/**
 * Клиент для эндпоинтов аутентификации: регистрация и логин.
 */
public class AuthApi {

    @Step("Регистрация пользователя: username={request.user.username}, email={request.user.email}")
    public static Response register(RegisterRequest request) {
        return given()
                .spec(RequestSpecs.unauthenticated())
                .body(request)
                .when()
                .post("/users");
    }

    @Step("Авторизация пользователя: email={request.user.email}")
    public static Response login(LoginRequest request) {
        return given()
                .spec(RequestSpecs.unauthenticated())
                .body(request)
                .when()
                .post("/users/login");
    }

    @Step("Получение текущего пользователя")
    public static Response getCurrentUser(String token) {
        return given()
                .spec(RequestSpecs.authenticated(token))
                .when()
                .get("/user");
    }

    /**
     * Вспомогательный метод: регистрирует пользователя и возвращает JWT-токен.
     * Используется в setup-методах других тестов.
     */
    @Step("Регистрация и получение токена для: {username}")
    public static String registerAndGetToken(String username, String email, String password) {
        RegisterRequest request = new RegisterRequest(username, email, password);
        return register(request)
                .then()
                .statusCode(200)
                .extract()
                .as(UserResponse.class)
                .getUser()
                .getToken();
    }
}
```

### 5.2 ArticleApi -- статьи

```java
package api.client;

import api.models.request.ArticleRequest;
import api.specs.RequestSpecs;
import io.qameta.allure.Step;
import io.restassured.response.Response;

import static io.restassured.RestAssured.given;

/**
 * Клиент для CRUD-операций со статьями.
 */
public class ArticleApi {

    @Step("Создание статьи: {request.article.title}")
    public static Response createArticle(String token, ArticleRequest request) {
        return given()
                .spec(RequestSpecs.authenticated(token))
                .body(request)
                .when()
                .post("/articles");
    }

    @Step("Получение статьи по slug: {slug}")
    public static Response getArticle(String slug) {
        return given()
                .spec(RequestSpecs.unauthenticated())
                .when()
                .get("/articles/{slug}", slug);
    }

    @Step("Обновление статьи: {slug}")
    public static Response updateArticle(String token, String slug, ArticleRequest request) {
        return given()
                .spec(RequestSpecs.authenticated(token))
                .body(request)
                .when()
                .put("/articles/{slug}", slug);
    }

    @Step("Удаление статьи: {slug}")
    public static Response deleteArticle(String token, String slug) {
        return given()
                .spec(RequestSpecs.authenticated(token))
                .when()
                .delete("/articles/{slug}", slug);
    }

    @Step("Получение списка статей (limit={limit}, offset={offset})")
    public static Response listArticles(int limit, int offset) {
        return given()
                .spec(RequestSpecs.unauthenticated())
                .queryParam("limit", limit)
                .queryParam("offset", offset)
                .when()
                .get("/articles");
    }

    @Step("Получение статей по тегу: {tag}")
    public static Response listArticlesByTag(String tag) {
        return given()
                .spec(RequestSpecs.unauthenticated())
                .queryParam("tag", tag)
                .when()
                .get("/articles");
    }

    @Step("Получение статей автора: {author}")
    public static Response listArticlesByAuthor(String author) {
        return given()
                .spec(RequestSpecs.unauthenticated())
                .queryParam("author", author)
                .when()
                .get("/articles");
    }

    @Step("Получение ленты подписок (Your Feed)")
    public static Response getFeed(String token) {
        return given()
                .spec(RequestSpecs.authenticated(token))
                .when()
                .get("/articles/feed");
    }
}
```

### 5.3 CommentApi -- комментарии

```java
package api.client;

import api.models.request.CommentRequest;
import api.specs.RequestSpecs;
import io.qameta.allure.Step;
import io.restassured.response.Response;

import static io.restassured.RestAssured.given;

/**
 * Клиент для операций с комментариями.
 */
public class CommentApi {

    @Step("Добавление комментария к статье: {slug}")
    public static Response addComment(String token, String slug, CommentRequest request) {
        return given()
                .spec(RequestSpecs.authenticated(token))
                .body(request)
                .when()
                .post("/articles/{slug}/comments", slug);
    }

    @Step("Получение комментариев статьи: {slug}")
    public static Response getComments(String slug) {
        return given()
                .spec(RequestSpecs.unauthenticated())
                .when()
                .get("/articles/{slug}/comments", slug);
    }

    @Step("Удаление комментария id={commentId} из статьи: {slug}")
    public static Response deleteComment(String token, String slug, int commentId) {
        return given()
                .spec(RequestSpecs.authenticated(token))
                .when()
                .delete("/articles/{slug}/comments/{id}", slug, commentId);
    }
}
```

### 5.4 ProfileApi и FavoriteApi

```java
package api.client;

import api.specs.RequestSpecs;
import io.qameta.allure.Step;
import io.restassured.response.Response;

import static io.restassured.RestAssured.given;

/**
 * Клиент для операций с профилями и подписками.
 */
public class ProfileApi {

    @Step("Получение профиля пользователя: {username}")
    public static Response getProfile(String username) {
        return given()
                .spec(RequestSpecs.unauthenticated())
                .when()
                .get("/profiles/{username}", username);
    }

    @Step("Подписка на пользователя: {username}")
    public static Response followUser(String token, String username) {
        return given()
                .spec(RequestSpecs.authenticated(token))
                .when()
                .post("/profiles/{username}/follow", username);
    }

    @Step("Отписка от пользователя: {username}")
    public static Response unfollowUser(String token, String username) {
        return given()
                .spec(RequestSpecs.authenticated(token))
                .when()
                .delete("/profiles/{username}/follow", username);
    }
}
```

```java
package api.client;

import api.specs.RequestSpecs;
import io.qameta.allure.Step;
import io.restassured.response.Response;

import static io.restassured.RestAssured.given;

/**
 * Клиент для операций с избранным (favorites).
 */
public class FavoriteApi {

    @Step("Добавление статьи в избранное: {slug}")
    public static Response favoriteArticle(String token, String slug) {
        return given()
                .spec(RequestSpecs.authenticated(token))
                .when()
                .post("/articles/{slug}/favorite", slug);
    }

    @Step("Удаление статьи из избранного: {slug}")
    public static Response unfavoriteArticle(String token, String slug) {
        return given()
                .spec(RequestSpecs.authenticated(token))
                .when()
                .delete("/articles/{slug}/favorite", slug);
    }
}
```

### 5.5 TagApi -- теги

```java
package api.client;

import api.specs.RequestSpecs;
import io.qameta.allure.Step;
import io.restassured.response.Response;

import static io.restassured.RestAssured.given;

/**
 * Клиент для получения тегов.
 */
public class TagApi {

    @Step("Получение списка всех тегов")
    public static Response getTags() {
        return given()
                .spec(RequestSpecs.unauthenticated())
                .when()
                .get("/tags");
    }
}
```

---

## Часть 6: JSON Schema валидация

### 6.1 Файл user-response.json

Поместите в `src/test/resources/json-schemas/user-response.json`:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["user"],
  "properties": {
    "user": {
      "type": "object",
      "required": ["email", "token", "username"],
      "properties": {
        "email": { "type": "string", "format": "email" },
        "token": { "type": "string", "minLength": 1 },
        "username": { "type": "string", "minLength": 1 },
        "bio": { "type": ["string", "null"] },
        "image": { "type": ["string", "null"] }
      }
    }
  }
}
```

### 6.2 Файл article-response.json

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["article"],
  "properties": {
    "article": {
      "type": "object",
      "required": ["slug", "title", "description", "body", "tagList",
                    "createdAt", "updatedAt", "favorited", "favoritesCount", "author"],
      "properties": {
        "slug": { "type": "string" },
        "title": { "type": "string" },
        "description": { "type": "string" },
        "body": { "type": "string" },
        "tagList": {
          "type": "array",
          "items": { "type": "string" }
        },
        "createdAt": { "type": "string" },
        "updatedAt": { "type": "string" },
        "favorited": { "type": "boolean" },
        "favoritesCount": { "type": "integer", "minimum": 0 },
        "author": {
          "type": "object",
          "required": ["username"],
          "properties": {
            "username": { "type": "string" },
            "bio": { "type": ["string", "null"] },
            "image": { "type": ["string", "null"] },
            "following": { "type": "boolean" }
          }
        }
      }
    }
  }
}
```

### 6.3 Файл error-response.json

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["errors"],
  "properties": {
    "errors": {
      "type": "object",
      "additionalProperties": {
        "type": "array",
        "items": { "type": "string" }
      }
    }
  }
}
```

### 6.4 Использование JSON Schema в тестах

```java
import static io.restassured.module.jsv.JsonSchemaValidator.matchesJsonSchemaInClasspath;

// Внутри теста — проверяем, что ответ соответствует JSON Schema
response
    .then()
    .body(matchesJsonSchemaInClasspath("json-schemas/user-response.json"));
```

---

## Часть 7: Управление токеном

### 7.1 Проблема

Многие тесты требуют авторизованного пользователя. Регистрировать нового пользователя
в каждом тесте -- медленно и засоряет базу. Нужен механизм повторного использования токена.

### 7.2 Решение: TokenManager

```java
package api.client;

import api.models.request.LoginRequest;
import api.models.request.RegisterRequest;
import api.models.response.UserResponse;
import utils.DataGenerator;

/**
 * Управляет JWT-токенами для тестов.
 * Регистрирует пользователя один раз и кэширует токен.
 * ThreadLocal обеспечивает потокобезопасность при параллельном запуске.
 */
public class TokenManager {

    // Кэш токена для каждого потока (для параллельного запуска)
    private static final ThreadLocal<String> tokenHolder = new ThreadLocal<>();
    private static final ThreadLocal<String> usernameHolder = new ThreadLocal<>();

    /**
     * Возвращает токен, регистрируя нового пользователя при первом вызове.
     */
    public static String getToken() {
        if (tokenHolder.get() == null) {
            createNewUser();
        }
        return tokenHolder.get();
    }

    /**
     * Возвращает username текущего тестового пользователя.
     */
    public static String getUsername() {
        if (usernameHolder.get() == null) {
            createNewUser();
        }
        return usernameHolder.get();
    }

    /**
     * Принудительно создаёт нового пользователя.
     * Полезно, если нужен чистый пользователь для конкретного теста.
     */
    public static void createNewUser() {
        String username = DataGenerator.randomUsername();
        String email = DataGenerator.randomEmail();
        String password = DataGenerator.randomPassword();

        RegisterRequest request = new RegisterRequest(username, email, password);
        UserResponse response = AuthApi.register(request)
                .then()
                .statusCode(200)
                .extract()
                .as(UserResponse.class);

        tokenHolder.set(response.getUser().getToken());
        usernameHolder.set(response.getUser().getUsername());
    }

    /** Очистка после завершения теста */
    public static void cleanup() {
        tokenHolder.remove();
        usernameHolder.remove();
    }
}
```

---

## Часть 8: Базовый тестовый класс

```java
package api.tests;

import api.client.TokenManager;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Tag;

/**
 * Базовый класс для всех API-тестов.
 * Помечен тегом "api" для фильтрации при запуске через Maven.
 */
@Tag("api")
public abstract class BaseApiTest {

    /**
     * Очищаем TokenManager после каждого теста,
     * чтобы обеспечить изоляцию при параллельном запуске.
     */
    @AfterEach
    void cleanupToken() {
        TokenManager.cleanup();
    }
}
```

---

## Часть 9: Тесты — Authentication

```java
package api.tests;

import api.client.AuthApi;
import api.models.request.LoginRequest;
import api.models.request.RegisterRequest;
import api.models.response.ErrorResponse;
import api.models.response.UserResponse;
import io.qameta.allure.*;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import utils.DataGenerator;

import static io.restassured.module.jsv.JsonSchemaValidator.matchesJsonSchemaInClasspath;
import static org.assertj.core.api.Assertions.assertThat;

@Epic("Conduit Blog Platform")
@Feature("Authentication")
@DisplayName("API: Аутентификация")
public class AuthApiTest extends BaseApiTest {

    @Test
    @Story("Register")
    @Severity(SeverityLevel.BLOCKER)
    @DisplayName("Успешная регистрация нового пользователя")
    void shouldRegisterNewUser() {
        // Подготовка уникальных данных
        String username = DataGenerator.randomUsername();
        String email = DataGenerator.randomEmail();
        String password = DataGenerator.randomPassword();
        RegisterRequest request = new RegisterRequest(username, email, password);

        // Выполнение запроса и проверка
        UserResponse response = AuthApi.register(request)
                .then()
                .statusCode(200)
                .body(matchesJsonSchemaInClasspath("json-schemas/user-response.json"))
                .extract()
                .as(UserResponse.class);

        // Валидация полей ответа
        assertThat(response.getUser().getUsername()).isEqualTo(username);
        assertThat(response.getUser().getEmail()).isEqualTo(email);
        assertThat(response.getUser().getToken()).isNotBlank();
    }

    @Test
    @Story("Register")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("Регистрация с дублирующим email возвращает 422")
    void shouldNotRegisterWithDuplicateEmail() {
        // Регистрируем первого пользователя
        String email = DataGenerator.randomEmail();
        RegisterRequest first = new RegisterRequest(
                DataGenerator.randomUsername(), email, DataGenerator.randomPassword());
        AuthApi.register(first).then().statusCode(200);

        // Пытаемся зарегистрировать второго с тем же email
        RegisterRequest second = new RegisterRequest(
                DataGenerator.randomUsername(), email, DataGenerator.randomPassword());
        ErrorResponse errorResponse = AuthApi.register(second)
                .then()
                .statusCode(422)
                .body(matchesJsonSchemaInClasspath("json-schemas/error-response.json"))
                .extract()
                .as(ErrorResponse.class);

        assertThat(errorResponse.getErrors()).containsKey("email");
    }

    @Test
    @Story("Register")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("Регистрация с пустыми полями возвращает ошибки валидации")
    void shouldNotRegisterWithEmptyFields() {
        RegisterRequest request = new RegisterRequest("", "", "");

        ErrorResponse response = AuthApi.register(request)
                .then()
                .statusCode(422)
                .extract()
                .as(ErrorResponse.class);

        assertThat(response.getErrors()).isNotEmpty();
    }

    @Test
    @Story("Login")
    @Severity(SeverityLevel.BLOCKER)
    @DisplayName("Успешный логин зарегистрированного пользователя")
    void shouldLoginWithValidCredentials() {
        // Сначала регистрируем пользователя
        String email = DataGenerator.randomEmail();
        String password = DataGenerator.randomPassword();
        RegisterRequest regRequest = new RegisterRequest(
                DataGenerator.randomUsername(), email, password);
        AuthApi.register(regRequest).then().statusCode(200);

        // Логинимся
        LoginRequest loginRequest = new LoginRequest(email, password);
        UserResponse response = AuthApi.login(loginRequest)
                .then()
                .statusCode(200)
                .body(matchesJsonSchemaInClasspath("json-schemas/user-response.json"))
                .extract()
                .as(UserResponse.class);

        assertThat(response.getUser().getEmail()).isEqualTo(email);
        assertThat(response.getUser().getToken()).isNotBlank();
    }

    @Test
    @Story("Login")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("Логин с неверным паролем возвращает 422")
    void shouldNotLoginWithWrongPassword() {
        LoginRequest request = new LoginRequest(
                DataGenerator.randomEmail(), "wrongpassword");

        AuthApi.login(request)
                .then()
                .statusCode(422);
    }

    @Test
    @Story("Current User")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("Получение текущего пользователя по токену")
    void shouldGetCurrentUser() {
        // Регистрируемся и получаем токен
        String username = DataGenerator.randomUsername();
        String email = DataGenerator.randomEmail();
        String token = AuthApi.registerAndGetToken(
                username, email, DataGenerator.randomPassword());

        // Запрашиваем текущего пользователя
        UserResponse response = AuthApi.getCurrentUser(token)
                .then()
                .statusCode(200)
                .extract()
                .as(UserResponse.class);

        assertThat(response.getUser().getUsername()).isEqualTo(username);
        assertThat(response.getUser().getEmail()).isEqualTo(email);
    }

    @Test
    @Story("Current User")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("Запрос текущего пользователя без токена возвращает 401")
    void shouldReturn401WithoutToken() {
        AuthApi.getCurrentUser("")
                .then()
                .statusCode(401);
    }
}
```

---

## Часть 10: Тесты -- Articles (CRUD)

```java
package api.tests;

import api.client.ArticleApi;
import api.client.TokenManager;
import api.models.request.ArticleRequest;
import api.models.response.ArticleResponse;
import io.qameta.allure.*;
import org.junit.jupiter.api.*;
import utils.DataGenerator;

import java.util.List;

import static io.restassured.module.jsv.JsonSchemaValidator.matchesJsonSchemaInClasspath;
import static org.assertj.core.api.Assertions.assertThat;

@Epic("Conduit Blog Platform")
@Feature("Articles")
@DisplayName("API: Статьи")
public class ArticleApiTest extends BaseApiTest {

    private String token;
    private String createdSlug;

    @BeforeEach
    void setUp() {
        // Получаем токен авторизованного пользователя
        token = TokenManager.getToken();
    }

    @Test
    @Story("Create Article")
    @Severity(SeverityLevel.BLOCKER)
    @DisplayName("Создание статьи со всеми полями")
    void shouldCreateArticle() {
        String title = DataGenerator.randomTitle();
        String description = DataGenerator.randomDescription();
        String body = DataGenerator.randomBody();
        List<String> tags = List.of(DataGenerator.randomTag(), DataGenerator.randomTag());

        ArticleRequest request = new ArticleRequest(title, description, body, tags);

        ArticleResponse response = ArticleApi.createArticle(token, request)
                .then()
                .statusCode(200)
                .body(matchesJsonSchemaInClasspath("json-schemas/article-response.json"))
                .extract()
                .as(ArticleResponse.class);

        assertThat(response.getArticle().getTitle()).isEqualTo(title);
        assertThat(response.getArticle().getDescription()).isEqualTo(description);
        assertThat(response.getArticle().getBody()).isEqualTo(body);
        assertThat(response.getArticle().getTagList()).containsExactlyInAnyOrderElementsOf(tags);
        assertThat(response.getArticle().getSlug()).isNotBlank();
        assertThat(response.getArticle().getAuthor().getUsername())
                .isEqualTo(TokenManager.getUsername());
    }

    @Test
    @Story("Get Article")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("Получение статьи по slug")
    void shouldGetArticleBySlug() {
        // Создаём статью
        String title = DataGenerator.randomTitle();
        ArticleRequest createReq = new ArticleRequest(
                title, DataGenerator.randomDescription(),
                DataGenerator.randomBody(), List.of());
        String slug = ArticleApi.createArticle(token, createReq)
                .then().statusCode(200)
                .extract().as(ArticleResponse.class)
                .getArticle().getSlug();

        // Получаем по slug
        ArticleResponse response = ArticleApi.getArticle(slug)
                .then()
                .statusCode(200)
                .extract()
                .as(ArticleResponse.class);

        assertThat(response.getArticle().getTitle()).isEqualTo(title);
        assertThat(response.getArticle().getSlug()).isEqualTo(slug);
    }

    @Test
    @Story("Update Article")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("Обновление заголовка и тела статьи")
    void shouldUpdateArticle() {
        // Создаём статью
        ArticleRequest createReq = new ArticleRequest(
                DataGenerator.randomTitle(), DataGenerator.randomDescription(),
                DataGenerator.randomBody(), List.of());
        String slug = ArticleApi.createArticle(token, createReq)
                .then().statusCode(200)
                .extract().as(ArticleResponse.class)
                .getArticle().getSlug();

        // Обновляем
        String newTitle = "Updated: " + DataGenerator.randomTitle();
        String newBody = "Updated: " + DataGenerator.randomBody();
        ArticleRequest updateReq = new ArticleRequest(newTitle, null, newBody, null);

        ArticleResponse response = ArticleApi.updateArticle(token, slug, updateReq)
                .then()
                .statusCode(200)
                .extract()
                .as(ArticleResponse.class);

        assertThat(response.getArticle().getTitle()).isEqualTo(newTitle);
        assertThat(response.getArticle().getBody()).isEqualTo(newBody);
    }

    @Test
    @Story("Delete Article")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("Удаление своей статьи")
    void shouldDeleteArticle() {
        // Создаём статью
        ArticleRequest createReq = new ArticleRequest(
                DataGenerator.randomTitle(), DataGenerator.randomDescription(),
                DataGenerator.randomBody(), List.of());
        String slug = ArticleApi.createArticle(token, createReq)
                .then().statusCode(200)
                .extract().as(ArticleResponse.class)
                .getArticle().getSlug();

        // Удаляем
        ArticleApi.deleteArticle(token, slug)
                .then()
                .statusCode(204);

        // Проверяем, что статья больше не доступна
        ArticleApi.getArticle(slug)
                .then()
                .statusCode(404);
    }

    @Test
    @Story("List Articles")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("Получение списка статей с пагинацией")
    void shouldListArticlesWithPagination() {
        ArticleApi.listArticles(10, 0)
                .then()
                .statusCode(200);
    }

    @Test
    @Story("Create Article")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("Создание статьи без авторизации возвращает 401")
    void shouldNotCreateArticleWithoutAuth() {
        ArticleRequest request = new ArticleRequest(
                DataGenerator.randomTitle(), DataGenerator.randomDescription(),
                DataGenerator.randomBody(), List.of());

        ArticleApi.createArticle("", request)
                .then()
                .statusCode(401);
    }
}
```

---

## Часть 11: Тесты -- Comments, Favorites, Tags, Profile

### 11.1 CommentApiTest

```java
package api.tests;

import api.client.ArticleApi;
import api.client.CommentApi;
import api.client.TokenManager;
import api.models.request.ArticleRequest;
import api.models.request.CommentRequest;
import api.models.response.ArticleResponse;
import io.qameta.allure.*;
import org.junit.jupiter.api.*;
import utils.DataGenerator;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.hamcrest.Matchers.*;

@Epic("Conduit Blog Platform")
@Feature("Comments")
@DisplayName("API: Комментарии")
public class CommentApiTest extends BaseApiTest {

    private String token;
    private String articleSlug;

    @BeforeEach
    void setUp() {
        token = TokenManager.getToken();

        // Создаём статью для комментирования
        ArticleRequest request = new ArticleRequest(
                DataGenerator.randomTitle(), DataGenerator.randomDescription(),
                DataGenerator.randomBody(), List.of());
        articleSlug = ArticleApi.createArticle(token, request)
                .then().statusCode(200)
                .extract().as(ArticleResponse.class)
                .getArticle().getSlug();
    }

    @Test
    @Story("Add Comment")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("Добавление комментария к статье")
    void shouldAddComment() {
        String commentBody = DataGenerator.randomComment();
        CommentRequest request = new CommentRequest(commentBody);

        CommentApi.addComment(token, articleSlug, request)
                .then()
                .statusCode(200)
                .body("comment.body", equalTo(commentBody))
                .body("comment.id", notNullValue())
                .body("comment.author.username", equalTo(TokenManager.getUsername()));
    }

    @Test
    @Story("Get Comments")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("Получение списка комментариев статьи")
    void shouldGetCommentsForArticle() {
        // Добавляем два комментария
        CommentApi.addComment(token, articleSlug,
                new CommentRequest("Первый комментарий")).then().statusCode(200);
        CommentApi.addComment(token, articleSlug,
                new CommentRequest("Второй комментарий")).then().statusCode(200);

        // Получаем список
        CommentApi.getComments(articleSlug)
                .then()
                .statusCode(200)
                .body("comments.size()", equalTo(2));
    }

    @Test
    @Story("Delete Comment")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("Удаление своего комментария")
    void shouldDeleteOwnComment() {
        // Добавляем комментарий
        CommentRequest request = new CommentRequest(DataGenerator.randomComment());
        int commentId = CommentApi.addComment(token, articleSlug, request)
                .then().statusCode(200)
                .extract().path("comment.id");

        // Удаляем
        CommentApi.deleteComment(token, articleSlug, commentId)
                .then()
                .statusCode(200);

        // Проверяем, что комментарий удалён
        CommentApi.getComments(articleSlug)
                .then()
                .statusCode(200)
                .body("comments.size()", equalTo(0));
    }
}
```

### 11.2 FavoriteApiTest

```java
package api.tests;

import api.client.ArticleApi;
import api.client.FavoriteApi;
import api.client.TokenManager;
import api.models.request.ArticleRequest;
import api.models.response.ArticleResponse;
import io.qameta.allure.*;
import org.junit.jupiter.api.*;
import utils.DataGenerator;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

@Epic("Conduit Blog Platform")
@Feature("Favorites")
@DisplayName("API: Избранное")
public class FavoriteApiTest extends BaseApiTest {

    private String token;
    private String articleSlug;

    @BeforeEach
    void setUp() {
        token = TokenManager.getToken();

        ArticleRequest request = new ArticleRequest(
                DataGenerator.randomTitle(), DataGenerator.randomDescription(),
                DataGenerator.randomBody(), List.of());
        articleSlug = ArticleApi.createArticle(token, request)
                .then().statusCode(200)
                .extract().as(ArticleResponse.class)
                .getArticle().getSlug();
    }

    @Test
    @Story("Favorite Article")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("Добавление статьи в избранное")
    void shouldFavoriteArticle() {
        ArticleResponse response = FavoriteApi.favoriteArticle(token, articleSlug)
                .then()
                .statusCode(200)
                .extract()
                .as(ArticleResponse.class);

        assertThat(response.getArticle().isFavorited()).isTrue();
        assertThat(response.getArticle().getFavoritesCount()).isEqualTo(1);
    }

    @Test
    @Story("Unfavorite Article")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("Удаление статьи из избранного")
    void shouldUnfavoriteArticle() {
        // Сначала добавляем в избранное
        FavoriteApi.favoriteArticle(token, articleSlug).then().statusCode(200);

        // Убираем из избранного
        ArticleResponse response = FavoriteApi.unfavoriteArticle(token, articleSlug)
                .then()
                .statusCode(200)
                .extract()
                .as(ArticleResponse.class);

        assertThat(response.getArticle().isFavorited()).isFalse();
        assertThat(response.getArticle().getFavoritesCount()).isEqualTo(0);
    }
}
```

### 11.3 TagApiTest

```java
package api.tests;

import api.client.TagApi;
import io.qameta.allure.*;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import static org.hamcrest.Matchers.*;

@Epic("Conduit Blog Platform")
@Feature("Tags")
@DisplayName("API: Теги")
public class TagApiTest extends BaseApiTest {

    @Test
    @Story("Get Tags")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("Получение списка тегов")
    void shouldGetTagsList() {
        TagApi.getTags()
                .then()
                .statusCode(200)
                .body("tags", notNullValue())
                .body("tags", instanceOf(java.util.List.class));
    }
}
```

### 11.4 ProfileApiTest

```java
package api.tests;

import api.client.AuthApi;
import api.client.ProfileApi;
import api.client.TokenManager;
import io.qameta.allure.*;
import org.junit.jupiter.api.*;
import utils.DataGenerator;

import static org.hamcrest.Matchers.*;

@Epic("Conduit Blog Platform")
@Feature("Profiles")
@DisplayName("API: Профили пользователей")
public class ProfileApiTest extends BaseApiTest {

    private String token;
    private String targetUsername;

    @BeforeEach
    void setUp() {
        token = TokenManager.getToken();

        // Создаём второго пользователя (цель для follow/unfollow)
        targetUsername = DataGenerator.randomUsername();
        AuthApi.registerAndGetToken(
                targetUsername, DataGenerator.randomEmail(), DataGenerator.randomPassword());
    }

    @Test
    @Story("Get Profile")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("Получение профиля пользователя")
    void shouldGetUserProfile() {
        ProfileApi.getProfile(targetUsername)
                .then()
                .statusCode(200)
                .body("profile.username", equalTo(targetUsername))
                .body("profile.following", equalTo(false));
    }

    @Test
    @Story("Follow User")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("Подписка на пользователя")
    void shouldFollowUser() {
        ProfileApi.followUser(token, targetUsername)
                .then()
                .statusCode(200)
                .body("profile.username", equalTo(targetUsername))
                .body("profile.following", equalTo(true));
    }

    @Test
    @Story("Unfollow User")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("Отписка от пользователя")
    void shouldUnfollowUser() {
        // Подписываемся
        ProfileApi.followUser(token, targetUsername).then().statusCode(200);

        // Отписываемся
        ProfileApi.unfollowUser(token, targetUsername)
                .then()
                .statusCode(200)
                .body("profile.following", equalTo(false));
    }
}
```

---

## Часть 12: Запуск и анализ

### 12.1 Запуск всех API-тестов

```bash
# Запуск только API-тестов (по JUnit Tag)
mvn clean test -Dgroups=api

# Запуск конкретного тестового класса
mvn clean test -Dtest=AuthApiTest

# Запуск с указанием другого base URL
mvn clean test -Dgroups=api -Dbase.url=https://api.realworld.io/api

# Генерация Allure-отчёта
mvn allure:serve
```

### 12.2 Ожидаемое покрытие

| Группа эндпоинтов | Количество тестов | Покрытие |
|--------------------|-------------------|----------|
| Authentication (register, login, current user) | 8 | Позитивные + негативные |
| Articles (CRUD, list, feed) | 7 | CRUD + пагинация + без авторизации |
| Comments (add, list, delete) | 3 | Полный CRUD |
| Favorites (favorite, unfavorite) | 2 | Добавление + удаление |
| Profiles (get, follow, unfollow) | 3 | Получение + подписка + отписка |
| Tags (list) | 1 | Получение списка |
| **Итого** | **24+** | Расширяется негативными сценариями |

### 12.3 Рекомендации по расширению

Для полноценного покрытия добавьте следующие тесты:

- **Boundary testing:** пустой title, очень длинный body (10000+ символов)
- **Авторизация:** попытка удалить чужую статью, чужой комментарий
- **Пагинация:** проверка `articlesCount`, корректность offset
- **Фильтры:** фильтрация по тегу, по автору, по favorited
- **Конкурентность:** два пользователя лайкают одну статью одновременно
- **Идемпотентность:** двойной favorite не увеличивает счётчик дважды

---

## Практическое задание

### Задание 1: Реализация базовой структуры

1. Создайте пакеты `api/client`, `api/models/request`, `api/models/response`, `api/specs`, `api/tests`
2. Реализуйте `ConfigReader` и `config.properties`
3. Реализуйте `DataGenerator`
4. Убедитесь, что проект компилируется: `mvn compile -pl . -Dskip.test=true`

### Задание 2: Тесты аутентификации

1. Реализуйте `AuthApi`, `RegisterRequest`, `LoginRequest`, `UserResponse`
2. Напишите минимум 5 тестов для регистрации и логина
3. Добавьте JSON Schema валидацию для user-response
4. Запустите тесты и проверьте Allure-отчёт

### Задание 3: Тесты статей

1. Реализуйте `ArticleApi`, `ArticleRequest`, `ArticleResponse`
2. Реализуйте `TokenManager` для переиспользования авторизации
3. Напишите тесты: create, get, update, delete, list
4. Добавьте тест на создание статьи без авторизации (негативный)

### Задание 4: Комментарии, профили, избранное, теги

1. Реализуйте оставшиеся API-клиенты и модели
2. Напишите тесты для каждой группы эндпоинтов
3. Добейтесь прохождения всех тестов: `mvn clean test -Dgroups=api`
4. Сгенерируйте Allure-отчёт: `mvn allure:serve`

### Задание 5: Расширение покрытия

1. Добавьте минимум 15 дополнительных негативных тестов
2. Добавьте JSON Schema для article-response и error-response
3. Добавьте `@Tag("smoke")` к 5-7 ключевым тестам
4. Общее количество API-тестов должно быть не менее 40

---

## Чек-лист самопроверки

- [ ] Структура проекта соответствует описанной (client, models, specs, tests)
- [ ] `ConfigReader` читает properties с переопределением через env-переменные
- [ ] POJO-модели покрывают все request/response форматы Conduit API
- [ ] `RequestSpecs` содержит спецификации с Allure-фильтром и без
- [ ] `ResponseSpecs` содержит переиспользуемые проверки статусов
- [ ] `TokenManager` обеспечивает потокобезопасное управление токенами
- [ ] Написано 24+ тестов, покрывающих все группы эндпоинтов
- [ ] JSON Schema валидация применяется к ответам register, login, create article
- [ ] Все тесты помечены `@Epic`, `@Feature`, `@Story`, `@Severity`
- [ ] API-клиенты используют `@Step` для отображения шагов в Allure
- [ ] Тесты проходят при запуске `mvn clean test -Dgroups=api`
- [ ] Allure-отчёт генерируется и отображает иерархию тестов
- [ ] Готов перейти к этапу UI-автоматизации
