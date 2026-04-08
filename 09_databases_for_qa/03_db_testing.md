# Тестирование через базу данных

## Обзор

API-ответы показывают, что система **говорит** тестировщику. База данных показывает, что
**на самом деле произошло**. API может вернуть `200 OK`, но если данные не записались корректно
в базу — это баг. QA-инженер, умеющий проверять состояние БД, обнаруживает дефекты, которые
невозможно поймать только через интерфейс или API.

В этом разделе рассматривается, когда и зачем QA обращается напрямую к базе данных, как работать
с JDBC в Java, какие паттерны использовать для верификации данных и как правильно
управлять тестовыми данными.

---

## Когда API-ответа недостаточно

### Сценарии, требующие проверки через БД

1. **API возвращает сокращённые данные.** Endpoint `/api/orders` может не включать внутренние поля
   вроде `internal_status`, `processing_queue` или `retry_count`. Эти данные видны только в БД.

2. **Проверка побочных эффектов.** При создании заказа система должна: записать заказ, обновить
   остатки на складе, создать запись в audit log, отправить событие. API вернёт только заказ.

3. **Soft delete.** API может вернуть `204 No Content` при удалении, но запись в базе не удаляется
   физически — устанавливается флаг `is_deleted = true`. Это нужно проверить.

4. **Асинхронные процессы.** После API-вызова фоновый процесс обрабатывает данные. Результат
   появляется в БД через некоторое время, и его не получить через API.

5. **Подготовка тестовых данных.** Создание сложных состояний через API требует множества вызовов.
   Прямая вставка в БД — быстрее и надёжнее для preconditions.

---

## JDBC в Java — основы

### Зависимость (Maven)

```xml
<!-- PostgreSQL драйвер -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <version>42.7.4</version>
    <scope>test</scope>
</dependency>
```

### Connection — подключение к БД

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

public class DbConnection {

    // Параметры подключения (в реальном проекте — из конфигурации)
    private static final String URL = "jdbc:postgresql://localhost:5432/testdb";
    private static final String USER = "test_user";
    private static final String PASSWORD = "test_pass";

    public static Connection getConnection() throws SQLException {
        return DriverManager.getConnection(URL, USER, PASSWORD);
    }
}
```

**Важно:** всегда используйте `try-with-resources` для автоматического закрытия ресурсов:

```java
// Правильно: ресурсы закрываются автоматически
try (Connection conn = DbConnection.getConnection()) {
    // работа с БД
}

// Неправильно: утечка соединений
Connection conn = DbConnection.getConnection();
// ... если возникнет исключение, соединение не закроется
```

### Statement — простые запросы

```java
import java.sql.Connection;
import java.sql.ResultSet;
import java.sql.Statement;

// Получение данных через Statement (для статических запросов без параметров)
try (Connection conn = DbConnection.getConnection();
     Statement stmt = conn.createStatement();
     ResultSet rs = stmt.executeQuery("SELECT COUNT(*) FROM orders WHERE status = 'PENDING'")) {

    if (rs.next()) {
        int count = rs.getInt(1);
        System.out.println("Заказов в ожидании: " + count);
    }
}
```

### PreparedStatement — параметризованные запросы

```java
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;

// PreparedStatement защищает от SQL injection и улучшает производительность
String sql = "SELECT order_id, status, total_amount FROM orders WHERE customer_id = ? AND status = ?";

try (Connection conn = DbConnection.getConnection();
     PreparedStatement pstmt = conn.prepareStatement(sql)) {

    pstmt.setLong(1, customerId);     // Первый параметр — customer_id
    pstmt.setString(2, "COMPLETED");  // Второй параметр — status

    try (ResultSet rs = pstmt.executeQuery()) {
        while (rs.next()) {
            long orderId = rs.getLong("order_id");
            String status = rs.getString("status");
            BigDecimal amount = rs.getBigDecimal("total_amount");
            // Проверки...
        }
    }
}
```

**Никогда не подставляйте параметры через конкатенацию строк:**

```java
// ОПАСНО! SQL injection
String sql = "SELECT * FROM users WHERE email = '" + userEmail + "'";

