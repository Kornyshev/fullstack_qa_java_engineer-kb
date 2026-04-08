# Слои тестирования

## Обзор

Современное приложение — это не просто веб-страница. Это UI, REST API, база данных, очереди сообщений,
внешние интеграции. Тестировать всё исключительно через UI — дорого, медленно и хрупко. Тестировать только API —
значит пропустить баги отображения. Зрелый тестовый фреймворк объединяет **несколько слоёв тестирования**
в единую систему, позволяя выбирать оптимальный уровень для каждой проверки. В этом разделе — как
комбинировать UI, API и DB слои в одном фреймворке и когда использовать каждый из них.

---

## Три слоя тестирования

```
┌────────────────────────────────────────────┐
│              UI Layer (Браузер)             │  Медленно, хрупко, но видит то, что видит пользователь
├────────────────────────────────────────────┤
│              API Layer (HTTP)              │  Быстро, стабильно, проверяет бизнес-логику
├────────────────────────────────────────────┤
│              DB Layer (SQL)                │  Мгновенно, проверяет данные напрямую
└────────────────────────────────────────────┘
```

### UI Layer — Слой пользовательского интерфейса

**Что тестируем:** визуальное отображение, пользовательские сценарии, клиентская валидация, навигация.

**Инструменты:** Selenide, Selenium, Playwright.

**Преимущества:**
- Проверяет то, что видит реальный пользователь.
- Находит баги вёрстки, JS-ошибки, проблемы UX.

**Недостатки:**
- Медленные (секунды на каждое действие).
- Хрупкие (ломаются при любом изменении UI).
- Дорогие в поддержке.

```java
// UI-тест: проверяем, что пользователь видит результаты поиска
@Test
@DisplayName("Поиск товара отображает релевантные результаты")
void searchDisplaysRelevantResults() {
    open("/catalog");
    $("[data-testid='search-input']").setValue("iPhone").pressEnter();
    $$(".product-card").shouldHave(CollectionCondition.sizeGreaterThan(0));
    $$(".product-card .product-name")
            .first()
            .shouldHave(Condition.text("iPhone"));
}
```

### API Layer — Слой прикладного интерфейса

**Что тестируем:** бизнес-логику, валидацию на сервере, статус-коды, структуру ответов, авторизацию.

**Инструменты:** REST Assured, HttpClient, OkHttp.

**Преимущества:**
- Быстрые (миллисекунды).
- Стабильные (не зависят от рендеринга UI).
- Покрывают бизнес-логику напрямую.

**Недостатки:**
- Не видят UI-баги.
- Требуют знания API-контрактов.

```java
// API-тест: проверяем поиск через REST API
@Test
@DisplayName("Поиск через API возвращает товары по ключевому слову")
void searchApiReturnsProductsByKeyword() {
    given()
        .queryParam("q", "iPhone")
        .queryParam("limit", 10)
    .when()
        .get("/api/v1/products/search")
    .then()
        .statusCode(200)
        .body("items", hasSize(greaterThan(0)))
        .body("items[0].name", containsString("iPhone"))
        .body("total", greaterThan(0));
}
```

### DB Layer — Слой базы данных

**Что тестируем:** корректность сохранения данных, целостность, триггеры, хранимые процедуры.

**Инструменты:** JDBC, JdbcTemplate, jOOQ, Hibernate.

**Преимущества:**
- Мгновенные проверки.
- Доступ к данным, недоступным через API/UI.
- Проверка побочных эффектов (аудит-логи, статусы, timestamps).

**Недостатки:**
- Тесная связь со схемой БД (хрупкость при миграциях).
- Не проверяют бизнес-логику приложения.

```java
// DB-проверка: данные корректно сохранились в базу
@Test
@DisplayName("Заказ сохраняется в БД с правильным статусом")
void orderIsSavedWithCorrectStatus() {
    // Подготовка: создаём заказ через API
    int orderId = apiClient.createOrder(orderData);

    // Проверка: в БД заказ имеет правильный статус
    String status = jdbcTemplate.queryForObject(
        "SELECT status FROM orders WHERE id = ?",
        String.class,
        orderId
    );
    assertThat(status).isEqualTo("CREATED");
}
```

---

## Когда использовать какой слой

### Матрица принятия решений

| Что проверяем | UI | API | DB |
|---------------|:--:|:---:|:--:|
| Отображение элементов | + | - | - |
| Бизнес-правила валидации | - | + | - |
| Сохранение данных | - | - | + |
| Навигация и переходы | + | - | - |
| Авторизация и роли | - | + | - |
| Пользовательский сценарий (E2E) | + | +/- | +/- |
| Производительность API | - | + | - |
| Целостность данных | - | - | + |
| Клиентская валидация (JS) | + | - | - |
| Серверная валидация | - | + | - |

