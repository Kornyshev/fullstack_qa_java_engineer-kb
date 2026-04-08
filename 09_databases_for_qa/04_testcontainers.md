# TestContainers для QA

## Обзор

TestContainers — это Java-библиотека, которая позволяет запускать Docker-контейнеры
прямо из тестов. Вместо того чтобы настраивать и поддерживать отдельную тестовую базу данных,
QA-инженер получает **свежую, изолированную БД для каждого тестового прогона**, которая
автоматически создаётся и уничтожается.

Это решает ключевые проблемы тестирования с базами данных:
- **«У меня локально не работает»** — контейнер одинаков везде: локально и в CI.
- **Грязные данные от прошлых прогонов** — каждый запуск начинает с чистой БД.
- **Конфликты при параллельном запуске** — каждый тест (или набор тестов) получает свой контейнер.
- **«Нужен DBA, чтобы поднять тестовую базу»** — Docker + TestContainers делают это автоматически.

---

## Предварительные требования

1. **Docker** должен быть установлен и запущен на машине, где выполняются тесты.
2. **Java 17+**.
3. Доступ к Docker Hub (или локальный registry) для скачивания образов.

Проверка Docker:
```bash
docker --version
docker info  # Должен показать информацию о запущенном daemon
```

---

## Подключение зависимостей (Maven)

```xml
<!-- BOM для управления версиями TestContainers -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>testcontainers-bom</artifactId>
            <version>1.20.4</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- Ядро TestContainers -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>testcontainers</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- Поддержка JUnit 5 -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- Модуль PostgreSQL -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>postgresql</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- Модуль MongoDB (если нужен) -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>mongodb</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- PostgreSQL JDBC драйвер -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <version>42.7.4</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

## PostgreSQL контейнер

### Базовый пример

```java
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.junit.jupiter.api.Test;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.sql.Statement;

import static org.assertj.core.api.Assertions.assertThat;

@Testcontainers  // Активирует управление жизненным циклом контейнеров
class PostgresBasicTest {

    // @Container — контейнер создаётся перед тестами и останавливается после
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("testdb")       // Имя базы данных
            .withUsername("test")             // Пользователь
            .withPassword("test");           // Пароль

    @Test
    void shouldConnectToPostgres() throws Exception {
        // Получаем JDBC URL — порт динамический, TestContainers назначает его автоматически
        String jdbcUrl = postgres.getJdbcUrl();
        String username = postgres.getUsername();
        String password = postgres.getPassword();

        try (Connection conn = DriverManager.getConnection(jdbcUrl, username, password);
             Statement stmt = conn.createStatement()) {

            // Создаём тестовую таблицу
            stmt.execute("""
                CREATE TABLE users (
                    id SERIAL PRIMARY KEY,
                    email VARCHAR(255) NOT NULL UNIQUE,
                    status VARCHAR(50) DEFAULT 'ACTIVE'
                )
            """);

            // Вставляем данные
            stmt.execute("INSERT INTO users (email) VALUES ('user@test.com')");

            // Проверяем
            try (ResultSet rs = stmt.executeQuery("SELECT COUNT(*) FROM users")) {
                rs.next();
                assertThat(rs.getInt(1)).isEqualTo(1);
            }
        }
    }
}
```

### С init-скриптом

```java
@Container
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test")
        // SQL-скрипт из classpath, выполняется при старте контейнера
        .withInitScript("db/init.sql");
```

Файл `src/test/resources/db/init.sql`:
```sql
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    first_name VARCHAR(100),
    status VARCHAR(50) DEFAULT 'ACTIVE',
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES users(user_id),
    status VARCHAR(50) DEFAULT 'PENDING',
    total_amount DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Предзагрузка тестовых данных
INSERT INTO users (email, first_name) VALUES
    ('alice@test.com', 'Alice'),
    ('bob@test.com', 'Bob');
```

---

## MongoDB контейнер

```java
import org.testcontainers.containers.MongoDBContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import com.mongodb.client.MongoClient;
import com.mongodb.client.MongoClients;
import com.mongodb.client.MongoCollection;
import com.mongodb.client.MongoDatabase;
import org.bson.Document;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

@Testcontainers
class MongoBasicTest {

    @Container
    static MongoDBContainer mongo = new MongoDBContainer("mongo:7.0");

    @Test
    void shouldInsertAndFindDocument() {
        // Подключение по динамическому URL
        try (MongoClient client = MongoClients.create(mongo.getReplicaSetUrl())) {
            MongoDatabase db = client.getDatabase("testdb");
            MongoCollection<Document> collection = db.getCollection("orders");

            // Вставка документа
            Document order = new Document()
                    .append("orderId", "ORD-001")
                    .append("status", "PENDING")
                    .append("amount", 150.00);
            collection.insertOne(order);

            // Проверка
            Document found = collection.find(new Document("orderId", "ORD-001")).first();
            assertThat(found).isNotNull();
            assertThat(found.getString("status")).isEqualTo("PENDING");
        }
    }
}
```

---

## GenericContainer — произвольный контейнер

Для сервисов, у которых нет специального модуля TestContainers.

```java
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.containers.wait.strategy.Wait;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.junit.jupiter.api.Test;

