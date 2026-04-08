# Тестирование аутентификации в API

## Обзор

Аутентификация и авторизация — критически важные аспекты безопасности API. Для QA-инженера умение тестировать различные механизмы аутентификации является обязательным навыком. Баги в этой области могут привести к утечке данных, несанкционированному доступу и серьёзным последствиям для бизнеса.

**Аутентификация** (authentication) — процесс подтверждения личности: «Кто ты?»
**Авторизация** (authorization) — процесс проверки прав доступа: «Что тебе можно делать?»

В этом разделе рассмотрены основные типы аутентификации, как их тестировать с помощью REST Assured, и какие аспекты безопасности необходимо проверять.

---

## Basic Authentication

### Как работает

Basic Auth — простейший механизм аутентификации, при котором имя пользователя и пароль передаются в заголовке `Authorization` в формате Base64.

```
Процесс:
1. Клиент формирует строку: "username:password"
2. Кодирует её в Base64: "dXNlcm5hbWU6cGFzc3dvcmQ="
3. Отправляет заголовок: Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=
```

> **Важно:** Base64 — это **кодирование**, а не **шифрование**. Любой может декодировать строку. Поэтому Basic Auth следует использовать **только через HTTPS**.

### Тестирование с REST Assured

```java
@Test
@DisplayName("Basic Auth — успешная аутентификация")
void shouldAuthenticateWithBasicAuth() {
    given()
        .auth().basic("admin", "password123")    // REST Assured формирует заголовок
    .when()
        .get("/api/secured/resource")
    .then()
        .statusCode(200);
}

@Test
@DisplayName("Basic Auth — preemptive (отправить сразу, без challenge)")
void shouldAuthenticateWithPreemptiveBasicAuth() {
    // Preemptive — отправляет заголовок сразу, не дожидаясь 401
    given()
        .auth().preemptive().basic("admin", "password123")
    .when()
        .get("/api/secured/resource")
    .then()
        .statusCode(200);
}

@Test
@DisplayName("Basic Auth — неверные учётные данные")
void shouldReturn401WithInvalidCredentials() {
    given()
        .auth().preemptive().basic("admin", "wrongPassword")
    .when()
        .get("/api/secured/resource")
    .then()
        .statusCode(401)
        .header("WWW-Authenticate", containsString("Basic"));
}

@Test
@DisplayName("Basic Auth — отсутствие заголовка авторизации")
void shouldReturn401WithoutAuthHeader() {
    given()
        // Без auth
    .when()
        .get("/api/secured/resource")
    .then()
        .statusCode(401);
}
```

### Что проверять

- Успешная аутентификация с корректными данными
- Отказ при неверном пароле (401)
- Отказ при отсутствующем заголовке (401)
- Отказ при пустом имени пользователя или пароле
- Поведение при спецсимволах в пароле (`:`, `@`, кириллица)
- Наличие заголовка `WWW-Authenticate` в ответе 401

---

## Bearer Token (JWT)

### Как работает JWT

JWT (JSON Web Token) — компактный, URL-безопасный токен, состоящий из трёх частей, разделённых точками.

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6Ik
pvaG4gRG9lIiwicm9sZSI6IkFETUlOIiwiZXhw
IjoxNzA3MjMxMjAwfQ.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

#### Структура JWT

| Часть | Содержимое | Пример (декодированный) |
|-------|-----------|------------------------|
| **Header** | Алгоритм подписи и тип токена | `{"alg": "HS256", "typ": "JWT"}` |
| **Payload** | Полезные данные (claims) | `{"sub": "1234", "name": "John", "role": "ADMIN", "exp": 1707231200}` |
| **Signature** | Подпись для проверки целостности | `HMACSHA256(base64(header) + "." + base64(payload), secret)` |

#### Стандартные claims в payload

| Claim | Назначение | Пример |
|-------|-----------|--------|
| `sub` | Subject — идентификатор пользователя | `"user-42"` |
| `exp` | Expiration — время истечения (Unix timestamp) | `1707231200` |
| `iat` | Issued At — время создания | `1707144800` |
| `iss` | Issuer — кто выдал токен | `"auth.example.com"` |
| `aud` | Audience — для кого предназначен | `"api.example.com"` |
| `role` | Роль пользователя (кастомный claim) | `"ADMIN"` |

### Тестирование с REST Assured