### Правило: тестируй на самом низком подходящем уровне

```
Можно проверить через DB?    → Проверяй через DB
Нет → Можно через API?      → Проверяй через API
Нет → Только через UI?      → Проверяй через UI
```

Это следует из **пирамиды тестирования**: чем ниже уровень, тем быстрее и стабильнее тесты.

---

## Кросс-слойные тесты

Самая мощная техника — комбинирование слоёв в одном тесте:

### Паттерн: Setup via API → Action via UI → Verify via DB

```java
@Test
@DisplayName("Пользователь может оформить заказ")
void userCanPlaceOrder() {
    // 1. SETUP через API — быстро создаём пользователя и товар
    User user = apiClient.createUser(UserFactory.randomUser());
    Product product = apiClient.createProduct(ProductFactory.inStock());
    String authToken = apiClient.login(user.getUsername(), user.getPassword());

    // 2. ACTION через UI — пользовательский сценарий
    // Устанавливаем cookies с токеном, чтобы не логиниться через UI
    WebDriverRunner.getWebDriver().manage()
        .addCookie(new Cookie("auth_token", authToken));
    open("/catalog");

    catalogPage.searchProduct(product.getName());
    catalogPage.addToCart(product.getName());
    cartPage.open();
    cartPage.proceedToCheckout();
    checkoutPage.fillAddress("ул. Тестовая, д. 1");
    checkoutPage.confirmOrder();
    checkoutPage.verifyOrderConfirmation();

    // 3. VERIFY через DB — проверяем, что данные корректно сохранились
    Order order = dbHelper.getLastOrderByUserId(user.getId());
    assertAll(
        () -> assertThat(order.getStatus()).isEqualTo("CONFIRMED"),
        () -> assertThat(order.getTotalAmount()).isEqualTo(product.getPrice()),
        () -> assertThat(order.getDeliveryAddress()).isEqualTo("ул. Тестовая, д. 1"),
        () -> assertThat(order.getCreatedAt()).isCloseTo(
            Instant.now(), within(1, ChronoUnit.MINUTES)
        )
    );
}
```

### Паттерн: Setup via DB → Action via API → Verify via API

```java
@Test
@DisplayName("API корректно возвращает данные пользователя после обновления")
void apiReturnsUpdatedUserData() {
    // 1. SETUP через DB — напрямую вставляем пользователя
    int userId = dbHelper.insertUser(
        "testuser", "test@email.com", "ACTIVE"
    );

    // 2. ACTION через API — обновляем данные
    given()
        .contentType(ContentType.JSON)
        .body(Map.of("email", "updated@email.com"))
    .when()
        .patch("/api/v1/users/" + userId)
    .then()
        .statusCode(200);

    // 3. VERIFY через API — проверяем, что данные обновились
    given()
    .when()
        .get("/api/v1/users/" + userId)
    .then()
        .statusCode(200)
        .body("email", equalTo("updated@email.com"));
}
```

---

## Абстракции слоёв в коде

### Интерфейсы для каждого слоя

```java
// Абстракция: работа с пользователями независимо от слоя
public interface UserService {
    User createUser(User user);
    User getUserById(int id);
    void deleteUser(int id);
    void updateUser(int id, User updatedData);
}

// Реализация через API
public class UserApiService implements UserService {

    @Override
    public User createUser(User user) {
        return given()
            .contentType(ContentType.JSON)
            .body(user)
        .when()
            .post("/api/v1/users")
        .then()
            .statusCode(201)
            .extract().as(User.class);
    }

    @Override
    public User getUserById(int id) {
        return given()
        .when()
            .get("/api/v1/users/" + id)
        .then()
            .statusCode(200)
            .extract().as(User.class);
    }

    @Override
    public void deleteUser(int id) {
        given()
        .when()
            .delete("/api/v1/users/" + id)
        .then()
            .statusCode(204);
    }

    @Override
    public void updateUser(int id, User updatedData) {
        given()
            .contentType(ContentType.JSON)
            .body(updatedData)
        .when()
            .put("/api/v1/users/" + id)
        .then()
            .statusCode(200);
    }
}

// Реализация через DB (для прямой работы с данными)
public class UserDbService implements UserService {

    private final JdbcTemplate jdbc;

    public UserDbService(JdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    @Override
    public User createUser(User user) {
        int id = jdbc.queryForObject(
            "INSERT INTO users (username, email, password) VALUES (?, ?, ?) RETURNING id",
            Integer.class,
            user.getUsername(), user.getEmail(), user.getPassword()
        );
        user.setId(id);
        return user;
    }

    @Override
    public User getUserById(int id) {
        return jdbc.queryForObject(
            "SELECT * FROM users WHERE id = ?",
            new UserRowMapper(),
            id
        );
    }

    @Override
    public void deleteUser(int id) {
        jdbc.update("DELETE FROM users WHERE id = ?", id);
    }

    @Override
    public void updateUser(int id, User updatedData) {
        jdbc.update(
            "UPDATE users SET email = ?, username = ? WHERE id = ?",
            updatedData.getEmail(), updatedData.getUsername(), id
        );
    }
}
```