@Testcontainers
class RedisContainerTest {

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
            .withExposedPorts(6379)  // Порт внутри контейнера
            .waitingFor(Wait.forListeningPort());  // Ждём, пока порт будет доступен

    @Test
    void shouldConnectToRedis() {
        String host = redis.getHost();
        int port = redis.getMappedPort(6379);  // Динамический порт на хосте

        // Подключение к Redis по host:port
        System.out.println("Redis доступен на: " + host + ":" + port);
    }
}
```

---

## Жизненный цикл контейнеров

### static поле + @Container — один контейнер на весь тестовый класс

```java
@Testcontainers
class SharedContainerTest {

    // static = контейнер создаётся один раз для всех тестов в классе
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @Test
    void test1() { /* контейнер уже запущен */ }

    @Test
    void test2() { /* тот же контейнер */ }
    // Контейнер останавливается после всех тестов класса
}
```

### Нестатическое поле + @Container — свой контейнер для каждого теста

```java
@Testcontainers
class IsolatedContainerTest {

    // Без static = новый контейнер для КАЖДОГО теста (медленнее, но полная изоляция)
    @Container
    PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @Test
    void test1() { /* свой контейнер */ }

    @Test
    void test2() { /* другой контейнер */ }
}
```

### Программное управление (без аннотаций)

```java
class ManualContainerTest {

    static PostgreSQLContainer<?> postgres;

    @BeforeAll
    static void startContainer() {
        postgres = new PostgreSQLContainer<>("postgres:16-alpine");
        postgres.start();  // Ручной запуск
    }

    @AfterAll
    static void stopContainer() {
        postgres.stop();  // Ручная остановка
    }
}
```

---

## Миграции: Flyway и Liquibase

Тестовая БД должна иметь ту же схему, что и production. Миграции обеспечивают это.

### Flyway

```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
    <version>10.21.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-database-postgresql</artifactId>
    <version>10.21.0</version>
    <scope>test</scope>
</dependency>
```

```java
import org.flywaydb.core.Flyway;

@Testcontainers
class FlywayMigrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @BeforeAll
    static void migrate() {
        // Применяем миграции из src/main/resources/db/migration
        Flyway flyway = Flyway.configure()
                .dataSource(postgres.getJdbcUrl(), postgres.getUsername(), postgres.getPassword())
                .locations("classpath:db/migration")
                .load();
        flyway.migrate();
    }

    @Test
    void shouldHaveCorrectSchema() throws Exception {
        try (var conn = DriverManager.getConnection(
                postgres.getJdbcUrl(), postgres.getUsername(), postgres.getPassword());
             var stmt = conn.createStatement();
             var rs = stmt.executeQuery(
                 "SELECT table_name FROM information_schema.tables WHERE table_schema = 'public'")) {

            var tables = new java.util.ArrayList<String>();
            while (rs.next()) {
                tables.add(rs.getString("table_name"));
            }
            assertThat(tables).contains("users", "orders", "order_items");
        }
    }
}
```

### Liquibase

```xml
<dependency>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-core</artifactId>
    <version>4.29.2</version>
    <scope>test</scope>
</dependency>
```

```java
import liquibase.Liquibase;
import liquibase.database.DatabaseFactory;
import liquibase.database.jvm.JdbcConnection;
import liquibase.resource.ClassLoaderResourceAccessor;

@BeforeAll
static void migrate() throws Exception {
    var conn = DriverManager.getConnection(
        postgres.getJdbcUrl(), postgres.getUsername(), postgres.getPassword()
    );
    var database = DatabaseFactory.getInstance()
        .findCorrectDatabaseImplementation(new JdbcConnection(conn));
    var liquibase = new Liquibase(
        "db/changelog/db.changelog-master.xml",
        new ClassLoaderResourceAccessor(),
        database
    );
    liquibase.update("");
}
```

---

## Интеграция с Spring Boot

```java
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@SpringBootTest
@Testcontainers
class SpringBootIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("testdb");

    // Динамическая подстановка свойств Spring из контейнера
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        // Spring Boot + Flyway автоматически применит миграции
    }

    @Autowired
    private UserRepository userRepository;

    @Test
    void shouldSaveAndRetrieveUser() {
        User user = new User("integration@test.com", "IntTest");
        userRepository.save(user);

        var found = userRepository.findByEmail("integration@test.com");
        assertThat(found).isPresent();
        assertThat(found.get().getFirstName()).isEqualTo("IntTest");
    }
}
```

### Базовый абстрактный класс для переиспользования

```java
/**
 * Базовый класс для интеграционных тестов.
 * Один контейнер PostgreSQL на все тесты (Singleton pattern).
 */