```java
class JwtAuthTest {

    private String authToken;

    @BeforeEach
    void obtainToken() {
        // Получение токена через endpoint логина
        authToken = given()
            .contentType(ContentType.JSON)
            .body("""
                {
                    "username": "testuser",
                    "password": "testpass"
                }
                """)
        .when()
            .post("/api/auth/login")
        .then()
            .statusCode(200)
            .extract()
            .path("token");
    }

    @Test
    @DisplayName("Доступ с валидным токеном")
    void shouldAccessWithValidToken() {
        given()
            .header("Authorization", "Bearer " + authToken)
        .when()
            .get("/api/users/me")
        .then()
            .statusCode(200)
            .body("username", equalTo("testuser"));
    }

    @Test
    @DisplayName("Отказ с невалидным токеном")
    void shouldReturn401WithInvalidToken() {
        given()
            .header("Authorization", "Bearer invalid.token.here")
        .when()
            .get("/api/users/me")
        .then()
            .statusCode(401);
    }

    @Test
    @DisplayName("Отказ без токена")
    void shouldReturn401WithoutToken() {
        given()
        .when()
            .get("/api/users/me")
        .then()
            .statusCode(401);
    }

    @Test
    @DisplayName("Отказ с истёкшим токеном")
    void shouldReturn401WithExpiredToken() {
        // Токен с exp в прошлом (создан заранее для тестов)
        String expiredToken = "eyJhbGciOiJIUzI1NiJ9." +
            "eyJzdWIiOiJ0ZXN0IiwiZXhwIjoxNjAwMDAwMDAwfQ." +
            "signature";

        given()
            .header("Authorization", "Bearer " + expiredToken)
        .when()
            .get("/api/users/me")
        .then()
            .statusCode(401);
    }

    @Test
    @DisplayName("Отказ при изменённом payload (подпись невалидна)")
    void shouldReturn401WithTamperedToken() {
        // Изменяем payload токена — подпись станет невалидной
        String[] parts = authToken.split("\\.");
        String tamperedPayload = Base64.getUrlEncoder()
            .encodeToString("{\"sub\":\"admin\",\"role\":\"SUPERADMIN\"}".getBytes());
        String tamperedToken = parts[0] + "." + tamperedPayload + "." + parts[2];

        given()
            .header("Authorization", "Bearer " + tamperedToken)
        .when()
            .get("/api/users/me")
        .then()
            .statusCode(401);
    }

    @Test
    @DisplayName("Проверка ролевой авторизации — недостаточно прав")
    void shouldReturn403WithInsufficientRole() {
        // Токен обычного пользователя
        given()
            .header("Authorization", "Bearer " + authToken)
        .when()
            .get("/api/admin/settings")
        .then()
            .statusCode(403); // Аутентифицирован, но нет прав
    }
}
```

### Что проверять при тестировании JWT

- Доступ с валидным токеном
- Отказ без токена (401)
- Отказ с невалидным токеном (401)
- Отказ с истёкшим токеном (401)
- Отказ с изменённым payload (подпись невалидна) (401)
- Ролевая авторизация (403 при недостаточных правах)
- Поведение при пустом значении `Authorization: Bearer`
- Формат заголовка: `Bearer` с большой буквы, пробел перед токеном

---

## OAuth 2.0

### Основные потоки (Flows)

| Flow | Когда используется | Участие пользователя |
|------|-------------------|---------------------|
| Authorization Code | Веб-приложения, серверные | Да (логин через браузер) |
| Client Credentials | Межсервисное взаимодействие (M2M) | Нет |
| Password (Resource Owner) | Устаревший, не рекомендуется | Да (передача логин/пароль) |
| Implicit | Устаревший, заменён на Authorization Code + PKCE | Да |

### Client Credentials Flow (для автоматизации)

Наиболее подходящий flow для автоматизированных тестов, когда не нужно участие пользователя.

```
1. Клиент → Authorization Server:
   POST /oauth/token
   grant_type=client_credentials
   client_id=my-client
   client_secret=my-secret

2. Authorization Server → Клиент:
   {
     "access_token": "eyJhbGci...",
     "token_type": "Bearer",
     "expires_in": 3600
   }

3. Клиент → Resource Server:
   GET /api/resource
   Authorization: Bearer eyJhbGci...
```

### Тестирование с REST Assured

