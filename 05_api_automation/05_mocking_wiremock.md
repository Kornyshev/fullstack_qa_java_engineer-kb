# Мокирование API с WireMock

## Обзор

WireMock — это инструмент для создания mock-серверов HTTP-сервисов. Он позволяет имитировать поведение внешних API, что критически важно при тестировании в изолированной среде. Для QA-инженера WireMock — незаменимый инструмент, когда внешний сервис недоступен, нестабилен, платный или когда нужно смоделировать определённые сценарии (ошибки, задержки).

### Зачем мокировать внешние сервисы

| Проблема | Решение с WireMock |
|----------|-------------------|
| Внешний сервис недоступен (downtime) | Мок всегда доступен |
| Внешний сервис платный (API calls) | Мок бесплатен |
| Нужно смоделировать ошибку (500, timeout) | Мок может вернуть любой ответ |
| Внешний сервис медленный | Мок отвечает мгновенно |
| Тесты нестабильны из-за внешних зависимостей | Мок детерминирован |
| Нужно протестировать граничные случаи | Мок полностью контролируем |

---

## Настройка WireMock

### Maven-зависимости

```xml
<dependencies>
    <!-- WireMock -->
    <dependency>
        <groupId>org.wiremock</groupId>
        <artifactId>wiremock-standalone</artifactId>
        <version>3.5.4</version>
        <scope>test</scope>
    </dependency>

    <!-- JUnit 5 -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.10.2</version>
        <scope>test</scope>
    </dependency>

    <!-- REST Assured для тестирования через mock -->
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>rest-assured</artifactId>
        <version>5.4.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Запуск WireMock как JUnit 5 Extension

```java
import com.github.tomakehurst.wiremock.junit5.WireMockExtension;
import com.github.tomakehurst.wiremock.junit5.WireMockTest;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static com.github.tomakehurst.wiremock.core.WireMockConfiguration.wireMockConfig;

// Вариант 1: аннотация @WireMockTest (автоматическая настройка)
@WireMockTest(httpPort = 8089)
class SimpleWireMockTest {