@SpringBootTest
@Testcontainers
public abstract class BaseIntegrationTest {

    // Singleton-контейнер: запускается один раз для всех наследников
    static final PostgreSQLContainer<?> POSTGRES;

    static {
        POSTGRES = new PostgreSQLContainer<>("postgres:16-alpine")
                .withDatabaseName("testdb")
                .withUsername("test")
                .withPassword("test");
        POSTGRES.start();  // Ручной старт — не используем @Container
    }

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);
    }
}

// Тестовые классы наследуются — контейнер общий
class UserServiceTest extends BaseIntegrationTest {
    @Test
    void testCreateUser() { /* ... */ }
}

class OrderServiceTest extends BaseIntegrationTest {
    @Test
    void testCreateOrder() { /* ... */ }
}
```

---

## Reusable Containers — повторное использование

По умолчанию контейнер останавливается после тестов. Для ускорения локальной разработки
можно оставить его запущенным между прогонами.

### Шаг 1. Файл конфигурации

Создайте файл `~/.testcontainers.properties`:
```properties
testcontainers.reuse.enable=true
```

### Шаг 2. Указать `.withReuse(true)`

```java
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("testdb")
        .withReuse(true);  // Контейнер не останавливается после тестов
```

**Важно:**
- Reusable containers — только для **локальной разработки**, не для CI.
- Состояние данных сохраняется между запусками — нужно очищать в `@BeforeEach`.
- В CI всегда используйте стандартный жизненный цикл (без reuse).

---

## Производительность: советы

| Проблема                                | Решение                                         |
|-----------------------------------------|-------------------------------------------------|
| Контейнер стартует долго (10-20 сек)    | Singleton-паттерн: один контейнер на все тесты  |
| Каждый тест запускает свой контейнер    | Используйте `static` поле с `@Container`        |
| Медленная загрузка образа               | `docker pull` образа заранее в CI pipeline      |
| Миграции выполняются долго              | Singleton + `@BeforeAll` миграция один раз      |
| Тесты в CI без Docker                   | Используйте Docker-in-Docker или remote Docker  |
| Локальная разработка: частые перезапуски | `withReuse(true)` + файл `.testcontainers.properties` |

### Alpine-образы

Всегда предпочитайте Alpine-варианты — они значительно легче:
```java
// Тяжёлый образ (~400 MB)
new PostgreSQLContainer<>("postgres:16")

