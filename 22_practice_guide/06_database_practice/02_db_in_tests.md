# Проверка данных в БД из автотестов

## Введение

Автоматизированные тесты не должны верить только ответу API или UI. Полноценная проверка
включает верификацию данных непосредственно в базе данных. Это гарантирует, что данные
действительно сохранены, а не просто закэшированы в приложении. В этом руководстве мы
рассмотрим два подхода: прямое подключение через JDBC и использование TestContainers
для изолированной БД в Docker.

> **Цель:** Научиться подключаться к базе данных из автотестов, выполнять SQL-запросы,
> маппить результаты в объекты и использовать TestContainers для изолированного окружения.

---

## Часть 1: JDBC — прямое подключение к БД

### Шаг 1: Maven-зависимости

```xml
<dependencies>
    <!-- PostgreSQL JDBC Driver -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <version>42.7.3</version>
        <scope>test</scope>
    </dependency>

    <!-- HikariCP — пул соединений (быстрый и надёжный) -->
    <dependency>
        <groupId>com.zaxxer</groupId>
        <artifactId>HikariCP</artifactId>
        <version>5.1.0</version>
        <scope>test</scope>
    </dependency>

    <!-- JUnit 5 -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.10.2</version>
        <scope>test</scope>
    </dependency>

    <!-- AssertJ — удобные assertions -->
    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <version>3.25.3</version>
        <scope>test</scope>
    </dependency>

    <!-- REST Assured — для API-вызовов -->
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>rest-assured</artifactId>
        <version>5.4.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Шаг 2: Класс подключения к БД

```java
public class DatabaseConnection {

    private static HikariDataSource dataSource;

    // Инициализация пула соединений — вызывается один раз
    public static void init() {
        if (dataSource != null) {
            return;
        }

        HikariConfig config = new HikariConfig();
        // Параметры подключения из конфигурации
        config.setJdbcUrl(ConfigReader.getDbUrl());
        config.setUsername(ConfigReader.getDbUser());
        config.setPassword(ConfigReader.getDbPassword());

        // Настройки пула
        config.setMaximumPoolSize(5);        // Максимум 5 соединений
        config.setMinimumIdle(1);             // Минимум 1 свободное соединение
        config.setConnectionTimeout(10000);   // Таймаут подключения: 10 секунд
        config.setIdleTimeout(300000);        // Время простоя: 5 минут
        config.setMaxLifetime(600000);        // Максимальное время жизни: 10 минут

        // Тестовый запрос при получении соединения из пула
        config.setConnectionTestQuery("SELECT 1");

        dataSource = new HikariDataSource(config);
    }

    // Получение соединения из пула
    public static Connection getConnection() throws SQLException {
        if (dataSource == null) {
            init();
        }
        return dataSource.getConnection();
    }

    // Закрытие пула соединений
    public static void close() {
        if (dataSource != null && !dataSource.isClosed()) {
            dataSource.close();
        }
    }
}
```

### Шаг 3: Выполнение запросов — DatabaseClient

```java
public class DatabaseClient {