```java
class OAuth2Test {

    private static String accessToken;

    @BeforeAll
    static void obtainOAuth2Token() {
        // Получение токена через Client Credentials flow
        accessToken = given()
            .contentType("application/x-www-form-urlencoded")
            .formParam("grant_type", "client_credentials")
            .formParam("client_id", "test-client")
            .formParam("client_secret", "test-secret")
            .formParam("scope", "read write")
        .when()
            .post("https://auth.example.com/oauth/token")
        .then()
            .statusCode(200)
            .body("token_type", equalToIgnoringCase("bearer"))
            .body("expires_in", greaterThan(0))
            .extract()
            .path("access_token");
    }

    @Test
    @DisplayName("Доступ к ресурсу с OAuth2 токеном")
    void shouldAccessResourceWithOAuth2Token() {
        given()
            .auth().oauth2(accessToken)    // Удобный метод REST Assured
        .when()
            .get("/api/resource")
        .then()
            .statusCode(200);
    }

    @Test
    @DisplayName("Отказ с невалидным client_secret")
    void shouldRejectInvalidClientSecret() {
        given()
            .contentType("application/x-www-form-urlencoded")
            .formParam("grant_type", "client_credentials")
            .formParam("client_id", "test-client")
            .formParam("client_secret", "wrong-secret")
        .when()
            .post("https://auth.example.com/oauth/token")
        .then()
            .statusCode(401);
    }

    @Test
    @DisplayName("Отказ при запросе scope, к которому нет доступа")
    void shouldRejectUnauthorizedScope() {
        given()
            .contentType("application/x-www-form-urlencoded")
            .formParam("grant_type", "client_credentials")
            .formParam("client_id", "test-client")
            .formParam("client_secret", "test-secret")
            .formParam("scope", "admin:delete")
        .when()
            .post("https://auth.example.com/oauth/token")
        .then()
            .statusCode(400)
            .body("error", equalTo("invalid_scope"));
    }
}
```

---

## API Key Authentication

### Как работает

API Key — уникальный ключ, выданный клиенту. Передаётся в заголовке, query parameter или cookie.

```
# Через заголовок (наиболее безопасный вариант)
X-Api-Key: sk-abc123def456

# Через query parameter (менее безопасный — ключ виден в URL/логах)
GET /api/resource?api_key=sk-abc123def456
```

### Тестирование с REST Assured

```java
@Test
@DisplayName("API Key в заголовке — успешный запрос")
void shouldAuthenticateWithApiKey() {
    given()
        .header("X-Api-Key", "valid-api-key-123")
    .when()
        .get("/api/data")
    .then()
        .statusCode(200);
}

@Test
@DisplayName("API Key — невалидный ключ")
void shouldReturn401WithInvalidApiKey() {
    given()
        .header("X-Api-Key", "invalid-key")
    .when()
        .get("/api/data")
    .then()
        .statusCode(401);
}

@Test
@DisplayName("API Key — отсутствующий ключ")
void shouldReturn401WithoutApiKey() {
    given()
    .when()
        .get("/api/data")
    .then()
        .statusCode(401);
}

@Test
@DisplayName("API Key через query parameter")
void shouldAuthenticateWithApiKeyInQuery() {
    given()
        .queryParam("api_key", "valid-api-key-123")
    .when()
        .get("/api/data")
    .then()
        .statusCode(200);
}
```

### Что проверять

- Валидный ключ — доступ предоставлен
- Невалидный ключ — 401
- Отсутствующий ключ — 401
- Отозванный ключ — 401 (если поддерживается)
- Rate limiting по ключу (429)
- Ключ в логах не должен быть в открытом виде

---

## Cookie-based Authentication

### Как работает

Серверные сессии: при логине сервер создаёт сессию и отправляет cookie с идентификатором сессии.

```
1. POST /api/auth/login  →  Set-Cookie: SESSION=abc123; HttpOnly; Secure; SameSite=Strict
2. GET /api/resource     →  Cookie: SESSION=abc123
```

### Тестирование с REST Assured