// Лёгкий образ (~80 MB)
new PostgreSQLContainer<>("postgres:16-alpine")
```

---

## Связь с тестированием

| Задача QA                                    | Как решает TestContainers                       |
|----------------------------------------------|-------------------------------------------------|
| Изолированная БД для тестов                  | Свежий контейнер для каждого прогона            |
| Одинаковое окружение локально и в CI         | Docker-образ гарантирует идентичность           |
| Тестирование миграций                        | Flyway/Liquibase на чистом контейнере           |
| Тестирование с разными версиями СУБД         | Просто меняете тег образа (`postgres:15`, `16`) |
| Параллельный запуск тестовых наборов         | Каждый набор получает свой контейнер            |
| Интеграционные тесты с реальной БД           | Полноценная СУБД вместо H2/mocks                |

---

## Типичные ошибки

1. **Docker не запущен** — `Could not connect to Docker daemon`. Убедитесь, что Docker Desktop
   запущен или `systemctl start docker` на Linux.

2. **Забыть `@Testcontainers`** — без этой аннотации `@Container` не работает, контейнеры
   не стартуют автоматически.

3. **Hardcoded порт** — TestContainers выделяет **случайный** порт. Используйте
   `container.getMappedPort(internalPort)` и `container.getJdbcUrl()`.

4. **Нестатическое поле вместо статического** — новый контейнер для каждого теста.
   Это корректно, но **медленно**. Для большинства случаев достаточно `static`.

5. **Не добавлен модуль СУБД** — `PostgreSQLContainer` требует зависимость
   `testcontainers:postgresql`. Без неё — ошибка компиляции.

6. **Забыть `@DynamicPropertySource` в Spring** — тесты пытаются подключиться к localhost:5432
   вместо динамического порта контейнера.

7. **Reusable containers в CI** — в CI контейнер должен создаваться и уничтожаться каждый раз.
   Reuse может привести к утечке ресурсов и конфликтам между pipeline.

8. **Тяжёлые образы замедляют CI** — используйте Alpine-версии и кэшируйте Docker layers
   в CI pipeline.

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Что такое TestContainers и какую проблему решает?
2. Какие предварительные условия нужны для работы TestContainers?
3. Чем тестирование с TestContainers лучше, чем с H2 in-memory базой?
4. Что делают аннотации `@Testcontainers` и `@Container`?
5. Как получить JDBC URL контейнера PostgreSQL?

### 🟡 Средний уровень
6. Как использовать TestContainers с Spring Boot?
7. Чем отличается `static` и нестатическое поле с `@Container`?
8. Как применить миграции Flyway к контейнерной базе?
9. Что такое Singleton-паттерн для контейнеров и когда его использовать?
10. Как создать контейнер для произвольного сервиса (не из списка модулей)?

### 🔴 Продвинутый уровень
11. Как оптимизировать время старта контейнеров в CI pipeline?
12. Как организовать тестирование с несколькими связанными контейнерами (БД + Redis + Kafka)?
13. Что такое reusable containers? Когда их безопасно использовать?
14. Как тестировать совместимость приложения с разными версиями СУБД через TestContainers?
15. Как обеспечить изоляцию данных при Singleton-контейнере для множества тестовых классов?

---

## Практические задания

### Задание 1. Базовый тест с PostgreSQL

Создайте тест, который поднимает PostgreSQL-контейнер, создаёт таблицу, вставляет запись
и проверяет, что она корректно читается.

<details>
<summary>Решение</summary>

```java
@Testcontainers
class BasicPostgresTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("qa_test");

    @Test
    void shouldInsertAndRead() throws Exception {
        try (var conn = DriverManager.getConnection(
                postgres.getJdbcUrl(), postgres.getUsername(), postgres.getPassword());
             var stmt = conn.createStatement()) {

            stmt.execute("CREATE TABLE products (id SERIAL, name VARCHAR(100), price DECIMAL)");
            stmt.execute("INSERT INTO products (name, price) VALUES ('Widget', 29.99)");

            try (var rs = stmt.executeQuery("SELECT name, price FROM products WHERE id = 1")) {
                assertThat(rs.next()).isTrue();
                assertThat(rs.getString("name")).isEqualTo("Widget");
                assertThat(rs.getBigDecimal("price")).isEqualByComparingTo("29.99");
            }
        }
    }
}
```

</details>

### Задание 2. Init-скрипт и предзагрузка данных

Создайте тест с init-скриптом, который создаёт схему и вставляет тестовые данные.
Проверьте, что данные доступны в тесте.

<details>
<summary>Решение</summary>

```sql
-- src/test/resources/db/test-init.sql
CREATE TABLE users (id SERIAL PRIMARY KEY, email VARCHAR(255) UNIQUE, status VARCHAR(50));
INSERT INTO users (email, status) VALUES ('preloaded@test.com', 'ACTIVE');
```

```java
@Testcontainers
class InitScriptTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
            .withInitScript("db/test-init.sql");

    @Test
    void shouldHavePreloadedData() throws Exception {
        try (var conn = DriverManager.getConnection(
                postgres.getJdbcUrl(), postgres.getUsername(), postgres.getPassword());
             var stmt = conn.createStatement();
             var rs = stmt.executeQuery("SELECT status FROM users WHERE email = 'preloaded@test.com'")) {

            assertThat(rs.next()).isTrue();
            assertThat(rs.getString("status")).isEqualTo("ACTIVE");
        }
    }
}
```

</details>

### Задание 3. Spring Boot интеграция

Напишите базовый абстрактный класс для интеграционных тестов с TestContainers + Spring Boot.

<details>
<summary>Решение</summary>

```java
@SpringBootTest
@Testcontainers
public abstract class AbstractIT {

    static final PostgreSQLContainer<?> POSTGRES;

    static {
        POSTGRES = new PostgreSQLContainer<>("postgres:16-alpine")
                .withDatabaseName("integration_test");
        POSTGRES.start();
    }

    @DynamicPropertySource
    static void dbProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);
        registry.add("spring.jpa.hibernate.ddl-auto", () -> "validate");
        registry.add("spring.flyway.enabled", () -> "true");
    }
}
```

</details>

---

## Дополнительные ресурсы

- [TestContainers docs](https://testcontainers.com/) — официальная документация
- [TestContainers GitHub](https://github.com/testcontainers/testcontainers-java) — исходный код и примеры
- [TestContainers + Spring Boot Guide](https://spring.io/blog/2023/06/23/improved-testcontainers-support-in-spring-boot-3-1) — интеграция со Spring Boot 3.1+
- [Docker Hub](https://hub.docker.com/) — реестр Docker-образов
- [Flyway Documentation](https://documentation.red-gate.com/fd) — миграции базы данных
- [Baeldung: TestContainers](https://www.baeldung.com/spring-boot-testcontainers) — практические руководства
