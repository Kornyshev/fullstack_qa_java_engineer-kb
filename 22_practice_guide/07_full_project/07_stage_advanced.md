# Этап 6: Продвинутое -- полный фреймворк с DB, Docker и мониторингом

## Обзор

На предыдущих этапах мы построили фреймворк с API-тестами, UI-тестами, Allure-репортингом и CI/CD.
Это уже серьёзный уровень, но в реальных проектах QA-инженеру требуется больше:

- **Проверка данных в базе данных** -- убедиться, что API не просто вернул 200 OK, а реально
  записал данные в PostgreSQL с правильными значениями
- **Изолированное тестовое окружение** -- docker-compose, который поднимает всё приложение целиком
- **Тестирование асинхронных процессов** -- Kafka, RabbitMQ, очереди событий
- **Мониторинг результатов** -- отслеживание стабильности тестов, flaky-метрики
- **Управление тестовыми данными** -- генерация, очистка, изоляция между тестами
- **Анализ code coverage** -- JaCoCo для оценки покрытия кода тестами

**Deliverable:** полный продвинутый фреймворк тестирования, готовый для enterprise-проекта.

---

## Часть 1: Проверка данных в БД с TestContainers

### 1.1 Что такое TestContainers

TestContainers -- Java-библиотека, которая позволяет поднимать Docker-контейнеры прямо из
тестового кода. Для QA это означает: можно запустить реальный PostgreSQL (не mock, не in-memory H2),
выполнить тест и проверить, что данные корректно сохранились в БД.

### 1.2 Подключение зависимостей

```xml
<dependencies>
    <!-- TestContainers: ядро -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>testcontainers</artifactId>
        <version>1.19.7</version>
        <scope>test</scope>
    </dependency>

    <!-- TestContainers: модуль PostgreSQL -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>postgresql</artifactId>
        <version>1.19.7</version>
        <scope>test</scope>
    </dependency>

    <!-- TestContainers: интеграция с JUnit 5 -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>1.19.7</version>
        <scope>test</scope>
    </dependency>

    <!-- JDBC-драйвер PostgreSQL -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <version>42.7.3</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### 1.3 Базовый тест с PostgreSQL-контейнером

```java
import org.junit.jupiter.api.*;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.sql.*;

@Testcontainers
@Tag("db")
public class DatabaseVerificationTest {

    // Контейнер PostgreSQL: создаётся один раз на весь класс
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("conduit_test")
            .withUsername("test_user")
            .withPassword("test_password")
            // Начальная миграция: создание таблиц
            .withInitScript("db/init.sql");

    private Connection connection;

    @BeforeEach
    void setUp() throws SQLException {
        // Подключаемся к контейнеру
        connection = DriverManager.getConnection(
                postgres.getJdbcUrl(),
                postgres.getUsername(),
                postgres.getPassword()
        );
    }

    @AfterEach
    void tearDown() throws SQLException {
        if (connection != null && !connection.isClosed()) {
            connection.close();
        }
    }