```java
@Test
@DisplayName("Cookie-based Auth — полный flow")
void shouldAuthenticateWithCookies() {
    // Шаг 1: логин и получение cookie
    Response loginResponse = given()
        .contentType(ContentType.JSON)
        .body("""
            { "username": "testuser", "password": "testpass" }
            """)
    .when()
        .post("/api/auth/login")
    .then()
        .statusCode(200)
        .cookie("SESSION")    // Проверяем наличие cookie
        .extract()
        .response();

    String sessionCookie = loginResponse.getCookie("SESSION");

    // Шаг 2: доступ к защищённому ресурсу с cookie
    given()
        .cookie("SESSION", sessionCookie)
    .when()
        .get("/api/users/me")
    .then()
        .statusCode(200)
        .body("username", equalTo("testuser"));

    // Шаг 3: логаут
    given()
        .cookie("SESSION", sessionCookie)
    .when()
        .post("/api/auth/logout")
    .then()
        .statusCode(200);

    // Шаг 4: попытка доступа после логаута
    given()
        .cookie("SESSION", sessionCookie)
    .when()
        .get("/api/users/me")
    .then()
        .statusCode(401); // Сессия должна быть невалидна
}
```

### Проверки безопасности cookie

```java
@Test
@DisplayName("Проверка атрибутов безопасности cookie")
void shouldSetSecureCookieAttributes() {
    Response response = given()
        .contentType(ContentType.JSON)
        .body("""
            { "username": "testuser", "password": "testpass" }
            """)
    .when()
        .post("/api/auth/login");

    // Получаем подробную информацию о cookie
    Cookie sessionCookie = response.getDetailedCookie("SESSION");

    assertNotNull(sessionCookie);
    assertTrue(sessionCookie.isHttpOnly(), "Cookie должен быть HttpOnly");
    assertTrue(sessionCookie.isSecured(), "Cookie должен быть Secure");
    assertEquals("Strict", sessionCookie.getSameSite(), "SameSite должен быть Strict");
}
```

---

## Тестирование безопасности аутентификации

### Общие проверки безопасности

```java
class SecurityTest {

    @Test
    @DisplayName("Brute Force — блокировка после множественных неудачных попыток")
    void shouldLockAccountAfterMultipleFailedAttempts() {
        // Выполняем несколько неудачных попыток логина
        for (int i = 0; i < 5; i++) {
            given()
                .contentType(ContentType.JSON)
                .body("""
                    { "username": "testuser", "password": "wrongpass" }
                    """)
            .when()
                .post("/api/auth/login")
            .then()
                .statusCode(401);
        }

        // После блокировки даже верный пароль не должен работать
        given()
            .contentType(ContentType.JSON)
            .body("""
                { "username": "testuser", "password": "correctpass" }
                """)
        .when()
            .post("/api/auth/login")
        .then()
            .statusCode(anyOf(is(429), is(423))); // Too Many Requests или Locked
    }

    @Test
    @DisplayName("Чувствительные данные не должны возвращаться в ответе")
    void shouldNotExposePasswordInResponse() {
        Response response = given()
            .header("Authorization", "Bearer valid-token")
        .when()
            .get("/api/users/me")
        .then()
            .statusCode(200)
            .extract().response();

        String responseBody = response.getBody().asString();

        // Пароль и токены не должны присутствовать в ответе
        assertFalse(responseBody.contains("password"));
        assertFalse(responseBody.contains("secret"));
    }

    @Test
    @DisplayName("Сообщение об ошибке не раскрывает деталей")
    void shouldNotRevealSpecificAuthError() {
        // Неверное имя пользователя
        String messageForWrongUser = given()
            .contentType(ContentType.JSON)
            .body("""
                { "username": "nonexistent", "password": "pass" }
                """)
        .when()
            .post("/api/auth/login")
        .then()
            .statusCode(401)
            .extract().path("message");

        // Неверный пароль
        String messageForWrongPassword = given()
            .contentType(ContentType.JSON)
            .body("""
                { "username": "testuser", "password": "wrongpass" }
                """)
        .when()
            .post("/api/auth/login")
        .then()
            .statusCode(401)
            .extract().path("message");

        // Сообщения должны быть одинаковыми (не раскрывать, что именно неверно)
        assertEquals(messageForWrongUser, messageForWrongPassword,
            "Сообщения об ошибке не должны отличаться для неверного логина и пароля");
    }

    @Test
    @DisplayName("Токен не должен передаваться через URL")
    void shouldNotAcceptTokenInUrl() {
        // Токен в URL — уязвимость (попадает в логи, referer, историю браузера)
        given()
            .queryParam("token", "valid-jwt-token")
        .when()
            .get("/api/users/me")
        .then()
            .statusCode(401); // Не должен принимать токен из URL
    }
}
```