    // Выполнение SELECT-запроса и маппинг результата
    public static <T> List<T> executeQuery(String sql, RowMapper<T> mapper,
                                            Object... params) {
        List<T> results = new ArrayList<>();

        try (Connection conn = DatabaseConnection.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {

            // Установка параметров запроса
            for (int i = 0; i < params.length; i++) {
                stmt.setObject(i + 1, params[i]);
            }

            try (ResultSet rs = stmt.executeQuery()) {
                while (rs.next()) {
                    results.add(mapper.map(rs));
                }
            }

        } catch (SQLException e) {
            throw new RuntimeException(
                "Ошибка выполнения SQL-запроса: " + sql, e);
        }

        return results;
    }

    // Выполнение SELECT-запроса, возвращающего одну запись
    public static <T> Optional<T> executeQueryForOne(String sql,
                                                      RowMapper<T> mapper,
                                                      Object... params) {
        List<T> results = executeQuery(sql, mapper, params);
        return results.isEmpty() ? Optional.empty() : Optional.of(results.get(0));
    }

    // Выполнение INSERT/UPDATE/DELETE
    public static int executeUpdate(String sql, Object... params) {
        try (Connection conn = DatabaseConnection.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {

            for (int i = 0; i < params.length; i++) {
                stmt.setObject(i + 1, params[i]);
            }

            return stmt.executeUpdate();

        } catch (SQLException e) {
            throw new RuntimeException(
                "Ошибка выполнения SQL-обновления: " + sql, e);
        }
    }

    // Выполнение SELECT COUNT(*)
    public static int count(String tableName, String whereClause,
                            Object... params) {
        String sql = "SELECT COUNT(*) FROM " + tableName;
        if (whereClause != null && !whereClause.isEmpty()) {
            sql += " WHERE " + whereClause;
        }

        try (Connection conn = DatabaseConnection.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {

            for (int i = 0; i < params.length; i++) {
                stmt.setObject(i + 1, params[i]);
            }

            try (ResultSet rs = stmt.executeQuery()) {
                rs.next();
                return rs.getInt(1);
            }

        } catch (SQLException e) {
            throw new RuntimeException(
                "Ошибка подсчёта записей в " + tableName, e);
        }
    }

    // Функциональный интерфейс для маппинга строки ResultSet в объект
    @FunctionalInterface
    public interface RowMapper<T> {
        T map(ResultSet rs) throws SQLException;
    }
}
```

---

## Часть 2: Repository-классы — доменный доступ к данным

### UserRepository

```java
public class UserRepository {

    // Поиск пользователя по email
    public static Optional<UserEntity> findByEmail(String email) {
        String sql = """
            SELECT id, username, email, bio, image, created_at, updated_at
            FROM users
            WHERE email = ?
            """;

        return DatabaseClient.executeQueryForOne(sql, rs -> new UserEntity(
            rs.getInt("id"),
            rs.getString("username"),
            rs.getString("email"),
            rs.getString("bio"),
            rs.getString("image"),
            rs.getTimestamp("created_at").toLocalDateTime(),
            rs.getTimestamp("updated_at").toLocalDateTime()
        ), email);
    }

    // Поиск пользователя по username
    public static Optional<UserEntity> findByUsername(String username) {
        String sql = """
            SELECT id, username, email, bio, image, created_at, updated_at
            FROM users
            WHERE username = ?
            """;

        return DatabaseClient.executeQueryForOne(sql, rs -> new UserEntity(
            rs.getInt("id"),
            rs.getString("username"),
            rs.getString("email"),
            rs.getString("bio"),
            rs.getString("image"),
            rs.getTimestamp("created_at").toLocalDateTime(),
            rs.getTimestamp("updated_at").toLocalDateTime()
        ), username);
    }

    // Подсчёт пользователей
    public static int countAll() {
        return DatabaseClient.count("users", null);
    }

    // Удаление тестовых пользователей (очистка после тестов)
    public static int deleteByEmailPattern(String emailPattern) {
        return DatabaseClient.executeUpdate(
            "DELETE FROM users WHERE email LIKE ?",
            emailPattern + "%"
        );
    }
}
```

### ArticleRepository

```java
public class ArticleRepository {

    // Поиск статьи по slug
    public static Optional<ArticleEntity> findBySlug(String slug) {
        String sql = """
            SELECT a.id, a.slug, a.title, a.description, a.body,
                   a.author_id, u.username AS author_username,
                   a.created_at, a.updated_at
            FROM articles a
            INNER JOIN users u ON a.author_id = u.id
            WHERE a.slug = ?
            """;

        return DatabaseClient.executeQueryForOne(sql, rs -> new ArticleEntity(
            rs.getInt("id"),
            rs.getString("slug"),
            rs.getString("title"),
            rs.getString("description"),
            rs.getString("body"),
            rs.getInt("author_id"),
            rs.getString("author_username"),
            rs.getTimestamp("created_at").toLocalDateTime(),
            rs.getTimestamp("updated_at").toLocalDateTime()
        ), slug);
    }

    // Получение тегов статьи
    public static List<String> getTagsBySlug(String slug) {
        String sql = """
            SELECT t.name
            FROM tags t
            INNER JOIN article_tags at2 ON t.id = at2.tag_id
            INNER JOIN articles a ON at2.article_id = a.id
            WHERE a.slug = ?
            ORDER BY t.name
            """;

        return DatabaseClient.executeQuery(sql,
            rs -> rs.getString("name"), slug);
    }

    // Подсчёт комментариев к статье
    public static int countComments(String slug) {
        String sql = """
            SELECT COUNT(c.id)
            FROM comments c
            INNER JOIN articles a ON c.article_id = a.id
            WHERE a.slug = ?
            """;

        return DatabaseClient.executeQuery(sql,
            rs -> rs.getInt(1), slug).stream()
            .findFirst().orElse(0);
    }

    // Подсчёт лайков статьи
    public static int countFavorites(String slug) {
        String sql = """
            SELECT COUNT(f.user_id)
            FROM favorites f
            INNER JOIN articles a ON f.article_id = a.id
            WHERE a.slug = ?
            """;

        return DatabaseClient.executeQuery(sql,
            rs -> rs.getInt(1), slug).stream()
            .findFirst().orElse(0);
    }
}
```

### Entity-классы

```java
// Представление записи пользователя из БД
public record UserEntity(
    int id,
    String username,
    String email,
    String bio,
    String image,
    LocalDateTime createdAt,
    LocalDateTime updatedAt
) {}

// Представление записи статьи из БД
public record ArticleEntity(
    int id,
    String slug,
    String title,
    String description,
    String body,
    int authorId,
    String authorUsername,
    LocalDateTime createdAt,
    LocalDateTime updatedAt
) {}
```

---

## Часть 3: Полный пример — API-тест с проверкой в БД

### Тест: Создание статьи с верификацией в БД

```java
@Epic("Статьи")
@Feature("Создание статьи")
public class CreateArticleDbVerificationTest {

    private static String authToken;
    private static String testEmail;
    private static String testUsername;

    @BeforeAll
    static void setUp() {
        // Инициализируем подключение к БД
        DatabaseConnection.init();

        // Создаём тестового пользователя через API
        testEmail = DataGenerator.uniqueEmail();
        testUsername = DataGenerator.uniqueUsername();
        String password = "TestPass123!";

        authToken = given()
            .contentType(ContentType.JSON)
            .body(String.format(
                "{\"user\":{\"email\":\"%s\",\"password\":\"%s\",\"username\":\"%s\"}}",
                testEmail, password, testUsername))
            .post(ConfigReader.getBaseUrl() + "/api/users")
            .then()
            .statusCode(200)
            .extract()
            .path("user.token");
    }

    @AfterAll
    static void tearDown() {
        // Очистка тестовых данных
        UserRepository.deleteByEmailPattern("test_");
        DatabaseConnection.close();
    }

    @Test
    @Story("Создание статьи с тегами — проверка в БД")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("API создаёт статью → данные корректно сохраняются в БД")
    void testCreateArticle_verifyInDatabase() {
        // ========== ПОДГОТОВКА ==========
        String title = "Test Article " + System.currentTimeMillis();
        String description = "Article description for DB verification";
        String body = "Full article body content for testing";
        List<String> tags = List.of("java", "testing", "automation");

        String requestBody = String.format("""
            {
                "article": {
                    "title": "%s",
                    "description": "%s",
                    "body": "%s",
                    "tagList": ["java", "testing", "automation"]
                }
            }
            """, title, description, body);

        // ========== ДЕЙСТВИЕ: создаём статью через API ==========
        String slug = given()
            .contentType(ContentType.JSON)
            .header("Authorization", "Token " + authToken)
            .body(requestBody)
        .when()
            .post(ConfigReader.getBaseUrl() + "/api/articles")
        .then()
            .statusCode(200)
            .body("article.title", equalTo(title))
            .extract()
            .path("article.slug");

        // ========== ПРОВЕРКА 1: Статья существует в БД ==========
        Optional<ArticleEntity> articleOpt = ArticleRepository.findBySlug(slug);

        assertThat(articleOpt)
            .as("Статья должна существовать в базе данных")
            .isPresent();

        ArticleEntity article = articleOpt.get();

        // ========== ПРОВЕРКА 2: Все поля сохранены корректно ==========
        assertAll("Проверка полей статьи в БД",
            () -> assertThat(article.title())
                .as("Заголовок статьи")
                .isEqualTo(title),

            () -> assertThat(article.description())
                .as("Описание статьи")
                .isEqualTo(description),

            () -> assertThat(article.body())
                .as("Тело статьи")
                .isEqualTo(body),

            () -> assertThat(article.authorUsername())
                .as("Автор статьи")
                .isEqualTo(testUsername),

            () -> assertThat(article.createdAt())
                .as("Дата создания — в пределах последней минуты")
                .isAfter(LocalDateTime.now().minusMinutes(1))
        );

        // ========== ПРОВЕРКА 3: Теги привязаны к статье ==========
        List<String> savedTags = ArticleRepository.getTagsBySlug(slug);

        assertThat(savedTags)
            .as("Теги статьи в БД")
            .containsExactlyInAnyOrder("automation", "java", "testing");
    }

    @Test
    @Story("Удаление статьи — каскадное удаление в БД")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("Удаление статьи через API удаляет все связанные данные из БД")
    void testDeleteArticle_verifyCascadeInDatabase() {
        // Создаём статью
        String slug = createArticleViaApi("Delete Me " + System.currentTimeMillis());

        // Добавляем комментарий
        given()
            .contentType(ContentType.JSON)
            .header("Authorization", "Token " + authToken)
            .body("{\"comment\":{\"body\":\"Test comment\"}}")
            .post(ConfigReader.getBaseUrl() + "/api/articles/" + slug + "/comments")
            .then()
            .statusCode(200);

        // Добавляем лайк
        given()
            .header("Authorization", "Token " + authToken)
            .post(ConfigReader.getBaseUrl() + "/api/articles/" + slug + "/favorite")
            .then()
            .statusCode(200);

        // Проверяем, что комментарий и лайк есть в БД
        assertThat(ArticleRepository.countComments(slug)).isGreaterThan(0);
        assertThat(ArticleRepository.countFavorites(slug)).isGreaterThan(0);

        // ========== ДЕЙСТВИЕ: удаляем статью ==========
        given()
            .header("Authorization", "Token " + authToken)
            .delete(ConfigReader.getBaseUrl() + "/api/articles/" + slug)
            .then()
            .statusCode(204);

        // ========== ПРОВЕРКА: каскадное удаление ==========
        assertAll("Каскадное удаление в БД",
            () -> assertThat(ArticleRepository.findBySlug(slug))
                .as("Статья удалена из БД")
                .isEmpty(),

            () -> assertThat(ArticleRepository.countComments(slug))
                .as("Комментарии удалены из БД")
                .isZero(),

            () -> assertThat(ArticleRepository.countFavorites(slug))
                .as("Лайки удалены из БД")
                .isZero()
        );
    }

    // Вспомогательный метод создания статьи
    private String createArticleViaApi(String title) {
        return given()
            .contentType(ContentType.JSON)
            .header("Authorization", "Token " + authToken)
            .body(String.format(
                "{\"article\":{\"title\":\"%s\",\"description\":\"desc\"," +
                "\"body\":\"body\",\"tagList\":[\"test\"]}}",
                title))
            .post(ConfigReader.getBaseUrl() + "/api/articles")
            .then()
            .statusCode(200)
            .extract()
            .path("article.slug");
    }
}
```

---

## Часть 4: TestContainers — изолированная БД в Docker

### Зависимости

```xml
<dependencies>
    <!-- TestContainers — базовый модуль -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>testcontainers</artifactId>
        <version>1.19.7</version>
        <scope>test</scope>
    </dependency>

    <!-- TestContainers — модуль PostgreSQL -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>postgresql</artifactId>
        <version>1.19.7</version>
        <scope>test</scope>
    </dependency>

    <!-- TestContainers — интеграция с JUnit 5 -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>1.19.7</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Тест с TestContainers

```java
// @Testcontainers включает поддержку жизненного цикла контейнеров
@Testcontainers
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
public class DatabaseIntegrationTest {

    // @Container — JUnit 5 автоматически запустит и остановит контейнер
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test")
        // SQL-скрипт для создания схемы — выполняется при запуске контейнера
        .withInitScript("db/init-schema.sql");

    private static Connection connection;

    @BeforeAll
    static void setUp() throws SQLException {
        // Подключаемся к контейнеру — порт назначается автоматически
        connection = DriverManager.getConnection(
            postgres.getJdbcUrl(),
            postgres.getUsername(),
            postgres.getPassword()
        );

        System.out.println("Подключение к TestContainers PostgreSQL:");
        System.out.println("  URL:  " + postgres.getJdbcUrl());
        System.out.println("  Port: " + postgres.getMappedPort(5432));
    }

    @AfterAll
    static void tearDown() throws SQLException {
        if (connection != null) {
            connection.close();
        }
        // Контейнер остановится автоматически благодаря @Testcontainers
    }

    @Test
    @Order(1)
    @DisplayName("Создание пользователя в изолированной БД")
    void testCreateUser() throws SQLException {
        String sql = """
            INSERT INTO users (username, email, password)
            VALUES (?, ?, ?)
            RETURNING id
            """;

        try (PreparedStatement stmt = connection.prepareStatement(sql)) {
            stmt.setString(1, "testuser");
            stmt.setString(2, "test@example.com");
            stmt.setString(3, "hashed_password");

            ResultSet rs = stmt.executeQuery();
            assertTrue(rs.next(), "Пользователь должен быть создан");

            int userId = rs.getInt("id");
            assertTrue(userId > 0, "ID пользователя должен быть положительным");
        }
    }

    @Test
    @Order(2)
    @DisplayName("Проверка данных пользователя")
    void testVerifyUser() throws SQLException {
        String sql = "SELECT username, email FROM users WHERE email = ?";

        try (PreparedStatement stmt = connection.prepareStatement(sql)) {
            stmt.setString(1, "test@example.com");

            ResultSet rs = stmt.executeQuery();
            assertTrue(rs.next(), "Пользователь должен существовать");

            assertEquals("testuser", rs.getString("username"));
            assertEquals("test@example.com", rs.getString("email"));
        }
    }

    @Test
    @Order(3)
    @DisplayName("Проверка уникальности email")
    void testUniqueEmailConstraint() throws SQLException {
        String sql = """
            INSERT INTO users (username, email, password)
            VALUES (?, ?, ?)
            """;

        try (PreparedStatement stmt = connection.prepareStatement(sql)) {
            stmt.setString(1, "another_user");
            stmt.setString(2, "test@example.com"); // Дубликат email
            stmt.setString(3, "password");

            // Должно выбросить исключение — нарушение уникальности
            assertThrows(SQLException.class, stmt::executeUpdate,
                "Дубликат email должен вызвать ошибку");
        }
    }

    @Test
    @Order(4)
    @DisplayName("Каскадное удаление: удаление пользователя удаляет его статьи")
    void testCascadeDelete() throws SQLException {
        // Создаём статью для пользователя
        try (PreparedStatement stmt = connection.prepareStatement(
                "INSERT INTO articles (slug, title, body, author_id) " +
                "VALUES (?, ?, ?, (SELECT id FROM users WHERE email = ?))")) {
            stmt.setString(1, "test-article");
            stmt.setString(2, "Test Article");
            stmt.setString(3, "Body");
            stmt.setString(4, "test@example.com");
            stmt.executeUpdate();
        }

        // Проверяем, что статья создана
        int articleCount = countRecords("articles", "slug = 'test-article'");
        assertEquals(1, articleCount, "Статья должна существовать");

        // Удаляем пользователя
        try (PreparedStatement stmt = connection.prepareStatement(
                "DELETE FROM users WHERE email = ?")) {
            stmt.setString(1, "test@example.com");
            stmt.executeUpdate();
        }

        // Проверяем каскадное удаление
        int articleCountAfter = countRecords("articles", "slug = 'test-article'");
        assertEquals(0, articleCountAfter,
            "Статья должна быть удалена каскадно при удалении автора");
    }

    private int countRecords(String table, String where) throws SQLException {
        String sql = "SELECT COUNT(*) FROM " + table + " WHERE " + where;
        try (Statement stmt = connection.createStatement();
             ResultSet rs = stmt.executeQuery(sql)) {
            rs.next();
            return rs.getInt(1);
        }
    }
}
```

### SQL-скрипт инициализации

Файл: `src/test/resources/db/init-schema.sql`

```sql
-- Схема базы данных для TestContainers
CREATE TABLE users (
    id          SERIAL PRIMARY KEY,
    username    VARCHAR(100) NOT NULL UNIQUE,
    email       VARCHAR(255) NOT NULL UNIQUE,
    password    VARCHAR(255) NOT NULL,
    bio         TEXT,
    image       VARCHAR(500),
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE articles (
    id          SERIAL PRIMARY KEY,
    slug        VARCHAR(255) NOT NULL UNIQUE,
    title       VARCHAR(255) NOT NULL,
    description TEXT,
    body        TEXT NOT NULL,
    author_id   INTEGER REFERENCES users(id) ON DELETE CASCADE,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE comments (
    id         SERIAL PRIMARY KEY,
    body       TEXT NOT NULL,
    article_id INTEGER REFERENCES articles(id) ON DELETE CASCADE,
    author_id  INTEGER REFERENCES users(id) ON DELETE CASCADE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Тестовые данные
INSERT INTO users (username, email, password) VALUES
    ('admin', 'admin@example.com', 'hashed_admin_password'),
    ('author1', 'author1@example.com', 'hashed_password_1'),
    ('reader1', 'reader1@example.com', 'hashed_password_2');
```

---

## Часть 5: Утилита ожидания данных в БД

Иногда данные появляются в БД с задержкой (асинхронная обработка, очереди).
Нужен механизм ожидания.

```java
public class DatabaseWaiter {

    // Ожидание появления записи в БД с указанным таймаутом
    public static <T> T waitForRecord(
            String description,
            Supplier<Optional<T>> query,
            Duration timeout,
            Duration pollInterval
    ) {
        Instant start = Instant.now();

        while (Duration.between(start, Instant.now()).compareTo(timeout) < 0) {
            Optional<T> result = query.get();
            if (result.isPresent()) {
                return result.get();
            }

            try {
                Thread.sleep(pollInterval.toMillis());
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new RuntimeException("Ожидание прервано", e);
            }
        }

        throw new AssertionError(
            "Запись не появилась в БД за " + timeout.getSeconds() + " секунд: "
            + description
        );
    }

    // Ожидание определённого количества записей
    public static void waitForCount(
            String table, String where, int expectedCount,
            Duration timeout, Object... params
    ) {
        Instant start = Instant.now();

        while (Duration.between(start, Instant.now()).compareTo(timeout) < 0) {
            int actual = DatabaseClient.count(table, where, params);
            if (actual == expectedCount) {
                return;
            }

            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new RuntimeException(e);
            }
        }

        int actual = DatabaseClient.count(table, where, params);
        throw new AssertionError(String.format(
            "Ожидалось %d записей в %s (WHERE %s), но найдено %d",
            expectedCount, table, where, actual
        ));
    }
}
```

### Использование в тесте

```java
@Test
void testAsyncArticleCreation() {
    // Создаём статью через API (обработка может быть асинхронной)
    String slug = createArticleViaApi("Async Article");

    // Ожидаем появление записи в БД (до 10 секунд, опрос каждые 500 мс)
    ArticleEntity article = DatabaseWaiter.waitForRecord(
        "Статья с slug=" + slug,
        () -> ArticleRepository.findBySlug(slug),
        Duration.ofSeconds(10),
        Duration.ofMillis(500)
    );

    // Проверяем данные
    assertThat(article.title()).isEqualTo("Async Article");
}
```

---

## Часть 6: Очистка тестовых данных

### Стратегии очистки

```java
public class TestDataCleaner {

    // Стратегия 1: Очистка по паттерну (для тестовых email-адресов)
    public static void cleanupByEmailPattern() {
        int deleted = DatabaseClient.executeUpdate(
            "DELETE FROM users WHERE email LIKE 'test_%@test.com'");
        System.out.println("Удалено тестовых пользователей: " + deleted);
    }

    // Стратегия 2: Очистка по времени создания (для CI-прогонов)
    public static void cleanupOlderThan(Duration age) {
        Timestamp cutoff = Timestamp.valueOf(
            LocalDateTime.now().minus(age));

        int articles = DatabaseClient.executeUpdate(
            "DELETE FROM articles WHERE created_at < ?", cutoff);
        int users = DatabaseClient.executeUpdate(
            "DELETE FROM users WHERE created_at < ? AND email LIKE 'test_%'",
            cutoff);

        System.out.printf("Очистка: %d статей, %d пользователей%n",
                          articles, users);
    }

    // Стратегия 3: Полная очистка таблиц (для TestContainers)
    public static void truncateAll() {
        DatabaseClient.executeUpdate("TRUNCATE TABLE comments CASCADE");
        DatabaseClient.executeUpdate("TRUNCATE TABLE favorites CASCADE");
        DatabaseClient.executeUpdate("TRUNCATE TABLE article_tags CASCADE");
        DatabaseClient.executeUpdate("TRUNCATE TABLE articles CASCADE");
        DatabaseClient.executeUpdate("TRUNCATE TABLE follows CASCADE");
        DatabaseClient.executeUpdate("TRUNCATE TABLE users CASCADE");
    }
}
```

---

## Практическое задание

### Задание 1: JDBC — подключение и базовые запросы

1. Добавьте PostgreSQL JDBC Driver в pom.xml
2. Реализуйте `DatabaseConnection` с пулом соединений HikariCP
3. Реализуйте `DatabaseClient` с методами `executeQuery` и `executeUpdate`
4. Напишите тест: создайте пользователя через API и проверьте его в БД через SQL

**Ожидаемый результат:** Тест создаёт пользователя через REST API, затем через JDBC
выполняет SELECT и проверяет, что запись создана с корректными данными.

### Задание 2: Repository-слой

1. Создайте `UserRepository` с методами `findByEmail`, `findByUsername`, `countAll`
2. Создайте `ArticleRepository` с методами `findBySlug`, `getTagsBySlug`
3. Напишите тест создания статьи с верификацией всех полей в БД
4. Напишите тест удаления статьи с проверкой каскадного удаления

### Задание 3: TestContainers

1. Добавьте зависимости TestContainers в pom.xml
2. Создайте SQL-скрипт инициализации схемы
3. Напишите тест с `@Testcontainers`, который:
   - Запускает PostgreSQL в Docker
   - Выполняет миграцию
   - Вставляет тестовые данные
   - Проверяет constraints (уникальность, NOT NULL, CASCADE)
4. Убедитесь, что после завершения теста контейнер останавливается

**Критерии оценки:**
- Подключение к БД через пул соединений (не прямое DriverManager.getConnection)
- SQL-запросы параметризованы (PreparedStatement, без конкатенации строк)
- Тестовые данные очищаются после прогона
- TestContainers-тесты запускаются в изолированном контейнере
- Все assertions информативны (сообщения об ошибке описывают, что проверяется)

---

## Чек-лист самопроверки

- [ ] Подключение к PostgreSQL через JDBC работает
- [ ] Используется пул соединений (HikariCP)
- [ ] SQL-запросы параметризованы через PreparedStatement
- [ ] Реализованы Repository-классы для основных сущностей
- [ ] Тест создаёт данные через API и проверяет в БД
- [ ] Тест проверяет каскадное удаление
- [ ] TestContainers запускает PostgreSQL в Docker
- [ ] Init-скрипт создаёт схему и начальные данные
- [ ] Реализован механизм ожидания данных в БД (DatabaseWaiter)
- [ ] Тестовые данные очищаются после прогона