// БЕЗОПАСНО: используйте PreparedStatement
String sql = "SELECT * FROM users WHERE email = ?";
```

### ResultSet — обработка результатов

```java
// Основные методы ResultSet
rs.next();                    // Переход к следующей строке (возвращает false, если строк больше нет)
rs.getString("column_name");  // Получить String по имени столбца
rs.getInt("column_name");     // Получить int
rs.getLong("column_name");    // Получить long
rs.getBigDecimal("column");   // Получить BigDecimal (для денежных значений)
rs.getTimestamp("column");    // Получить Timestamp
rs.getBoolean("column");     // Получить boolean
rs.wasNull();                 // Был ли последний прочитанный столбец NULL
```

### Модификация данных

```java
// INSERT — вставка тестовых данных
String insertSql = "INSERT INTO users (email, first_name, status) VALUES (?, ?, ?)";
try (Connection conn = DbConnection.getConnection();
     PreparedStatement pstmt = conn.prepareStatement(insertSql)) {

    pstmt.setString(1, "autotest@test.com");
    pstmt.setString(2, "AutoTest");
    pstmt.setString(3, "ACTIVE");

    int rowsInserted = pstmt.executeUpdate();  // Возвращает количество затронутых строк
    assert rowsInserted == 1 : "Ожидалась вставка одной строки";
}

// UPDATE — изменение состояния для теста
String updateSql = "UPDATE orders SET status = ? WHERE order_id = ?";
try (Connection conn = DbConnection.getConnection();
     PreparedStatement pstmt = conn.prepareStatement(updateSql)) {

    pstmt.setString(1, "SHIPPED");
    pstmt.setLong(2, orderId);

    int rowsUpdated = pstmt.executeUpdate();
    assert rowsUpdated == 1 : "Заказ не найден или не обновлён";
}

// DELETE — очистка тестовых данных
String deleteSql = "DELETE FROM users WHERE email LIKE ?";
try (Connection conn = DbConnection.getConnection();
     PreparedStatement pstmt = conn.prepareStatement(deleteSql)) {

    pstmt.setString(1, "%@test.com");

    int rowsDeleted = pstmt.executeUpdate();
    System.out.println("Удалено тестовых пользователей: " + rowsDeleted);
}
```

---

## Утилитный класс для тестов

```java
import java.sql.*;
import java.util.*;

/**
 * Утилитный класс для работы с БД в тестах.
 * Упрощает выполнение запросов и обработку результатов.
 */
public class DbUtils {

    private final String url;
    private final String user;
    private final String password;

    public DbUtils(String url, String user, String password) {
        this.url = url;
        this.user = user;
        this.password = password;
    }

    /**
     * Выполнить SELECT и вернуть результат как список Map.
     * Каждый Map — одна строка, ключ — имя столбца.
     */
    public List<Map<String, Object>> executeQuery(String sql, Object... params) {
        List<Map<String, Object>> results = new ArrayList<>();

        try (Connection conn = DriverManager.getConnection(url, user, password);
             PreparedStatement pstmt = conn.prepareStatement(sql)) {

            // Установить параметры
            for (int i = 0; i < params.length; i++) {
                pstmt.setObject(i + 1, params[i]);
            }

            try (ResultSet rs = pstmt.executeQuery()) {
                ResultSetMetaData meta = rs.getMetaData();
                int columnCount = meta.getColumnCount();

                while (rs.next()) {
                    Map<String, Object> row = new LinkedHashMap<>();
                    for (int i = 1; i <= columnCount; i++) {
                        row.put(meta.getColumnLabel(i), rs.getObject(i));
                    }
                    results.add(row);
                }
            }
        } catch (SQLException e) {
            throw new RuntimeException("Ошибка выполнения SQL: " + sql, e);
        }

        return results;
    }

    /**
     * Выполнить INSERT / UPDATE / DELETE.
     * Возвращает количество затронутых строк.
     */
    public int executeUpdate(String sql, Object... params) {
        try (Connection conn = DriverManager.getConnection(url, user, password);
             PreparedStatement pstmt = conn.prepareStatement(sql)) {

            for (int i = 0; i < params.length; i++) {
                pstmt.setObject(i + 1, params[i]);
            }

            return pstmt.executeUpdate();
        } catch (SQLException e) {
            throw new RuntimeException("Ошибка выполнения SQL: " + sql, e);
        }
    }

    /**
     * Получить одно значение (scalar query).
     */
    @SuppressWarnings("unchecked")
    public <T> T executeScalar(String sql, Object... params) {
        List<Map<String, Object>> rows = executeQuery(sql, params);
        if (rows.isEmpty()) return null;
        return (T) rows.get(0).values().iterator().next();
    }
}
```

---

## Паттерн: Setup через API, Verify в DB

Это основной паттерн для end-to-end тестирования с проверкой через базу.

```java
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;
import static io.restassured.RestAssured.given;

