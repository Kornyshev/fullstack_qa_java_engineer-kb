# Пример задачи: автоматизация

## Обзор

В этом разделе разберём **полный цикл автоматизации тестирования** на конкретном примере. Рассмотрим задачу
«Написать автотесты для нового API endpoint» от анализа документации до мержа PR. Каждый шаг сопровождается
примерами кода и артефактов.

Данный пример отражает реальный workflow QA Automation Engineer в Java-проекте с использованием REST Assured,
JUnit 5, Allure и Maven.

---

## Задача

```
Story: QA-345
Title: Автотесты для API управления пользователями
Priority: High
Sprint: Sprint 15

Описание:
Backend-команда реализовала новые API endpoints для управления пользователями.
Необходимо написать автоматизированные тесты.

Endpoints:
- POST /api/v1/users — создание пользователя
- GET /api/v1/users/{id} — получение пользователя по ID
- PUT /api/v1/users/{id} — обновление пользователя
- DELETE /api/v1/users/{id} — удаление пользователя
- GET /api/v1/users?page=0&size=10 — список пользователей с пагинацией

Acceptance Criteria:
1. Автотесты покрывают все CRUD-операции
2. Тесты включают позитивные и негативные сценарии
3. Используется REST Assured + JUnit 5
4. Добавлены Allure-аннотации
5. Тесты проходят в CI (Jenkins)
```

---

## Шаг 1: Анализ API-документации

### Изучение Swagger/OpenAPI

Первым делом QA изучает API-документацию (обычно Swagger UI или OpenAPI spec).

**Пример спецификации POST /api/v1/users:**

```yaml
# Фрагмент OpenAPI спецификации
paths:
  /api/v1/users:
    post:
      summary: Создание нового пользователя
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
      responses:
        '201':
          description: Пользователь создан
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserResponse'
        '400':
          description: Невалидные данные
        '409':
          description: Email уже существует

components:
  schemas:
    CreateUserRequest:
      type: object
      required: [name, email, password]
      properties:
        name:
          type: string
          minLength: 1
          maxLength: 100
        email:
          type: string
          format: email
        password:
          type: string
          minLength: 8
          maxLength: 128
        role:
          type: string
          enum: [USER, ADMIN]
          default: USER

    UserResponse:
      type: object
      properties:
        id:
          type: integer
          format: int64
        name:
          type: string
        email:
          type: string
        role:
          type: string
        createdAt:
          type: string
          format: date-time
        updatedAt:
          type: string
          format: date-time
```

### Составление тест-плана

```
Тест-план для API /api/v1/users:

POST /api/v1/users (создание):
  Позитивные:
    ✅ Создание пользователя с обязательными полями
    ✅ Создание пользователя со всеми полями (включая role)
    ✅ Создание пользователя с граничными значениями (имя 1 символ, имя 100 символов)
  Негативные:
    ✅ Создание без обязательного поля (name, email, password)
    ✅ Создание с невалидным email
    ✅ Создание с коротким паролем (< 8 символов)
    ✅ Создание с дублирующимся email (409 Conflict)

GET /api/v1/users/{id} (получение):
  Позитивные:
    ✅ Получение существующего пользователя
  Негативные:
    ✅ Получение несуществующего пользователя (404)

PUT /api/v1/users/{id} (обновление):
  Позитивные:
    ✅ Обновление имени пользователя
    ✅ Обновление всех полей
  Негативные:
    ✅ Обновление несуществующего пользователя (404)
    ✅ Обновление с невалидными данными (400)

DELETE /api/v1/users/{id} (удаление):
  Позитивные:
    ✅ Удаление существующего пользователя
  Негативные:
    ✅ Удаление несуществующего пользователя (404)
    ✅ Получение удалённого пользователя (404)

GET /api/v1/users?page&size (список):
  Позитивные:
    ✅ Получение первой страницы
    ✅ Пагинация (page=0, page=1)
  Негативные:
    ✅ Невалидные параметры пагинации
```

---

## Шаг 2: Создание POJO-моделей

### Request-модель

```java
package com.example.tests.model;

import com.fasterxml.jackson.annotation.JsonInclude;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

/**
 * Модель запроса для создания пользователя.
 * Используется в тестах POST /api/v1/users
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class CreateUserRequest {

    private String name;
    private String email;
    private String password;
    private String role;
}
```