    @Test
    @DisplayName("После создания пользователя запись появляется в таблице users")
    void shouldInsertUserIntoDatabase() throws SQLException {
        // Arrange: вставляем пользователя через SQL
        String insertSql = "INSERT INTO users (username, email, password_hash) VALUES (?, ?, ?)";
        try (PreparedStatement stmt = connection.prepareStatement(insertSql)) {
            stmt.setString(1, "testuser");
            stmt.setString(2, "test@example.com");
            stmt.setString(3, "hashed_password_123");
            stmt.executeUpdate();
        }

        // Act & Assert: проверяем, что запись существует
        String selectSql = "SELECT username, email FROM users WHERE username = ?";
        try (PreparedStatement stmt = connection.prepareStatement(selectSql)) {
            stmt.setString(1, "testuser");
            ResultSet rs = stmt.executeQuery();

            Assertions.assertTrue(rs.next(), "Запись пользователя должна существовать в БД");
            Assertions.assertEquals("testuser", rs.getString("username"));
            Assertions.assertEquals("test@example.com", rs.getString("email"));
        }
    }
}
```

### 1.4 Файл миграции `src/test/resources/db/init.sql`

```sql
-- Создание таблиц для тестовой базы данных
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(100) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    bio TEXT,
    image VARCHAR(512),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS articles (
    id SERIAL PRIMARY KEY,
    slug VARCHAR(255) NOT NULL UNIQUE,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    body TEXT,
    author_id INTEGER REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS comments (
    id SERIAL PRIMARY KEY,
    body TEXT NOT NULL,
    article_id INTEGER REFERENCES articles(id) ON DELETE CASCADE,
    author_id INTEGER REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS favorites (
    user_id INTEGER REFERENCES users(id),
    article_id INTEGER REFERENCES articles(id),
    PRIMARY KEY (user_id, article_id)
);
```

### 1.5 Утилитный класс для работы с БД

```java
public class DatabaseHelper {

    private final Connection connection;

    public DatabaseHelper(Connection connection) {
        this.connection = connection;
    }

    // Проверяем, существует ли запись в таблице по условию
    public boolean recordExists(String table, String column, Object value) throws SQLException {
        String sql = String.format("SELECT COUNT(*) FROM %s WHERE %s = ?", table, column);
        try (PreparedStatement stmt = connection.prepareStatement(sql)) {
            stmt.setObject(1, value);
            ResultSet rs = stmt.executeQuery();
            rs.next();
            return rs.getInt(1) > 0;
        }
    }

    // Получаем значение конкретного поля по условию
    public String getFieldValue(String table, String field, String whereColumn, Object whereValue)
            throws SQLException {
        String sql = String.format("SELECT %s FROM %s WHERE %s = ?", field, table, whereColumn);
        try (PreparedStatement stmt = connection.prepareStatement(sql)) {
            stmt.setObject(1, whereValue);
            ResultSet rs = stmt.executeQuery();
            if (rs.next()) {
                return rs.getString(1);
            }
            return null;
        }
    }

    // Подсчитываем количество записей в таблице
    public int countRecords(String table) throws SQLException {
        String sql = String.format("SELECT COUNT(*) FROM %s", table);
        try (Statement stmt = connection.createStatement()) {
            ResultSet rs = stmt.executeQuery(sql);
            rs.next();
            return rs.getInt(1);
        }
    }

    // Очищаем таблицу (для изоляции между тестами)
    public void truncateTable(String table) throws SQLException {
        String sql = String.format("TRUNCATE TABLE %s CASCADE", table);
        try (Statement stmt = connection.createStatement()) {
            stmt.execute(sql);
        }
    }
}
```

### 1.6 Интеграция: API-тест + проверка в БД

Самый мощный паттерн -- сценарий, где мы вызываем API и затем проверяем результат в БД:

```java
@Testcontainers
@Tag("integration")
public class ArticleIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("conduit_test")
            .withUsername("test_user")
            .withPassword("test_password")
            .withInitScript("db/init.sql");

    private DatabaseHelper dbHelper;

    @BeforeEach
    void setUp() throws SQLException {
        Connection conn = DriverManager.getConnection(
                postgres.getJdbcUrl(), postgres.getUsername(), postgres.getPassword());
        dbHelper = new DatabaseHelper(conn);
        // Очищаем таблицы перед каждым тестом для изоляции
        dbHelper.truncateTable("comments");
        dbHelper.truncateTable("favorites");
        dbHelper.truncateTable("articles");
        dbHelper.truncateTable("users");
    }

    @Test
    @DisplayName("Создание статьи через API сохраняет данные в БД")
    void shouldPersistArticleInDatabase() throws SQLException {
        // Arrange: создаём пользователя напрямую в БД
        try (PreparedStatement stmt = dbHelper.getConnection().prepareStatement(
                "INSERT INTO users (username, email, password_hash) VALUES ('author', 'a@test.com', 'hash')")) {
            stmt.executeUpdate();
        }

        // Act: вызываем API для создания статьи
        // (здесь подразумевается вызов через REST Assured к приложению,
        // которое подключено к этому же контейнеру PostgreSQL)

        // Assert: проверяем, что статья появилась в БД
        boolean exists = dbHelper.recordExists("articles", "slug", "test-article");
        Assertions.assertTrue(exists, "Статья должна быть сохранена в БД после вызова API");

        String title = dbHelper.getFieldValue("articles", "title", "slug", "test-article");
        Assertions.assertEquals("Test Article", title);
    }
}
```

---

## Часть 2: Docker-compose для полного тестового окружения

### 2.1 Архитектура тестового окружения

```
┌──────────────────────────────────────────────────────────────┐
│                    docker-compose.yml                         │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │  PostgreSQL   │  │  App Backend │  │  App Frontend│       │
│  │  :5432        │◄─┤  :8080       │  │  :3000       │       │
│  └──────────────┘  └──────┬───────┘  └──────────────┘       │
│                           │                                   │
│  ┌──────────────┐         │          ┌──────────────┐        │
│  │  Kafka        │◄───────┘          │  Selenoid     │       │
│  │  :9092        │                   │  :4444        │       │
│  └──────────────┘                    └──────────────┘        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
         ▲
         │  Тесты подключаются к контейнерам
         │
  ┌──────┴───────┐
  │  Test Runner  │  mvn clean test
  └──────────────┘
```

### 2.2 Файл docker-compose.yml

```yaml
version: '3.8'

services:
  # База данных PostgreSQL
  postgres:
    image: postgres:16-alpine
    container_name: conduit-db
    environment:
      POSTGRES_DB: conduit
      POSTGRES_USER: conduit_user
      POSTGRES_PASSWORD: conduit_pass
    ports:
      - "5432:5432"
    volumes:
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U conduit_user -d conduit"]
      interval: 5s
      timeout: 5s
      retries: 5

  # Backend приложения
  backend:
    image: conduit-backend:latest
    container_name: conduit-backend
    environment:
      DATABASE_URL: jdbc:postgresql://postgres:5432/conduit
      DATABASE_USER: conduit_user
      DATABASE_PASSWORD: conduit_pass
      JWT_SECRET: test-jwt-secret-key
    ports:
      - "8080:8080"
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/api/health"]
      interval: 10s
      timeout: 5s
      retries: 10

  # Frontend приложения
  frontend:
    image: conduit-frontend:latest
    container_name: conduit-frontend
    environment:
      API_URL: http://backend:8080/api
    ports:
      - "3000:3000"
    depends_on:
      - backend

  # Selenoid для запуска браузеров в контейнерах
  selenoid:
    image: aerokube/selenoid:1.11.3
    container_name: selenoid
    ports:
      - "4444:4444"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./selenoid/browsers.json:/etc/selenoid/browsers.json:ro
    command: ["-conf", "/etc/selenoid/browsers.json", "-video-output-dir", "/opt/selenoid/video"]

  # Selenoid UI для визуального наблюдения за тестами
  selenoid-ui:
    image: aerokube/selenoid-ui:1.10.11
    container_name: selenoid-ui
    ports:
      - "8900:8080"
    depends_on:
      - selenoid
    command: ["--selenoid-uri", "http://selenoid:4444"]
```

### 2.3 Файл конфигурации Selenoid `selenoid/browsers.json`

```json
{
  "chrome": {
    "default": "125.0",
    "versions": {
      "125.0": {
        "image": "selenoid/chrome:125.0",
        "port": "4444",
        "path": "/"
      }
    }
  },
  "firefox": {
    "default": "126.0",
    "versions": {
      "126.0": {
        "image": "selenoid/firefox:126.0",
        "port": "4444",
        "path": "/wd/hub"
      }
    }
  }
}
```

### 2.4 Скрипт запуска окружения и тестов

```bash
#!/bin/bash
# run-tests.sh -- Скрипт для поднятия окружения и запуска тестов

echo "=== Поднимаем тестовое окружение ==="
docker-compose up -d --wait

echo "=== Ждём готовности backend ==="
until curl -sf http://localhost:8080/api/health > /dev/null; do
    echo "Backend ещё не готов, ждём..."
    sleep 2
done
echo "Backend готов!"

echo "=== Запускаем тесты ==="
mvn clean test \
    -Dbase.url=http://localhost:8080/api \
    -Dapp.url=http://localhost:3000 \
    -Ddb.url=jdbc:postgresql://localhost:5432/conduit \
    -Ddb.user=conduit_user \
    -Ddb.password=conduit_pass \
    -Dselenide.remote=http://localhost:4444/wd/hub

TEST_EXIT_CODE=$?

echo "=== Генерируем Allure-отчёт ==="
allure generate target/allure-results --clean -o target/allure-report

echo "=== Останавливаем окружение ==="
docker-compose down

exit $TEST_EXIT_CODE
```

---

## Часть 3: Тестирование Kafka (асинхронные процессы)

### 3.1 Когда нужно тестировать Kafka

Во многих приложениях критические действия пользователя порождают **события** (events),
которые обрабатываются асинхронно через Kafka. Примеры:
- Создание заказа -> событие `OrderCreated` -> обработка оплаты
- Регистрация пользователя -> событие `UserRegistered` -> отправка email
- Публикация статьи -> событие `ArticlePublished` -> обновление индекса поиска

QA должен уметь проверить, что событие:
1. Отправлено в правильный topic
2. Содержит корректные данные
3. Обработано consumer'ом (проверяем побочный эффект)

### 3.2 Подключение TestContainers для Kafka

```xml
<!-- TestContainers: модуль Kafka -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>kafka</artifactId>
    <version>1.19.7</version>
    <scope>test</scope>
</dependency>

<!-- Kafka-клиент для отправки/чтения сообщений из теста -->
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>3.7.0</version>
    <scope>test</scope>
</dependency>
```

### 3.3 Тест с Kafka-контейнером

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.apache.kafka.common.serialization.StringSerializer;
import org.testcontainers.containers.KafkaContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;

import java.time.Duration;
import java.util.*;

@Testcontainers
@Tag("kafka")
public class KafkaEventTest {

    @Container
    static KafkaContainer kafka = new KafkaContainer(
            DockerImageName.parse("confluentinc/cp-kafka:7.6.0")
    );

    @Test
    @DisplayName("Событие ArticlePublished попадает в topic и содержит корректные данные")
    void shouldPublishArticleEvent() {
        String topic = "article-events";

        // Arrange: настраиваем producer
        Properties producerProps = new Properties();
        producerProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, kafka.getBootstrapServers());
        producerProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        producerProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

        // Act: отправляем событие (в реальном тесте это делает приложение)
        String eventJson = """
                {
                    "eventType": "ArticlePublished",
                    "articleId": 42,
                    "slug": "test-article",
                    "authorId": 1,
                    "timestamp": "2025-03-15T10:30:00Z"
                }
                """;

        try (KafkaProducer<String, String> producer = new KafkaProducer<>(producerProps)) {
            producer.send(new ProducerRecord<>(topic, "article-42", eventJson));
            producer.flush();
        }

        // Assert: читаем событие из topic и проверяем содержимое
        Properties consumerProps = new Properties();
        consumerProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, kafka.getBootstrapServers());
        consumerProps.put(ConsumerConfig.GROUP_ID_CONFIG, "test-group");
        consumerProps.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        consumerProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        consumerProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(consumerProps)) {
            consumer.subscribe(Collections.singletonList(topic));

            // Ожидаем получения сообщения (с таймаутом)
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(10));

            Assertions.assertFalse(records.isEmpty(), "Должно быть хотя бы одно сообщение в topic");

            ConsumerRecord<String, String> record = records.iterator().next();
            Assertions.assertEquals("article-42", record.key());
            Assertions.assertTrue(record.value().contains("ArticlePublished"));
            Assertions.assertTrue(record.value().contains("test-article"));
        }
    }
}
```

### 3.4 Паттерн: ожидание асинхронного результата

При тестировании асинхронных процессов нельзя проверять результат мгновенно -- нужно ждать.
Используем паттерн polling с библиотекой Awaitility:

```xml
<dependency>
    <groupId>org.awaitility</groupId>
    <artifactId>awaitility</artifactId>
    <version>4.2.1</version>
    <scope>test</scope>
</dependency>
```

```java
import org.awaitility.Awaitility;
import java.util.concurrent.TimeUnit;

@Test
@DisplayName("После публикации статьи поисковый индекс обновляется в течение 30 секунд")
void shouldUpdateSearchIndexAfterPublishing() {
    // Act: публикуем статью через API
    // ... вызов REST Assured ...

    // Assert: ожидаем, что статья появится в поисковом индексе
    Awaitility.await()
            .atMost(30, TimeUnit.SECONDS)       // Максимальное время ожидания
            .pollInterval(2, TimeUnit.SECONDS)   // Интервал между проверками
            .pollDelay(1, TimeUnit.SECONDS)      // Начальная задержка перед первой проверкой
            .untilAsserted(() -> {
                // Этот блок будет вызываться каждые 2 секунды,
                // пока assertion не пройдёт или не истечёт таймаут
                Response response = given()
                        .queryParam("q", "test-article")
                        .when()
                        .get("/api/search");

                response.then()
                        .statusCode(200)
                        .body("articles.size()", greaterThan(0))
                        .body("articles[0].slug", equalTo("test-article"));
            });
}
```

---

## Часть 4: Мониторинг результатов тестирования

### 4.1 Отслеживание flaky-тестов

Flaky-тесты -- тесты, которые то проходят, то падают без изменений в коде. Это серьёзная
проблема: команда перестаёт доверять результатам тестирования.

**Стратегия обнаружения flaky-тестов:**

```java
// Аннотация для повторного запуска упавших тестов
// Если тест прошёл со второй попытки -- он flaky
@RepeatedTest(3)  // Запускаем 3 раза
@DisplayName("Потенциально нестабильный тест")
void possiblyFlakyTest() {
    // ...
}
```

Более правильный подход -- использование JUnit 5 RetryExtension:

```java
// Кастомное расширение для повторного запуска упавших тестов
public class RetryExtension implements TestExecutionExceptionHandler {

    private static final int MAX_RETRIES = 2;

    @Override
    public void handleTestExecutionException(ExtensionContext context, Throwable throwable)
            throws Throwable {
        int retryCount = getRetryCount(context);
        if (retryCount < MAX_RETRIES) {
            setRetryCount(context, retryCount + 1);
            // Логируем повторный запуск для отслеживания flaky
            System.out.printf("[RETRY] Тест '%s' упал, попытка %d/%d%n",
                    context.getDisplayName(), retryCount + 1, MAX_RETRIES);
            // Повторно выполняем тест
            context.getRequiredTestMethod().invoke(context.getRequiredTestInstance());
        } else {
            throw throwable;
        }
    }

    private int getRetryCount(ExtensionContext context) {
        return context.getStore(ExtensionContext.Namespace.GLOBAL)
                .getOrDefault(context.getUniqueId() + "_retry", Integer.class, 0);
    }

    private void setRetryCount(ExtensionContext context, int count) {
        context.getStore(ExtensionContext.Namespace.GLOBAL)
                .put(context.getUniqueId() + "_retry", count);
    }
}

// Использование
@ExtendWith(RetryExtension.class)
public class UiTests {
    // Тесты будут автоматически перезапускаться при падении (до 2 раз)
}
```

### 4.2 Дашборд с метриками тестирования

Для мониторинга результатов создайте таблицу метрик, которую обновляете после каждого regression:

| Метрика | Целевое значение | Как считать |
|---------|-----------------|-------------|
| Pass rate | > 95% | Passed / Total * 100 |
| Flaky rate | < 3% | Tests_with_retries / Total * 100 |
| Среднее время suite | < 30 мин | Duration из Allure |
| Покрытие API endpoints | > 90% | Tested_endpoints / Total_endpoints |
| Покрытие UI scenarios | > 80% | Automated_scenarios / All_scenarios |
| Новые тесты за спринт | > 5 | Diff по git |

### 4.3 Allure категории для классификации падений

Создайте файл `src/test/resources/categories.json`:

```json
[
    {
        "name": "Баги продукта",
        "description": "Тесты, упавшие из-за реального бага в приложении",
        "matchedStatuses": ["failed"],
        "messageRegex": ".*Expected status code.*|.*Element not found.*"
    },
    {
        "name": "Проблемы окружения",
        "description": "Падения из-за недоступности сервиса, таймаутов сети",
        "matchedStatuses": ["broken"],
        "messageRegex": ".*Connection refused.*|.*timeout.*|.*UnreachableBrowserException.*"
    },
    {
        "name": "Проблемы тестовых данных",
        "description": "Падения из-за отсутствия или некорректности тестовых данных",
        "matchedStatuses": ["failed"],
        "messageRegex": ".*test data.*|.*not found in database.*"
    },
    {
        "name": "Flaky тесты",
        "description": "Тесты, которые нестабильно проходят",
        "matchedStatuses": ["failed", "broken"],
        "traceRegex": ".*StaleElementReferenceException.*|.*NoSuchSessionException.*"
    }
]
```

---

## Часть 5: Управление тестовыми данными

### 5.1 Стратегии управления данными

| Стратегия | Описание | Когда использовать |
|-----------|----------|--------------------|
| **Create before / delete after** | Каждый тест создаёт свои данные и удаляет после | Для изолированных тестов |
| **Shared fixtures** | Общий набор данных для группы тестов | Для read-only тестов |
| **Database snapshot** | Снимок БД, восстанавливаемый перед каждым suite | Для сложных взаимосвязей |
| **Factories / Builders** | Программная генерация данных по шаблону | Для масштабируемости |

### 5.2 Паттерн Test Data Builder

```java
// Builder для создания тестовых пользователей
public class UserBuilder {

    private String username = "user_" + UUID.randomUUID().toString().substring(0, 8);
    private String email = username + "@test.com";
    private String password = "TestPass123!";
    private String bio = "";
    private String image = "";

    public static UserBuilder aUser() {
        return new UserBuilder();
    }

    public UserBuilder withUsername(String username) {
        this.username = username;
        this.email = username + "@test.com";
        return this;
    }

    public UserBuilder withEmail(String email) {
        this.email = email;
        return this;
    }

    public UserBuilder withBio(String bio) {
        this.bio = bio;
        return this;
    }

    // Создаём пользователя через API и возвращаем токен
    public UserResponse createViaApi() {
        RegisterRequest request = new RegisterRequest(username, email, password);
        return given()
                .contentType(ContentType.JSON)
                .body(Map.of("user", request))
                .when()
                .post("/api/users")
                .then()
                .statusCode(201)
                .extract()
                .as(UserResponse.class);
    }

    // Создаём пользователя напрямую в БД (быстрее, для подготовки данных)
    public void createInDb(Connection connection) throws SQLException {
        String sql = "INSERT INTO users (username, email, password_hash, bio, image) VALUES (?, ?, ?, ?, ?)";
        try (PreparedStatement stmt = connection.prepareStatement(sql)) {
            stmt.setString(1, username);
            stmt.setString(2, email);
            stmt.setString(3, BCrypt.hashpw(password, BCrypt.gensalt()));
            stmt.setString(4, bio);
            stmt.setString(5, image);
            stmt.executeUpdate();
        }
    }
}
```

Использование:

```java
@Test
void shouldFollowAnotherUser() {
    // Arrange: создаём двух пользователей через builder
    UserResponse author = UserBuilder.aUser()
            .withUsername("author_jane")
            .createViaApi();

    UserResponse follower = UserBuilder.aUser()
            .withUsername("follower_bob")
            .createViaApi();

    // Act: follower подписывается на author
    given()
            .header("Authorization", "Token " + follower.getToken())
            .when()
            .post("/api/profiles/" + author.getUsername() + "/follow")
            .then()
            .statusCode(200)
            .body("profile.following", equalTo(true));
}
```

### 5.3 Очистка данных после тестов

```java
// Расширение JUnit 5 для очистки данных после каждого теста
public class DatabaseCleanupExtension implements AfterEachCallback {

    // Порядок удаления учитывает foreign key constraints
    private static final List<String> TABLES_TO_CLEAN = List.of(
            "comments", "favorites", "article_tags", "articles", "follows", "users"
    );

    @Override
    public void afterEach(ExtensionContext context) {
        try (Connection conn = getConnection()) {
            for (String table : TABLES_TO_CLEAN) {
                try (Statement stmt = conn.createStatement()) {
                    stmt.execute("DELETE FROM " + table);
                }
            }
        } catch (SQLException e) {
            throw new RuntimeException("Ошибка при очистке тестовых данных", e);
        }
    }

    private Connection getConnection() throws SQLException {
        return DriverManager.getConnection(
                TestConfig.DB_URL, TestConfig.DB_USER, TestConfig.DB_PASSWORD
        );
    }
}

// Использование: просто добавляем аннотацию к тестовому классу
@ExtendWith(DatabaseCleanupExtension.class)
public class ArticleApiTest {
    // После каждого теста таблицы будут очищены
}
```

---

## Часть 6: Анализ Code Coverage с JaCoCo

### 6.1 Подключение JaCoCo

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.12</version>
    <executions>
        <!-- Подготовка агента перед запуском тестов -->
        <execution>
            <id>prepare-agent</id>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <!-- Генерация отчёта после выполнения тестов -->
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
        <!-- Проверка порогов покрытия -->
        <execution>
            <id>check</id>
            <goals>
                <goal>check</goal>
            </goals>
            <configuration>
                <rules>
                    <rule>
                        <element>BUNDLE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.60</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

### 6.2 Интеграция JaCoCo с CI/CD

Добавьте шаг в GitHub Actions workflow:

```yaml
      - name: Generate coverage report
        run: mvn jacoco:report

      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: target/site/jacoco/
          retention-days: 30

      - name: Check coverage threshold
        run: mvn jacoco:check
```

---

## Практические упражнения

### Упражнение 1: TestContainers + PostgreSQL

**Задание:**
1. Добавьте зависимости TestContainers и PostgreSQL в `pom.xml`
2. Создайте файл миграции `init.sql` с таблицами вашего приложения
3. Напишите тест, который вставляет данные в таблицу и проверяет их через SELECT
4. Напишите тест, который вызывает API, а затем проверяет данные в БД

**Чек-лист готовности:**
- [ ] Контейнер PostgreSQL поднимается перед тестами
- [ ] Миграция `init.sql` выполняется автоматически
- [ ] Тесты изолированы (очистка данных между тестами)
- [ ] `DatabaseHelper` предоставляет удобные методы проверки

### Упражнение 2: Docker-compose окружение

**Задание:**
1. Создайте `docker-compose.yml` с PostgreSQL и backend вашего приложения
2. Добавьте healthcheck для каждого сервиса
3. Напишите скрипт `run-tests.sh`, который поднимает окружение и запускает тесты
4. Добавьте Selenoid для UI-тестов

**Чек-лист готовности:**
- [ ] `docker-compose up` поднимает все сервисы
- [ ] Healthcheck корректно определяет готовность сервисов
- [ ] Тесты подключаются к контейнерам и проходят
- [ ] `docker-compose down` корректно останавливает всё

### Упражнение 3: Test Data Builder

**Задание:**
1. Создайте `UserBuilder`, `ArticleBuilder`, `CommentBuilder` для основных сущностей
2. Каждый builder должен генерировать уникальные данные (используйте UUID)
3. Добавьте методы `createViaApi()` и `createInDb()`
4. Перепишите существующие тесты с использованием builder'ов

**Чек-лист готовности:**
- [ ] Builder'ы генерируют уникальные данные при каждом вызове
- [ ] Поддерживается fluent API: `UserBuilder.aUser().withUsername("test").createViaApi()`
- [ ] Тесты стали короче и читабельнее
- [ ] Нет конфликтов данных при параллельном запуске

### Упражнение 4: Мониторинг и метрики

**Задание:**
1. Настройте `categories.json` для классификации падений в Allure
2. Запустите regression 3 раза и проанализируйте тренды в Allure
3. Определите flaky-тесты и добавьте к ним аннотацию `@Flaky`
4. Создайте таблицу метрик тестирования и заполните её по результатам

**Чек-лист готовности:**
- [ ] Категории корректно классифицируют падения в Allure
- [ ] Flaky-тесты идентифицированы и помечены
- [ ] Графики трендов в Allure отображают данные за 3+ запуска
- [ ] Таблица метрик заполнена и отражает реальное состояние

---

## Итоговый чек-лист этапа

После выполнения всех упражнений вы должны иметь:

- [ ] TestContainers-тесты с PostgreSQL для проверки данных в БД
- [ ] `docker-compose.yml` для полного тестового окружения
- [ ] Скрипт `run-tests.sh` для запуска окружения + тестов
- [ ] Test Data Builder'ы для основных сущностей
- [ ] `DatabaseCleanupExtension` для изоляции данных между тестами
- [ ] `categories.json` для классификации падений в Allure
- [ ] JaCoCo-отчёт с анализом code coverage
- [ ] (Опционально) Kafka-тесты с TestContainers
- [ ] (Опционально) Awaitility для проверки асинхронных процессов

**Результат:** полноценный фреймворк enterprise-уровня. С таким проектом в портфолио
вы демонстрируете не только базовые навыки автоматизации, но и понимание работы
с базами данных, контейнеризации, асинхронных процессов и мониторинга качества.
Это уровень Senior QA Automation Engineer.