class OrderCreationDbTest {

    private final DbUtils db = new DbUtils(
        "jdbc:postgresql://localhost:5432/testdb", "test", "test"
    );

    @Test
    void createOrder_shouldPersistCorrectly() {
        // === SETUP: создаём заказ через API ===
        var response = given()
            .contentType("application/json")
            .body("""
                {
                    "customerId": 1,
                    "items": [
                        {"productId": 101, "quantity": 2},
                        {"productId": 102, "quantity": 1}
                    ]
                }
                """)
            .when()
            .post("/api/orders")
            .then()
            .statusCode(201)
            .extract().response();

        long orderId = response.jsonPath().getLong("orderId");

        // === VERIFY: проверяем данные в БД ===

        // Проверка записи заказа
        var orderRows = db.executeQuery(
            "SELECT status, total_amount, customer_id FROM orders WHERE order_id = ?",
            orderId
        );
        assertThat(orderRows).hasSize(1);
        assertThat(orderRows.get(0).get("status")).isEqualTo("PENDING");
        assertThat(orderRows.get(0).get("customer_id")).isEqualTo(1L);

        // Проверка позиций заказа
        var itemRows = db.executeQuery(
            "SELECT product_id, quantity FROM order_items WHERE order_id = ? ORDER BY product_id",
            orderId
        );
        assertThat(itemRows).hasSize(2);
        assertThat(itemRows.get(0).get("product_id")).isEqualTo(101);
        assertThat(itemRows.get(0).get("quantity")).isEqualTo(2);

        // Проверка побочного эффекта: audit log
        var auditRows = db.executeQuery(
            "SELECT action, entity_id FROM audit_log WHERE entity_type = 'ORDER' AND entity_id = ?",
            orderId
        );
        assertThat(auditRows)
            .extracting(row -> row.get("action"))
            .contains("CREATED");
    }
}
```

---

## Стратегии управления тестовыми данными

### 1. DELETE с WHERE — точечная очистка

```java
@AfterEach
void cleanup() {
    // Удаляем в правильном порядке (дочерние -> родительские)
    db.executeUpdate("DELETE FROM order_items WHERE order_id IN " +
        "(SELECT order_id FROM orders WHERE customer_id = ?)", TEST_CUSTOMER_ID);
    db.executeUpdate("DELETE FROM orders WHERE customer_id = ?", TEST_CUSTOMER_ID);
    db.executeUpdate("DELETE FROM users WHERE user_id = ?", TEST_CUSTOMER_ID);
}
```

**Плюсы:** точечное удаление, не затрагивает чужие данные.
**Минусы:** нужно знать порядок зависимостей, легко что-то пропустить.

### 2. TRUNCATE — полная очистка таблицы

```java
@BeforeEach
void resetDatabase() {
    // TRUNCATE быстрее DELETE, сбрасывает счётчики sequences
    db.executeUpdate("TRUNCATE TABLE order_items, orders, users CASCADE");
}
```

**Плюсы:** быстро, чисто, не нужно заботиться о порядке (CASCADE).
**Минусы:** удаляет **все** данные. Подходит только для изолированной тестовой БД.

### 3. Transaction rollback — откат транзакции

```java
import java.sql.Connection;
import org.junit.jupiter.api.*;

class TransactionalTest {

    private Connection conn;

    @BeforeEach
    void setup() throws Exception {
        conn = DbConnection.getConnection();
        conn.setAutoCommit(false);  // Начинаем транзакцию
    }

    @Test
    void testSomething() throws Exception {
        // Все INSERT/UPDATE/DELETE в рамках этой транзакции
        try (var pstmt = conn.prepareStatement(
                "INSERT INTO users (email) VALUES (?)")) {
            pstmt.setString(1, "transactional@test.com");
            pstmt.executeUpdate();
        }

        // Проверки в рамках той же транзакции видят вставленные данные
        try (var pstmt = conn.prepareStatement(
                "SELECT COUNT(*) FROM users WHERE email = ?")) {
            pstmt.setString(1, "transactional@test.com");
            try (var rs = pstmt.executeQuery()) {
                rs.next();
                Assertions.assertEquals(1, rs.getInt(1));
            }
        }
    }