### Response-модель

```java
package com.example.tests.model;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

/**
 * Модель ответа с данными пользователя.
 * Используется для десериализации ответов API
 */
@Data
@NoArgsConstructor
@AllArgsConstructor
public class UserResponse {

    private Long id;
    private String name;
    private String email;
    private String role;
    private String createdAt;
    private String updatedAt;
}
```

### Модель ошибки

```java
package com.example.tests.model;

import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.List;

/**
 * Модель ответа при ошибке валидации (400 Bad Request)
 */
@Data
@NoArgsConstructor
public class ErrorResponse {

    private int status;
    private String message;
    private List<FieldError> errors;

    @Data
    @NoArgsConstructor
    public static class FieldError {
        private String field;
        private String message;
    }
}
```

---

## Шаг 3: Настройка базовой конфигурации

### Спецификация запросов

```java
package com.example.tests.config;

import io.restassured.builder.RequestSpecBuilder;
import io.restassured.filter.log.LogDetail;
import io.restassured.http.ContentType;
import io.restassured.specification.RequestSpecification;

/**
 * Базовая конфигурация для всех API-запросов.
 * Содержит общие настройки: base URL, content type, логирование
 */
public class ApiConfig {

    private static final String BASE_URL = System.getProperty("base.url", "http://localhost:8080");

    public static RequestSpecification baseSpec() {
        return new RequestSpecBuilder()
                .setBaseUri(BASE_URL)
                .setContentType(ContentType.JSON)
                .setAccept(ContentType.JSON)
                .log(LogDetail.ALL)
                .build();
    }

    // Спецификация с авторизацией (если нужна)
    public static RequestSpecification authorizedSpec(String token) {
        return new RequestSpecBuilder()
                .addRequestSpecification(baseSpec())
                .addHeader("Authorization", "Bearer " + token)
                .build();
    }
}
```

### Генератор тестовых данных

```java
package com.example.tests.util;

import com.example.tests.model.CreateUserRequest;
import org.apache.commons.lang3.RandomStringUtils;

/**
 * Утилита для генерации тестовых данных.
 * Каждый вызов создаёт уникальные данные, чтобы тесты не конфликтовали
 */
public class TestDataGenerator {

    public static CreateUserRequest randomUser() {
        String uniqueId = RandomStringUtils.randomAlphanumeric(8).toLowerCase();
        return CreateUserRequest.builder()
                .name("Test User " + uniqueId)
                .email("testuser_" + uniqueId + "@example.com")
                .password("Password1")
                .role("USER")
                .build();
    }

    public static CreateUserRequest randomAdmin() {
        CreateUserRequest user = randomUser();
        user.setRole("ADMIN");
        return user;
    }
}
```

---

## Шаг 4: Написание тестов

### Тесты создания пользователя (POST)