### Вспомогательный класс для работы с БД

```java
// Хелпер для работы с базой данных в тестах
public class DbHelper {

    private final JdbcTemplate jdbc;

    public DbHelper(DataSource dataSource) {
        this.jdbc = new JdbcTemplate(dataSource);
    }

    // Получение последнего заказа пользователя
    public Order getLastOrderByUserId(int userId) {
        return jdbc.queryForObject(
            """
            SELECT * FROM orders
            WHERE user_id = ?
            ORDER BY created_at DESC
            LIMIT 1
            """,
            new OrderRowMapper(),
            userId
        );
    }

    // Очистка тестовых данных после теста
    public void cleanupTestData(int userId) {
        jdbc.update("DELETE FROM orders WHERE user_id = ?", userId);
        jdbc.update("DELETE FROM cart_items WHERE user_id = ?", userId);
        jdbc.update("DELETE FROM users WHERE id = ?", userId);
    }

    // Проверка наличия записи в аудит-логе
    public boolean auditLogExists(String action, int entityId) {
        Integer count = jdbc.queryForObject(
            "SELECT COUNT(*) FROM audit_log WHERE action = ? AND entity_id = ?",
            Integer.class,
            action, entityId
        );
        return count != null && count > 0;
    }
}
```

---

## Стратегии подготовки и очистки данных

### Подготовка данных (Setup)

| Стратегия | Скорость | Надёжность | Когда использовать |
|-----------|----------|------------|-------------------|
| Через API | Быстро | Высокая | Основной способ — создание пользователей, товаров, заказов |
| Через DB | Мгновенно | Средняя | Массовые вставки, сложные связи, обход валидации API |
| Через UI | Медленно | Низкая | Только если API/DB недоступны |
| Фикстуры (SQL-скрипты) | Мгновенно | Средняя | Эталонное состояние БД (справочники, роли) |

### Очистка данных (Cleanup)

| Стратегия | Описание |
|-----------|----------|
| **DELETE через DB** | Удаляем созданные в тесте записи по ID |
| **TRUNCATE** | Полная очистка таблиц (для тестовой БД) |
| **Транзакционный откат** | Оборачиваем тест в транзакцию, делаем ROLLBACK |
| **Docker container** | Поднимаем свежую БД для каждого прогона |
| **API DELETE** | Удаляем через API (если есть эндпоинт) |

### Пример: setup и cleanup

```java
public abstract class BaseIntegrationTest {

    protected static final UserApiService userApi = new UserApiService();
    protected static final DbHelper dbHelper = new DbHelper(DataSourceProvider.get());

    // Хранилище созданных данных для последующей очистки
    private final List<Integer> createdUserIds = new ArrayList<>();

    // Создание пользователя с автоматическим отслеживанием для очистки
    protected User createTestUser() {
        User user = userApi.createUser(UserFactory.randomUser());
        createdUserIds.add(user.getId());
        return user;
    }

    // Очистка после каждого теста
    @AfterEach
    void cleanup() {
        createdUserIds.forEach(id -> {
            try {
                dbHelper.cleanupTestData(id);
            } catch (Exception e) {
                log.warn("Не удалось очистить данные пользователя {}: {}",
                    id, e.getMessage());
            }
        });
        createdUserIds.clear();
    }
}
```

---

## Архитектура кросс-слойного фреймворка