    @Test
    void shouldReturnStubbedResponse() {
        // Настройка стаба
        stubFor(get(urlEqualTo("/api/users/1"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {
                        "id": 1,
                        "name": "Тестовый пользователь"
                    }
                    """)));

        // Тест через REST Assured
        given()
            .baseUri("http://localhost:8089")
        .when()
            .get("/api/users/1")
        .then()
            .statusCode(200)
            .body("name", equalTo("Тестовый пользователь"));
    }
}

// Вариант 2: WireMockExtension (больше контроля)
class AdvancedWireMockTest {

    @RegisterExtension
    static WireMockExtension wireMock = WireMockExtension.newInstance()
        .options(wireMockConfig()
            .dynamicPort()               // Случайный порт (избегаем конфликтов)
            .globalTemplating(true))     // Включить шаблонизацию ответов
        .build();

    @Test
    void shouldUseDynamicPort() {
        wireMock.stubFor(get("/api/health")
            .willReturn(ok()
                .withBody("{\"status\": \"UP\"}")));

        given()
            .baseUri(wireMock.baseUrl())  // Получаем URL с динамическим портом
        .when()
            .get("/api/health")
        .then()
            .statusCode(200)
            .body("status", equalTo("UP"));
    }
}
```

### Standalone-режим

WireMock можно запустить как отдельный процесс:

```bash
# Скачать и запустить standalone jar
java -jar wiremock-standalone-3.5.4.jar --port 8089

# Стабы загружаются из директории mappings/
# Файлы ответов — из директории __files/
```

---

## Stubbing (настройка стабов)

### Основные HTTP-методы

```java
// GET
stubFor(get(urlEqualTo("/api/users"))
    .willReturn(aResponse()
        .withStatus(200)
        .withHeader("Content-Type", "application/json")
        .withBody("[{\"id\": 1, \"name\": \"Иван\"}]")));

// POST
stubFor(post(urlEqualTo("/api/users"))
    .withHeader("Content-Type", equalTo("application/json"))
    .withRequestBody(containing("\"name\""))
    .willReturn(aResponse()
        .withStatus(201)
        .withHeader("Content-Type", "application/json")
        .withHeader("Location", "/api/users/42")
        .withBody("{\"id\": 42, \"name\": \"Новый пользователь\"}")));

// PUT
stubFor(put(urlEqualTo("/api/users/1"))
    .willReturn(aResponse()
        .withStatus(200)
        .withBody("{\"id\": 1, \"name\": \"Обновлённый\"}")));

// DELETE
stubFor(delete(urlEqualTo("/api/users/1"))
    .willReturn(aResponse()
        .withStatus(204)));

// PATCH
stubFor(patch(urlEqualTo("/api/users/1"))
    .willReturn(aResponse()
        .withStatus(200)
        .withBody("{\"id\": 1, \"name\": \"Частично обновлённый\"}")));
```

### Удобные методы ответов

```java
// Краткие варианты willReturn
stubFor(get("/api/health").willReturn(ok()));                          // 200
stubFor(get("/api/health").willReturn(ok("{\"status\": \"UP\"}")));    // 200 с телом
stubFor(post("/api/users").willReturn(created()));                     // 201
stubFor(get("/api/missing").willReturn(notFound()));                   // 404
stubFor(post("/api/bad").willReturn(badRequest()));                    // 400
stubFor(get("/api/error").willReturn(serverError()));                  // 500
stubFor(get("/api/forbidden").willReturn(forbidden()));                // 403
stubFor(get("/api/unauthorized").willReturn(unauthorized()));          // 401
stubFor(delete("/api/users/1").willReturn(noContent()));               // 204
```

---

## Request Matching (сопоставление запросов)

### URL Matching

```java
// Точное совпадение URL
stubFor(get(urlEqualTo("/api/users/1"))
    .willReturn(ok()));

// Совпадение пути (без query parameters)
stubFor(get(urlPathEqualTo("/api/users"))
    .willReturn(ok()));

// URL по regex
stubFor(get(urlMatching("/api/users/[0-9]+"))
    .willReturn(ok()));

// Путь по regex
stubFor(get(urlPathMatching("/api/users/.*"))
    .willReturn(ok()));
```

### Header Matching

```java
stubFor(get(urlEqualTo("/api/resource"))
    // Точное совпадение заголовка
    .withHeader("Accept", equalTo("application/json"))
    // Заголовок содержит подстроку
    .withHeader("Authorization", containing("Bearer"))
    // Заголовок соответствует regex
    .withHeader("X-Request-Id", matching("[a-f0-9-]{36}"))
    // Заголовок отсутствует
    .withHeader("X-Deprecated", absent())
    .willReturn(ok()));
```

### Query Parameter Matching

```java
stubFor(get(urlPathEqualTo("/api/users"))
    .withQueryParam("page", equalTo("1"))
    .withQueryParam("size", equalTo("10"))
    .withQueryParam("sort", containing("name"))
    .willReturn(ok()
        .withBody("{\"content\": [], \"totalPages\": 5}")));
```

### Request Body Matching

```java
// Тело содержит подстроку
stubFor(post(urlEqualTo("/api/users"))
    .withRequestBody(containing("\"name\""))
    .willReturn(created()));

// Тело точно совпадает с JSON
stubFor(post(urlEqualTo("/api/users"))
    .withRequestBody(equalToJson("""
        {
            "name": "Иван",
            "email": "ivan@test.com"
        }
        """))
    .willReturn(created()));

// JSON с игнорированием порядка полей и лишних полей
stubFor(post(urlEqualTo("/api/users"))
    .withRequestBody(equalToJson("""
        { "name": "Иван" }
        """, true, true))  // ignoreArrayOrder=true, ignoreExtraElements=true
    .willReturn(created()));

// Тело соответствует JSON Path
stubFor(post(urlEqualTo("/api/users"))
    .withRequestBody(matchingJsonPath("$.name"))           // Поле name существует
    .withRequestBody(matchingJsonPath("$.age", equalTo("25")))  // age == 25
    .willReturn(created()));

// Тело соответствует regex
stubFor(post(urlEqualTo("/api/users"))
    .withRequestBody(matching(".*\"email\":\".*@.*\".*"))
    .willReturn(created()));
```

### Приоритеты стабов

```java
// При совпадении нескольких стабов побеждает тот, у которого выше приоритет (меньшее число)
stubFor(get(urlMatching("/api/users/.*"))
    .atPriority(10)    // Низкий приоритет — общий случай
    .willReturn(ok().withBody("{\"role\": \"USER\"}")));

stubFor(get(urlEqualTo("/api/users/1"))
    .atPriority(1)     // Высокий приоритет — конкретный случай
    .willReturn(ok().withBody("{\"role\": \"ADMIN\"}")));
```

---

## Verification (проверка вызовов)

WireMock может проверять, что определённые запросы были сделаны.

```java
@Test
void shouldVerifyRequestWasMade() {
    // Настройка стаба
    stubFor(post(urlEqualTo("/api/notifications"))
        .willReturn(ok()));

    // Действие: вызываем сервис, который должен отправить уведомление
    given()
        .baseUri(wireMock.baseUrl())
        .contentType(ContentType.JSON)
        .body("{\"message\": \"Hello\"}")
    .when()
        .post("/api/notifications");

    // Верификация: проверяем, что запрос был сделан
    verify(postRequestedFor(urlEqualTo("/api/notifications"))
        .withHeader("Content-Type", equalTo("application/json"))
        .withRequestBody(containing("Hello")));

    // Проверка количества вызовов
    verify(1, postRequestedFor(urlEqualTo("/api/notifications")));

    // Проверка, что запрос НЕ был сделан
    verify(0, getRequestedFor(urlEqualTo("/api/other")));

    // Проверка, что запрос был сделан хотя бы раз
    verify(moreThanOrExactly(1), postRequestedFor(urlEqualTo("/api/notifications")));
}
```

### Получение списка всех запросов

```java
@Test
void shouldInspectAllRequests() {
    // ... выполнение тестов ...

    // Получить все запросы, сделанные к mock-серверу
    List<LoggedRequest> requests = findAll(
        postRequestedFor(urlPathMatching("/api/.*")));

    assertEquals(3, requests.size());

    // Инспекция конкретного запроса
    LoggedRequest firstRequest = requests.get(0);
    String body = firstRequest.getBodyAsString();
    String authHeader = firstRequest.getHeader("Authorization");
}
```

---

## Scenarios (имитация состояний)

Scenarios позволяют создавать stateful-поведение: ответ зависит от предыдущих запросов.

```java
@Test
void shouldSimulateStatefulBehavior() {
    // Начальное состояние: товар в наличии
    stubFor(get(urlEqualTo("/api/products/1"))
        .inScenario("Покупка товара")
        .whenScenarioStateIs(Scenario.STARTED)       // Начальное состояние
        .willReturn(ok().withBody("""
            { "id": 1, "name": "Ноутбук", "inStock": true }
            """))
        .willSetStateTo("Товар куплен"));             // Переход в новое состояние

    // После покупки: товара нет в наличии
    stubFor(get(urlEqualTo("/api/products/1"))
        .inScenario("Покупка товара")
        .whenScenarioStateIs("Товар куплен")           // Новое состояние
        .willReturn(ok().withBody("""
            { "id": 1, "name": "Ноутбук", "inStock": false }
            """)));

    // Стаб для покупки
    stubFor(post(urlEqualTo("/api/products/1/buy"))
        .inScenario("Покупка товара")
        .whenScenarioStateIs(Scenario.STARTED)
        .willReturn(ok())
        .willSetStateTo("Товар куплен"));

    // Тест: сначала товар в наличии
    given().baseUri(wireMock.baseUrl())
        .when().get("/api/products/1")
        .then().body("inStock", equalTo(true));

    // Покупаем товар
    given().baseUri(wireMock.baseUrl())
        .when().post("/api/products/1/buy")
        .then().statusCode(200);

    // После покупки — товара нет
    given().baseUri(wireMock.baseUrl())
        .when().get("/api/products/1")
        .then().body("inStock", equalTo(false));
}
```

---

## Response Templating (шаблонизация ответов)

Шаблонизация позволяет создавать динамические ответы на основе данных запроса.

```java
@RegisterExtension
static WireMockExtension wireMock = WireMockExtension.newInstance()
    .options(wireMockConfig()
        .dynamicPort()
        .globalTemplating(true))   // Включить шаблонизацию
    .build();

@Test
void shouldUseResponseTemplating() {
    // Ответ возвращает данные из запроса
    wireMock.stubFor(get(urlPathMatching("/api/users/(.+)"))
        .willReturn(ok()
            .withHeader("Content-Type", "application/json")
            .withBody("""
                {
                    "id": {{request.pathSegments.[2]}},
                    "requestUrl": "{{request.url}}",
                    "requestMethod": "{{request.method}}",
                    "timestamp": "{{now}}"
                }
                """)));

    // Запрос к /api/users/42 вернёт id=42
    given()
        .baseUri(wireMock.baseUrl())
    .when()
        .get("/api/users/42")
    .then()
        .body("id", equalTo("42"))
        .body("requestMethod", equalTo("GET"));
}

@Test
void shouldTemplateFromRequestBody() {
    // Эхо: возвращаем данные из тела запроса
    wireMock.stubFor(post(urlEqualTo("/api/echo"))
        .willReturn(ok()
            .withHeader("Content-Type", "application/json")
            .withBody("""
                {
                    "received_name": "{{jsonPath request.body '$.name'}}",
                    "received_email": "{{jsonPath request.body '$.email'}}",
                    "processed": true
                }
                """)));

    given()
        .baseUri(wireMock.baseUrl())
        .contentType(ContentType.JSON)
        .body("{\"name\": \"Иван\", \"email\": \"ivan@test.com\"}")
    .when()
        .post("/api/echo")
    .then()
        .body("received_name", equalTo("Иван"))
        .body("processed", equalTo(true));
}
```

---

## Fault Simulation (имитация сбоев)

### Задержки

```java
// Фиксированная задержка
stubFor(get(urlEqualTo("/api/slow"))
    .willReturn(ok()
        .withFixedDelay(5000)));  // 5 секунд

// Случайная задержка (реалистичнее)
stubFor(get(urlEqualTo("/api/variable"))
    .willReturn(ok()
        .withUniformRandomDelay(1000, 3000)));  // От 1 до 3 секунд

// Задержка по шаблону (lognormal — имитация реальной сети)
stubFor(get(urlEqualTo("/api/realistic"))
    .willReturn(ok()
        .withLogNormalRandomDelay(90, 0.1)));  // Медиана 90ms
```

### Ошибки

```java
// Обрыв соединения
stubFor(get(urlEqualTo("/api/connection-reset"))
    .willReturn(aResponse()
        .withFault(Fault.CONNECTION_RESET_BY_PEER)));

// Пустой ответ
stubFor(get(urlEqualTo("/api/empty"))
    .willReturn(aResponse()
        .withFault(Fault.EMPTY_RESPONSE)));

// Мусорные данные вместо ответа
stubFor(get(urlEqualTo("/api/garbage"))
    .willReturn(aResponse()
        .withFault(Fault.MALFORMED_RESPONSE_CHUNK)));

// Комбинация: задержка + ошибка сервера
stubFor(get(urlEqualTo("/api/timeout-then-error"))
    .willReturn(serverError()
        .withFixedDelay(10000)   // 10 секунд, потом 500
        .withBody("{\"error\": \"Internal Server Error\"}")));
```

### Тестирование устойчивости

```java
@Test
@DisplayName("Клиент корректно обрабатывает timeout")
void shouldHandleTimeout() {
    stubFor(get(urlEqualTo("/api/data"))
        .willReturn(ok()
            .withFixedDelay(30000)));  // 30 секунд

    // REST Assured с настроенным timeout
    given()
        .baseUri(wireMock.baseUrl())
        .config(RestAssured.config()
            .httpClient(httpClientConfig()
                .setParam("http.connection.timeout", 5000)
                .setParam("http.socket.timeout", 5000)))
    .when()
        .get("/api/data")
    .then()
        .time(greaterThan(4000L));  // Проверяем, что запрос занял более 4 секунд
}

@Test
@DisplayName("Клиент корректно обрабатывает 503 и делает retry")
void shouldRetryOnServiceUnavailable() {
    // Первый запрос — ошибка, второй — успех
    stubFor(get(urlEqualTo("/api/data"))
        .inScenario("Retry")
        .whenScenarioStateIs(Scenario.STARTED)
        .willReturn(serviceUnavailable())
        .willSetStateTo("Recovered"));

    stubFor(get(urlEqualTo("/api/data"))
        .inScenario("Retry")
        .whenScenarioStateIs("Recovered")
        .willReturn(ok().withBody("{\"data\": \"ok\"}")));

    // Тестируем, что клиент пережил 503 и получил данные после retry
    // (зависит от реализации retry-логики в клиенте)
}
```

---

## MockServer (краткий обзор)

MockServer — альтернатива WireMock с похожей функциональностью.

```xml
<dependency>
    <groupId>org.mock-server</groupId>
    <artifactId>mockserver-netty</artifactId>
    <version>5.15.0</version>
    <scope>test</scope>
</dependency>
```

```java
import org.mockserver.integration.ClientAndServer;
import static org.mockserver.model.HttpRequest.request;
import static org.mockserver.model.HttpResponse.response;

class MockServerExample {

    private ClientAndServer mockServer;

    @BeforeEach
    void setup() {
        mockServer = ClientAndServer.startClientAndServer(8089);
    }

    @AfterEach
    void teardown() {
        mockServer.stop();
    }

    @Test
    void shouldStubWithMockServer() {
        mockServer.when(
            request()
                .withMethod("GET")
                .withPath("/api/users/1")
        ).respond(
            response()
                .withStatusCode(200)
                .withHeader("Content-Type", "application/json")
                .withBody("{\"id\": 1, \"name\": \"Тест\"}")
        );

        given()
            .baseUri("http://localhost:8089")
        .when()
            .get("/api/users/1")
        .then()
            .statusCode(200)
            .body("name", equalTo("Тест"));
    }
}
```

### WireMock vs MockServer

| Аспект | WireMock | MockServer |
|--------|----------|------------|
| Популярность | Очень высокая | Высокая |
| JUnit-интеграция | Нативная (Extension) | Через Rule/Extension |
| Шаблонизация | Встроенная (Handlebars) | Через Velocity |
| Proxy-режим | Да | Да |
| HTTPS | Да | Да |
| Docker | Да | Да |
| Документация | Отличная | Хорошая |

---

## Когда мокировать, а когда использовать реальный сервис

| Ситуация | Рекомендация |
|---------|--------------|
| Unit/компонентные тесты | Мокировать |
| Внешний платный сервис | Мокировать |
| Внешний нестабильный сервис | Мокировать |
| Тестирование error-handling | Мокировать |
| Тестирование timeout/retry | Мокировать |
| E2E/acceptance-тесты | Реальный сервис |
| Тестирование интеграции | Реальный сервис (или контрактные тесты) |
| Performance-тесты | Реальный сервис |
| Smoke-тесты на staging | Реальный сервис |

---

## Связь с тестированием

WireMock используется QA-инженерами для:
- **Изоляция тестов** — независимость от внешних сервисов
- **Тестирование граничных случаев** — имитация ошибок, которые трудно воспроизвести
- **Ускорение тестов** — мок отвечает мгновенно
- **Тестирование устойчивости** — проверка поведения при сбоях (circuit breaker, retry)
- **Параллельный запуск** — каждый тест может иметь свой mock-сервер на отдельном порту

---

## Типичные ошибки

1. **Мокируют всё подряд** — E2E-тесты теряют смысл, если все зависимости замокированы
2. **Стабы не соответствуют реальному API** — мок возвращает структуру, отличную от реального сервиса
3. **Не используют dynamic port** — фиксированный порт вызывает конфликты при параллельном запуске
4. **Забывают про verify** — не проверяют, что запрос к моку действительно был сделан
5. **Не очищают стабы между тестами** — стабы из предыдущего теста влияют на следующий
6. **Слишком строгий matching** — тест ломается при незначительных изменениях запроса
7. **Не тестируют negative-сценарии** — мок всегда возвращает 200, хотя реальный сервис может вернуть ошибку
8. **Не обновляют стабы при изменении API** — мок становится устаревшим

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Что такое WireMock и зачем он нужен?
2. В каких ситуациях вы бы использовали мокирование API?
3. Как настроить простой стаб для GET-запроса?
4. Чем мокирование отличается от использования реального API?

### 🟡 Средний уровень
5. Как работает request matching в WireMock? Какие варианты сопоставления есть?
6. Что такое verify в WireMock и зачем нужна верификация?
7. Как использовать Scenarios для имитации stateful-поведения?
8. Как имитировать задержку или сбой сети?
9. Как организовать стабы при параллельном запуске тестов?
10. В чём разница между `urlEqualTo` и `urlPathEqualTo`?

### 🔴 Продвинутый уровень
11. Как работает Response Templating в WireMock?
12. Как настроить WireMock в Docker для использования в CI/CD?
13. Чем WireMock отличается от MockServer? Когда что выбирать?
14. Как использовать WireMock как Record/Playback proxy?
15. Как организовать управление стабами для большого количества микросервисов?

---

## Практические задания

### Задание 1: Базовые стабы
Создайте WireMock-стабы для CRUD-операций над ресурсом "товар" (product): GET список, GET по ID, POST создание, PUT обновление, DELETE удаление. Напишите тесты с REST Assured.

### Задание 2: Имитация ошибок
Используя WireMock, смоделируйте: 500 Internal Server Error, задержку 10 секунд, обрыв соединения. Напишите тесты, проверяющие, что ваш клиент корректно обрабатывает эти ситуации.

### Задание 3: Scenarios
Реализуйте сценарий: корзина покупок. Сначала корзина пуста → добавляем товар → корзина содержит товар → оформляем заказ → корзина снова пуста. Используйте Scenarios.

### Задание 4: Response Templating
Создайте стаб, который динамически формирует ответ на основе query parameters запроса (пагинация: page, size) и path parameter (ID ресурса).

---

## Дополнительные ресурсы

- [WireMock Official Documentation](https://wiremock.org/docs/)
- [WireMock GitHub](https://github.com/wiremock/wiremock)
- [WireMock Stubbing Guide](https://wiremock.org/docs/stubbing/)
- [WireMock Request Matching](https://wiremock.org/docs/request-matching/)
- [WireMock Response Templating](https://wiremock.org/docs/response-templating/)
- [MockServer Documentation](https://www.mock-server.com/)