```java
package com.example.tests.api;

import com.example.tests.config.ApiConfig;
import com.example.tests.model.CreateUserRequest;
import com.example.tests.model.ErrorResponse;
import com.example.tests.model.UserResponse;
import com.example.tests.util.TestDataGenerator;
import io.qameta.allure.*;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;

@Epic("Управление пользователями")
@Feature("Создание пользователя")
@Tag("api")
@Tag("users")
public class CreateUserTest {

    private static final String USERS_ENDPOINT = "/api/v1/users";

    @Test
    @Story("QA-345")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("Создание пользователя с валидными обязательными полями")
    @Description("Проверяем, что пользователь успешно создаётся при передаче всех обязательных полей")
    public void testCreateUserWithRequiredFields() {
        // Подготовка тестовых данных
        CreateUserRequest request = TestDataGenerator.randomUser();

        // Выполнение запроса
        UserResponse response = given()
                .spec(ApiConfig.baseSpec())
                .body(request)
                .when()
                .post(USERS_ENDPOINT)
                .then()
                .statusCode(201)
                .extract()
                .as(UserResponse.class);

        // Проверки
        assertThat(response.getId()).isNotNull().isPositive();
        assertThat(response.getName()).isEqualTo(request.getName());
        assertThat(response.getEmail()).isEqualTo(request.getEmail());
        assertThat(response.getRole()).isEqualTo("USER");
        assertThat(response.getCreatedAt()).isNotNull();
    }

    @Test
    @Story("QA-345")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("Создание пользователя с ролью ADMIN")
    public void testCreateUserWithAdminRole() {
        CreateUserRequest request = TestDataGenerator.randomAdmin();

        UserResponse response = given()
                .spec(ApiConfig.baseSpec())
                .body(request)
                .when()
                .post(USERS_ENDPOINT)
                .then()
                .statusCode(201)
                .extract()
                .as(UserResponse.class);

        assertThat(response.getRole()).isEqualTo("ADMIN");
    }

    @Test
    @Story("QA-345")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("Создание пользователя с дублирующимся email — 409 Conflict")
    public void testCreateUserDuplicateEmail() {
        // Создаём первого пользователя
        CreateUserRequest request = TestDataGenerator.randomUser();

        given()
                .spec(ApiConfig.baseSpec())
                .body(request)
                .when()
                .post(USERS_ENDPOINT)
                .then()
                .statusCode(201);

        // Пытаемся создать второго с тем же email
        CreateUserRequest duplicate = TestDataGenerator.randomUser();
        duplicate.setEmail(request.getEmail()); // тот же email

        given()
                .spec(ApiConfig.baseSpec())
                .body(duplicate)
                .when()
                .post(USERS_ENDPOINT)
                .then()
                .statusCode(409);
    }

    @Test
    @Story("QA-345")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("Создание пользователя без обязательного поля name — 400")
    public void testCreateUserWithoutName() {
        CreateUserRequest request = TestDataGenerator.randomUser();
        request.setName(null); // убираем обязательное поле

        ErrorResponse error = given()
                .spec(ApiConfig.baseSpec())
                .body(request)
                .when()
                .post(USERS_ENDPOINT)
                .then()
                .statusCode(400)
                .extract()
                .as(ErrorResponse.class);

        assertThat(error.getErrors())
                .extracting("field")
                .contains("name");
    }

    @Test
    @Story("QA-345")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("Создание пользователя с невалидным email — 400")
    public void testCreateUserWithInvalidEmail() {
        CreateUserRequest request = TestDataGenerator.randomUser();
        request.setEmail("not-an-email");

        given()
                .spec(ApiConfig.baseSpec())
                .body(request)
                .when()
                .post(USERS_ENDPOINT)
                .then()
                .statusCode(400);
    }

    @Test
    @Story("QA-345")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("Создание пользователя с коротким паролем — 400")
    public void testCreateUserWithShortPassword() {
        CreateUserRequest request = TestDataGenerator.randomUser();
        request.setPassword("Short1"); // меньше 8 символов

        given()
                .spec(ApiConfig.baseSpec())
                .body(request)
                .when()
                .post(USERS_ENDPOINT)
                .then()
                .statusCode(400);
    }
}
```

### Тесты получения, обновления и удаления