```
src/test/java/com/company/project/
├── tests/
│   ├── ui/                          ← UI-тесты
│   │   ├── LoginUiTest.java
│   │   └── OrderUiTest.java
│   ├── api/                         ← API-тесты
│   │   ├── UserApiTest.java
│   │   └── OrderApiTest.java
│   ├── db/                          ← DB-тесты
│   │   └── DataIntegrityTest.java
│   └── e2e/                         ← Кросс-слойные E2E тесты
│       └── OrderFlowE2eTest.java
├── pages/                           ← Page Objects (UI-слой)
│   ├── LoginPage.java
│   ├── CatalogPage.java
│   └── CheckoutPage.java
├── api/                             ← API-клиенты
│   ├── clients/
│   │   ├── UserApiClient.java
│   │   └── OrderApiClient.java
│   └── specs/
│       └── RequestSpecs.java
├── db/                              ← DB-хелперы
│   ├── DbHelper.java
│   ├── DataSourceProvider.java
│   └── mappers/
│       ├── UserRowMapper.java
│       └── OrderRowMapper.java
├── base/                            ← Базовые классы для каждого слоя
│   ├── BaseUiTest.java
│   ├── BaseApiTest.java
│   ├── BaseDbTest.java
│   └── BaseIntegrationTest.java
└── config/
    └── ProjectConfig.java
```

---

## Связь с тестированием

Комбинация слоёв — это ключ к эффективной автоматизации:

- **80% тестов через API** — быстрые, стабильные, покрывают бизнес-логику.
- **15% тестов через UI** — критические пользовательские сценарии (happy path).
- **5% тестов через DB** — проверка целостности данных, аудит-логи, побочные эффекты.
- **Кросс-слойные тесты** — для критических E2E-сценариев (оплата, регистрация).

Это соответствует **пирамиде тестирования** и обеспечивает баланс скорости, стабильности и покрытия.

---

## Типичные ошибки

1. **Всё через UI** — медленно, хрупко, дорого. 500 UI-тестов запускаются 4 часа.
2. **Игнорирование DB-слоя** — баги в сохранении данных обнаруживаются только в продакшене.
3. **Setup через UI** — авторизация через форму перед каждым тестом вместо установки cookie.
4. **Нет абстракции между слоями** — логика API-вызовов размазана по тестам.
5. **Жёсткая привязка к схеме БД** — миграция ломает все DB-тесты.
6. **Нет cleanup** — тесты загрязняют БД, создавая зависимости между запусками.
7. **Тестирование UI-логики через API** — и наоборот, API-логики через UI.
8. **Одинаковые проверки на всех слоях** — дублирование не добавляет ценности.

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Какие слои тестирования вы знаете?
2. Когда стоит тестировать через UI, а когда через API?
3. Что такое пирамида тестирования?
4. Зачем проверять данные в базе, если API уже возвращает корректный ответ?
5. Как подготовить тестовые данные для UI-теста?

### 🟡 Средний уровень
6. Опишите паттерн «Setup via API → Action via UI → Verify via DB».
7. Как организовать cleanup тестовых данных?
8. Как авторизоваться в UI-тесте без прохождения формы логина?
9. Какие стратегии подготовки данных вы используете?
10. Как объединить API-тесты и UI-тесты в одном фреймворке?

### 🔴 Продвинутый уровень
11. Как организовать абстракции, чтобы один и тот же бизнес-сценарий можно было проверить через API или UI?
12. Как обеспечить изоляцию тестов при работе с общей базой данных?
13. Как тестировать асинхронные процессы (очереди, events) в кросс-слойных тестах?
14. Как организовать тестирование при микросервисной архитектуре с несколькими БД?
15. Какие trade-offs у транзакционного отката vs физического удаления тестовых данных?

---

## Практические задания

### Задание 1: Кросс-слойный тест
Напишите тест для сценария «Регистрация пользователя»:
- Setup: нет (чистый сценарий).
- Action: регистрация через UI (заполнение формы).
- Verify: проверка через API (GET /users/{id}) и DB (SELECT FROM users).

### Задание 2: Оптимизация
Возьмите 5 UI-тестов. Определите, какие проверки можно перенести на API/DB-слой.
Перепишите тесты, оставив через UI только визуальные проверки.

### Задание 3: DbHelper
Реализуйте `DbHelper` с методами:
- `insertUser()` — вставка пользователя
- `getOrdersByUserId()` — получение заказов
- `cleanupByUserId()` — каскадная очистка данных пользователя
- `assertAuditLogContains()` — проверка аудит-лога

### Задание 4: Абстракция слоёв
Создайте интерфейс `ProductService` с реализациями `ProductApiService` и `ProductDbService`.
Напишите тест, который создаёт товар через API и проверяет его наличие через DB.

---

## Дополнительные ресурсы

- [Martin Fowler — Test Pyramid](https://martinfowler.com/bliki/TestPyramid.html)
- [Test Automation University — API Testing](https://testautomationu.applitools.com/exploring-service-apis-through-test-automation/)
- [Baeldung — Spring JdbcTemplate](https://www.baeldung.com/spring-jdbc-jdbctemplate)
- [REST Assured Documentation](https://rest-assured.io/)
- [Selenide Documentation](https://selenide.org/documentation.html)