### Чек-лист безопасности аутентификации

| Проверка | Ожидание |
|---------|----------|
| Запрос без токена | 401 |
| Невалидный токен | 401 |
| Истёкший токен | 401 |
| Изменённый payload JWT | 401 |
| Алгоритм `none` в JWT header | 401 |
| Токен другого пользователя для доступа к чужим данным | 403 |
| Множественные неудачные попыки логина | Блокировка (423/429) |
| Пароль в ответе API | Отсутствует |
| Одинаковое сообщение при неверном логине и пароле | Да |
| HttpOnly, Secure, SameSite на cookie | Установлены |
| Logout инвалидирует сессию/токен | Да |
| SQL-injection в поле логина | Не работает |

---

## Связь с тестированием

Тестирование аутентификации — обязательная часть тест-плана для любого API:
- **Функциональные тесты** — все auth-flow работают корректно
- **Негативные тесты** — система правильно отклоняет невалидные запросы
- **Тесты безопасности** — защита от brute force, token leakage, session fixation
- **Интеграционные тесты** — взаимодействие с Authorization Server
- **Performance-тесты** — нагрузка на endpoint логина

---

## Типичные ошибки

1. **Не тестируют истечение токена** — токен может жить бесконечно, что является уязвимостью
2. **Путают 401 и 403** — 401 = не аутентифицирован, 403 = нет прав
3. **Не проверяют logout** — после logout токен/сессия должны быть невалидны
4. **Хардкодят токены в тестах** — токены истекают; нужно получать их динамически в `@BeforeEach`
5. **Не проверяют атрибуты cookie** — HttpOnly, Secure, SameSite
6. **Не тестируют ротацию токенов** — refresh token flow
7. **Игнорируют CORS** — неправильная конфигурация CORS может позволить злоумышленнику перехватить токен
8. **Не проверяют error messages** — сообщения об ошибках не должны раскрывать детали (какое поле неверно)

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Чем аутентификация отличается от авторизации?
2. Как работает Basic Authentication?
3. Что такое JWT? Из чего он состоит?
4. Как передать Bearer Token в REST Assured?
5. Что означает код 401? А 403?

### 🟡 Средний уровень
6. Опишите Client Credentials flow в OAuth 2.0.
7. Какие claims содержит JWT? Что такое `exp`, `sub`, `iss`?
8. Почему Basic Auth нельзя использовать без HTTPS?
9. Как тестировать истечение JWT-токена?
10. Какие атрибуты безопасности должны быть у cookie сессии?
11. Как вы тестируете ролевую авторизацию?

### 🔴 Продвинутый уровень
12. Как тестировать refresh token flow?
13. Какие уязвимости JWT вы знаете? (alg: none, key confusion)
14. Как тестировать защиту от brute force?
15. Чем OAuth 2.0 отличается от OpenID Connect?
16. Как организовать управление тестовыми токенами в CI/CD pipeline?
17. Как тестировать PKCE flow для мобильных приложений?

---

## Практические задания

### Задание 1: Полный auth flow
Напишите набор тестов для auth flow: регистрация → логин → получение токена → доступ к защищённому ресурсу → logout → проверка инвалидации токена.

### Задание 2: JWT-тесты
Создайте тесты для проверки: валидный токен, невалидный токен, истёкший токен, токен с изменённым payload, отсутствующий токен, пустой заголовок Authorization.

### Задание 3: Ролевая авторизация
Имея три роли (USER, MANAGER, ADMIN), напишите параметризованные тесты, проверяющие доступ к endpoint-ам: USER не может удалять ресурсы, MANAGER может редактировать, только ADMIN может управлять пользователями.

### Задание 4: Чек-лист безопасности
Составьте полный чек-лист безопасности для API аутентификации. Автоматизируйте минимум 10 проверок.

---

## Дополнительные ресурсы

- [JWT.io — декодер и отладчик JWT](https://jwt.io)
- [OAuth 2.0 Simplified (Aaron Parecki)](https://www.oauth.com)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OWASP API Security Top 10](https://owasp.org/API-Security/)
- [RFC 7519 — JSON Web Token](https://datatracker.ietf.org/doc/html/rfc7519)
- [RFC 6749 — OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc6749)