```java
package com.example.tests.api;

import com.example.tests.config.ApiConfig;
import com.example.tests.model.CreateUserRequest;
import com.example.tests.model.UserResponse;
import com.example.tests.util.TestDataGenerator;
import io.qameta.allure.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;

@Epic("Управление пользователями")
@Feature("CRUD-операции с пользователями")
@Tag("api")
@Tag("users")
public class UserCrudTest {

    private static final String USERS_ENDPOINT = "/api/v1/users";
    private static final String USER_BY_ID_ENDPOINT = "/api/v1/users/{id}";

    private UserResponse createdUser;

    @BeforeEach
    @Step("Создание тестового пользователя")
    void setUp() {
        // Перед каждым тестом создаём нового пользователя
        CreateUserRequest request = TestDataGenerator.randomUser();

        createdUser = given()
                .spec(ApiConfig.baseSpec())
                .body(request)
                .when()
                .post(USERS_ENDPOINT)
                .then()
                .statusCode(201)
                .extract()
                .as(UserResponse.class);
    }

    @Test
    @Story("QA-345")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("Получение пользователя по ID")
    public void testGetUserById() {
        UserResponse response = given()
                .spec(ApiConfig.baseSpec())
                .pathParam("id", createdUser.getId())
                .when()
                .get(USER_BY_ID_ENDPOINT)
                .then()
                .statusCode(200)
                .extract()
                .as(UserResponse.class);

        assertThat(response.getId()).isEqualTo(createdUser.getId());
        assertThat(response.getName()).isEqualTo(createdUser.getName());
        assertThat(response.getEmail()).isEqualTo(createdUser.getEmail());
    }

    @Test
    @Story("QA-345")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("Получение несуществующего пользователя — 404")
    public void testGetNonExistentUser() {
        given()
                .spec(ApiConfig.baseSpec())
                .pathParam("id", 999999)
                .when()
                .get(USER_BY_ID_ENDPOINT)
                .then()
                .statusCode(404);
    }

    @Test
    @Story("QA-345")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("Обновление имени пользователя")
    public void testUpdateUserName() {
        CreateUserRequest updateRequest = CreateUserRequest.builder()
                .name("Updated Name")
                .email(createdUser.getEmail())
                .password("Password1")
                .build();

        UserResponse response = given()
                .spec(ApiConfig.baseSpec())
                .pathParam("id", createdUser.getId())
                .body(updateRequest)
                .when()
                .put(USER_BY_ID_ENDPOINT)
                .then()
                .statusCode(200)
                .extract()
                .as(UserResponse.class);

        assertThat(response.getName()).isEqualTo("Updated Name");
        assertThat(response.getUpdatedAt()).isNotEqualTo(createdUser.getUpdatedAt());
    }

    @Test
    @Story("QA-345")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("Удаление пользователя")
    public void testDeleteUser() {
        // Удаляем пользователя
        given()
                .spec(ApiConfig.baseSpec())
                .pathParam("id", createdUser.getId())
                .when()
                .delete(USER_BY_ID_ENDPOINT)
                .then()
                .statusCode(204);

        // Проверяем, что пользователь больше не доступен
        given()
                .spec(ApiConfig.baseSpec())
                .pathParam("id", createdUser.getId())
                .when()
                .get(USER_BY_ID_ENDPOINT)
                .then()
                .statusCode(404);
    }

    @Test
    @Story("QA-345")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("Удаление несуществующего пользователя — 404")
    public void testDeleteNonExistentUser() {
        given()
                .spec(ApiConfig.baseSpec())
                .pathParam("id", 999999)
                .when()
                .delete(USER_BY_ID_ENDPOINT)
                .then()
                .statusCode(404);
    }
}
```

---

## Шаг 5: Локальный запуск и проверка

### Запуск тестов

```bash
# Запуск всех тестов с тегом "users"
mvn clean test -Dgroups="users" -Dbase.url=http://localhost:8080

# Запуск с генерацией Allure-отчёта
mvn clean test -Dgroups="users" && mvn allure:serve
```

### Проверка результатов

```
Ожидаемый результат локального запуска:

Tests run: 12, Failures: 0, Errors: 0, Skipped: 0

Allure Report:
- 12 тестов passed
- Все @Story, @Epic, @Feature корректно отображаются
- Шаги запросов и ответов видны в отчёте
- Время выполнения < 30 секунд
```

---

## Шаг 6: Создание Pull Request

### Структура изменений

```
src/test/java/com/example/tests/
├── model/
│   ├── CreateUserRequest.java    (новый)
│   ├── UserResponse.java         (новый)
│   └── ErrorResponse.java        (новый)
├── config/
│   └── ApiConfig.java            (новый или обновлённый)
├── util/
│   └── TestDataGenerator.java    (новый или обновлённый)
└── api/
    ├── CreateUserTest.java       (новый)
    └── UserCrudTest.java         (новый)
```

### PR Description

```
Title: [QA-345] Автотесты для API управления пользователями

Описание:
Добавлены автоматизированные API-тесты для endpoints /api/v1/users.

Что сделано:
- Созданы POJO-модели для request/response
- Написаны 12 тестов (6 позитивных, 6 негативных)
- Добавлены Allure-аннотации (@Epic, @Feature, @Story, @Severity)
- Добавлен генератор тестовых данных

Покрытие:
- POST /api/v1/users — 6 тестов
- GET /api/v1/users/{id} — 2 теста
- PUT /api/v1/users/{id} — 1 тест
- DELETE /api/v1/users/{id} — 2 теста
- GET /api/v1/users (список) — 1 тест (в следующем PR)

Как проверить:
mvn clean test -Dgroups="users" -Dbase.url=http://localhost:8080
```