    @AfterEach
    void rollback() throws Exception {
        conn.rollback();  // Откат: все изменения отменяются
        conn.close();
    }
}
```

**Плюсы:** полная изоляция тестов, нулевой мусор после прогона.
**Минусы:** не работает, если тестируемое приложение использует **свою** транзакцию
(тест и приложение работают в разных соединениях).

### 4. Именованные маркеры

```java
// Использование уникальных идентификаторов для тестовых данных
private static final String TEST_PREFIX = "autotest_" + UUID.randomUUID().toString().substring(0, 8);

@BeforeEach
void setup() {
    db.executeUpdate(
        "INSERT INTO users (email, first_name) VALUES (?, ?)",
        TEST_PREFIX + "@test.com", TEST_PREFIX
    );
}

@AfterEach
void cleanup() {
    db.executeUpdate("DELETE FROM users WHERE email = ?", TEST_PREFIX + "@test.com");
}
```

**Плюсы:** можно запускать тесты параллельно — маркеры уникальны.

---

## Тестирование хранимых процедур

```java
@Test
void testCalculateOrderTotal_storedProcedure() throws Exception {
    try (Connection conn = DbConnection.getConnection()) {
        // Вызов хранимой процедуры / функции PostgreSQL
        try (var cstmt = conn.prepareCall("SELECT calculate_order_total(?)")) {
            cstmt.setLong(1, orderId);

            try (var rs = cstmt.executeQuery()) {
                rs.next();
                BigDecimal total = rs.getBigDecimal(1);
                assertThat(total).isEqualByComparingTo(new BigDecimal("1500.00"));
            }
        }
    }
}