---

## Шаг 7: Code Review и мерж

### Типичные замечания на Code Review

```
Ревьюер: "TestDataGenerator.randomUser() не гарантирует уникальность email
при параллельном запуске. Рассмотри добавление timestamp."
→ Fix: добавил System.currentTimeMillis() в email

Ревьюер: "В testDeleteUser() нет @Step аннотации для шага удаления."
→ Fix: добавил @Step

Ревьюер: "Хардкод URL в ApiConfig.java. Нужно брать из properties файла."
→ Fix: уже используется System.getProperty с default-значением
```

### После мержа

```
☐ PR смержен в develop
☐ CI pipeline запущен с новыми тестами
☐ Все 12 тестов проходят в CI
☐ Allure Report сгенерирован и доступен
☐ Story QA-345 обновлена — "Done"
☐ TMS обновлена — test cases связаны с автотестами
```

---

## Связь с тестированием

- Автоматизация API-тестов обеспечивает **быструю регрессию** при каждом коммите
- POJO-модели гарантируют **типизированную проверку** контракта API
- Allure-аннотации обеспечивают **прозрачность** — какие stories покрыты тестами
- Генератор тестовых данных обеспечивает **независимость** тестов друг от друга
- Code review автотестов повышает **качество тестового кода**

---

## Типичные ошибки

1. **Нет анализа API-документации** — пишем тесты, не понимая контракт
2. **Только happy path** — покрываем позитивные сценарии, забываем негативные
3. **Захардкоженные тестовые данные** — тесты конфликтуют при параллельном запуске
4. **Нет cleanup** — тестовые данные накапливаются в БД
5. **Отсутствие Allure-аннотаций** — отчёт не информативен
6. **Тест зависит от другого теста** — порядок выполнения важен (антипаттерн)
7. **Нет проверки response body** — проверяем только status code
8. **Copy-paste вместо переиспользования** — дублирование кода в тестах

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Какие инструменты вы используете для API-автоматизации?
2. Как вы определяете, какие тесты автоматизировать?
3. Что такое REST Assured? Как отправить POST-запрос?
4. Зачем нужны POJO-модели в тестах?
5. Что такое Allure Report и какие аннотации вы используете?

### 🟡 Средний уровень
6. Как организовать тестовые данные, чтобы тесты не зависели друг от друга?
7. Как обеспечить идемпотентность тестов (можно запускать повторно)?
8. Как вы интегрируете автотесты в CI/CD pipeline?
9. Как организовать проект автотестов (структура пакетов, паттерны)?
10. Как тестировать API, требующий авторизации (OAuth2, JWT)?

### 🔴 Продвинутый уровень
11. Как организовать параллельный запуск API-тестов без конфликтов?
12. Как реализовать contract testing между микросервисами?
13. Как тестировать асинхронные API (WebSocket, SSE, message queues)?
14. Как организовать тестирование API с database state verification?
15. Как построить test framework с нуля: какие паттерны и абстракции использовать?

---

## Практические задания

### Задание 1: Полный цикл
Возьмите любой публичный API (JSONPlaceholder, ReqRes, PetStore). Пройдите полный цикл:
- Анализ документации
- Создание POJO-моделей
- Написание 10+ тестов
- Allure-отчёт

### Задание 2: Негативные тесты
Для endpoint POST /api/v1/users напишите максимально полный набор негативных тестов. Цель: минимум 15 негативных сценариев.

### Задание 3: Test Framework
Создайте минимальный test framework:
- Base test class с общей конфигурацией
- Request/Response модели
- Генератор тестовых данных
- Allure-интеграция
- README с инструкцией по запуску

### Задание 4: Code Review
Найдите open-source проект с API-тестами на GitHub. Проведите code review: найдите 5 вещей, которые можно улучшить.

---

## Дополнительные ресурсы

- **REST Assured Documentation** (rest-assured.io) — официальная документация
- **Allure Framework** (docs.qameta.io) — документация Allure
- **«Java Testing with JUnit 5» by Boni García** — современные практики тестирования на Java
- **Baeldung — REST Assured Tutorial** (baeldung.com) — практические гайды
- **Swagger/OpenAPI Specification** (swagger.io) — стандарт описания API