@Test
void testCleanupExpiredOrders_procedure() throws Exception {
    // Подготовка: создать просроченные заказы
    db.executeUpdate(
        "INSERT INTO orders (order_id, status, created_at) VALUES (?, ?, ?)",
        99901, "PENDING", Timestamp.valueOf("2024-01-01 00:00:00")
    );

    // Вызов процедуры очистки
    try (Connection conn = DbConnection.getConnection();
         var stmt = conn.createStatement()) {
        stmt.execute("CALL cleanup_expired_orders(30)");  // старше 30 дней
    }

    // Проверка: заказ удалён или переведён в нужный статус
    var rows = db.executeQuery(
        "SELECT status FROM orders WHERE order_id = ?", 99901
    );
    assertThat(rows.get(0).get("status")).isEqualTo("EXPIRED");
}
```

---

## Связь с тестированием

| Задача QA                                   | Подход                                      |
|---------------------------------------------|---------------------------------------------|
| Проверить запись данных после API-вызова     | Setup API -> Verify DB                      |
| Подготовить сложное начальное состояние      | Прямой INSERT в БД                          |
| Проверить soft delete                        | SELECT + проверка флага `is_deleted`        |
| Тестировать побочные эффекты                 | SELECT из таблиц audit_log, notifications   |
| Изолировать тесты друг от друга              | Transaction rollback / TRUNCATE / маркеры   |
| Проверить бизнес-логику в хранимых процедурах| Вызов через JDBC + проверка результата      |

---

## Типичные ошибки

1. **Не закрывать Connection/Statement/ResultSet** — утечка соединений. Пул исчерпывается,
   тесты начинают зависать. Всегда используйте `try-with-resources`.

2. **Конкатенация параметров в SQL** — SQL injection, даже в тестах это плохая привычка.
   Используйте `PreparedStatement`.

3. **Удаление данных в неправильном порядке** — ошибка foreign key constraint. Сначала
   удаляйте дочерние записи, потом родительские.

4. **Забыть про `wasNull()`** — `getInt()` возвращает `0` для `NULL`. Если нужно отличить
   `NULL` от `0`, вызывайте `rs.wasNull()` после чтения.

5. **Проверка через БД слишком рано** — при асинхронной обработке данные могут ещё не
   записаться. Используйте polling с timeout:

   ```java
   // Ожидание появления записи в БД (с таймаутом)
   Awaitility.await()
       .atMost(Duration.ofSeconds(10))
       .pollInterval(Duration.ofMillis(500))
       .until(() -> {
           var rows = db.executeQuery(
               "SELECT status FROM orders WHERE order_id = ?", orderId
           );
           return !rows.isEmpty() && "PROCESSED".equals(rows.get(0).get("status"));
       });
   ```

6. **Жёсткая привязка к конкретным ID** — `WHERE user_id = 1` ломается при параллельном
   запуске или на другом окружении. Генерируйте уникальные данные.

7. **Тесты зависят от порядка выполнения** — один тест создаёт данные, другой на них
   рассчитывает. Каждый тест должен быть самодостаточным.

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Зачем QA проверять данные непосредственно в базе данных?
2. Что такое JDBC? Какие основные классы используются?
3. Чем `Statement` отличается от `PreparedStatement`?
4. Почему нельзя подставлять параметры через конкатенацию строк?
5. Что такое `ResultSet` и как с ним работать?

### 🟡 Средний уровень
6. Опишите паттерн «Setup через API, Verify в DB».
7. Какие стратегии очистки тестовых данных вы знаете? Сравните их.
8. Как тестировать soft delete через БД?
9. Как обрабатывать `NULL`-значения при чтении `ResultSet`?
10. Почему важен порядок удаления при наличии foreign keys?

### 🔴 Продвинутый уровень
11. Как обеспечить изоляцию тестов при параллельном выполнении и общей БД?
12. Как тестировать асинхронные процессы, результат которых появляется в БД с задержкой?
13. В чём проблема transaction rollback стратегии при интеграционном тестировании?
14. Как организовать переиспользуемый DbUtils для большого QA-проекта?
15. Как протестировать хранимую процедуру, которая модифицирует несколько таблиц?

---

## Практические задания

### Задание 1. Утилитный класс
Реализуйте метод `DbUtils.executeScalarInt(String sql, Object... params)`, который возвращает
единственное числовое значение из запроса (например, `SELECT COUNT(*) FROM ...`).

<details>
<summary>Решение</summary>

```java
public int executeScalarInt(String sql, Object... params) {
    List<Map<String, Object>> rows = executeQuery(sql, params);
    if (rows.isEmpty()) {
        throw new RuntimeException("Запрос не вернул результатов: " + sql);
    }
    Object value = rows.get(0).values().iterator().next();
    if (value instanceof Number) {
        return ((Number) value).intValue();
    }
    throw new RuntimeException("Результат не является числом: " + value);
}
```

</details>

### Задание 2. End-to-end тест с проверкой через БД
Напишите тест: вызовите `POST /api/users` для создания пользователя, затем проверьте в БД,
что запись создана, `created_at` не `NULL`, а `status` = `ACTIVE`.

<details>
<summary>Решение</summary>

```java
@Test
void createUser_shouldPersistWithCorrectDefaults() {
    var response = given()
        .contentType("application/json")
        .body("""
            {"email": "dbtest@test.com", "firstName": "DB", "lastName": "Test"}
            """)
        .when()
        .post("/api/users")
        .then()
        .statusCode(201)
        .extract().response();

    long userId = response.jsonPath().getLong("id");

    var rows = db.executeQuery(
        "SELECT email, status, created_at FROM users WHERE user_id = ?", userId
    );

    assertThat(rows).hasSize(1);
    Map<String, Object> user = rows.get(0);
    assertThat(user.get("email")).isEqualTo("dbtest@test.com");
    assertThat(user.get("status")).isEqualTo("ACTIVE");
    assertThat(user.get("created_at")).isNotNull();
}
```

</details>

### Задание 3. Стратегия очистки
Реализуйте `@AfterEach`-метод, который удаляет все данные, созданные тестом, используя
именованный маркер.

<details>
<summary>Решение</summary>

```java
private static final String TEST_MARKER = "autotest_" + System.currentTimeMillis();

@BeforeEach
void setup() {
    db.executeUpdate(
        "INSERT INTO users (email, first_name) VALUES (?, ?)",
        TEST_MARKER + "@test.com", TEST_MARKER
    );
}

@AfterEach
void cleanup() {
    // Сначала дочерние записи
    db.executeUpdate(
        "DELETE FROM orders WHERE customer_id IN " +
        "(SELECT user_id FROM users WHERE email LIKE ?)",
        TEST_MARKER + "%"
    );
    // Затем родительские
    db.executeUpdate("DELETE FROM users WHERE email LIKE ?", TEST_MARKER + "%");
}
```

</details>

---

## Дополнительные ресурсы

- [JDBC Tutorial (Oracle)](https://docs.oracle.com/javase/tutorial/jdbc/) — официальное руководство по JDBC
- [AssertJ](https://assertj.github.io/doc/) — библиотека fluent assertions для Java
- [Awaitility](https://github.com/awaitility/awaitility) — библиотека для ожидания асинхронных условий
- [HikariCP](https://github.com/brettwooldridge/HikariCP) — высокопроизводительный connection pool
- [DbUnit](http://dbunit.sourceforge.net/) — фреймворк для управления тестовыми данными в БД
- [Database Rider](https://github.com/database-rider/database-rider) — современная альтернатива DbUnit
