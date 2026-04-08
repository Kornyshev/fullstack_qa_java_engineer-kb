# Транзакции и ACID

## Обзор

Транзакция — это группа операций с базой данных, которая выполняется **как единое целое**:
либо все операции проходят успешно, либо ни одна из них не применяется. Транзакции — основа
надёжности любой системы, работающей с данными.

Для QA-инженера понимание транзакций критически важно по нескольким причинам:
- **Flaky-тесты** часто вызваны race conditions при параллельном доступе к данным.
- **Баги при конкурентных операциях** (двойные списания, потерянные обновления) возникают
  из-за неправильного уровня изоляции.
- **`@Transactional` в Spring-тестах** — мощный инструмент, который работает неочевидно,
  если не понимать механику транзакций.
- **Тестирование отката** — нужно проверить, что при ошибке частичные данные не остаются в БД.

---

## ACID — свойства транзакций

ACID — это четыре фундаментальных свойства, гарантирующих надёжность транзакций.

### Atomicity (Атомарность)

Транзакция — неделимая единица. Либо выполняются **все** операции, либо **ни одна**.

```sql
-- Перевод денег: списание + зачисление
BEGIN;
    UPDATE accounts SET balance = balance - 1000 WHERE account_id = 1;  -- Списание
    UPDATE accounts SET balance = balance + 1000 WHERE account_id = 2;  -- Зачисление
COMMIT;

-- Если на втором UPDATE произойдёт ошибка — первый UPDATE тоже откатится.
-- Деньги не «исчезнут».
```

**QA-сценарий:** вызвать API перевода средств, но при этом сделать так, чтобы целевой счёт
не существовал. Проверить, что баланс исходного счёта **не изменился** (откат атомарной
транзакции).

### Consistency (Согласованность)

Транзакция переводит базу из одного **корректного состояния** в другое. Все constraints,
triggers и бизнес-правила должны выполняться.

```sql
-- Constraint: баланс не может быть отрицательным
ALTER TABLE accounts ADD CONSTRAINT positive_balance CHECK (balance >= 0);

-- Попытка списать больше, чем есть
BEGIN;
    UPDATE accounts SET balance = balance - 999999 WHERE account_id = 1;
    -- Ошибка! CHECK constraint нарушен. Транзакция откатывается.
COMMIT;
```

**QA-сценарий:** попытаться через API оформить заказ на количество товара, превышающее остаток
на складе. Убедиться, что остаток не стал отрицательным.

### Isolation (Изолированность)

Параллельные транзакции не должны мешать друг другу. Каждая транзакция «видит» данные так,
как будто она выполняется в одиночку. Степень изоляции настраивается
(подробнее — в разделе об уровнях изоляции).

```
Транзакция A                    Транзакция B
─────────────────              ─────────────────
BEGIN;                         BEGIN;
SELECT balance FROM accounts
WHERE id = 1;
-- Видит: 5000
                               UPDATE accounts
                               SET balance = 3000
                               WHERE id = 1;
                               COMMIT;
SELECT balance FROM accounts
WHERE id = 1;
-- Что увидит? Зависит от уровня изоляции!
COMMIT;
```

### Durability (Долговечность)

После `COMMIT` данные **гарантированно сохранены**, даже если сервер упадёт через секунду.
СУБД записывает данные в WAL (Write-Ahead Log) перед подтверждением транзакции.

**QA-сценарий:** после успешного API-ответа о создании заказа перезапустить контейнер с БД
и проверить, что заказ на месте (в реальности такие тесты редки, но понимание важно).

---

## Уровни изоляции транзакций

SQL-стандарт определяет 4 уровня изоляции, каждый из которых балансирует между
**производительностью** и **корректностью**.

### Обзорная таблица

| Уровень            | Dirty Read | Non-Repeatable Read | Phantom Read |
|--------------------|:----------:|:-------------------:|:------------:|
| READ UNCOMMITTED   |   Да       |        Да           |     Да       |
| READ COMMITTED     |   Нет      |        Да           |     Да       |
| REPEATABLE READ    |   Нет      |        Нет          |     Да*      |
| SERIALIZABLE       |   Нет      |        Нет          |     Нет      |

\* В PostgreSQL REPEATABLE READ также предотвращает phantom read.

---

### Dirty Read (Грязное чтение)

Транзакция читает данные, которые **ещё не зафиксированы** другой транзакцией.
Если та транзакция откатится — прочитанные данные окажутся «призрачными».

```
Транзакция A                        Транзакция B
───────────────                    ───────────────
BEGIN;
UPDATE accounts SET balance = 0
WHERE id = 1;
-- balance = 0 (ещё не COMMIT)
                                   BEGIN;
                                   SELECT balance FROM accounts
                                   WHERE id = 1;
                                   -- Dirty Read: видит balance = 0 !!!
                                   COMMIT;
ROLLBACK;
-- balance снова 5000
                                   -- Транзакция B работала с ложными данными
```

**Уровень READ UNCOMMITTED** допускает dirty read. В PostgreSQL этот уровень
автоматически повышается до READ COMMITTED, поэтому dirty read невозможен.

**QA-сценарий:** в системах с MySQL/SQL Server на уровне READ UNCOMMITTED можно
проверить, видит ли отчёт незафиксированные данные. Это баг, если отчёт должен быть точным.

---

### Non-Repeatable Read (Неповторяемое чтение)

Два одинаковых `SELECT` в одной транзакции возвращают **разные значения**,
потому что другая транзакция успела изменить и зафиксировать данные.

```
Транзакция A                        Транзакция B
───────────────                    ───────────────
BEGIN;
SELECT balance FROM accounts
WHERE id = 1;
-- Видит: 5000
                                   BEGIN;
                                   UPDATE accounts SET balance = 3000
                                   WHERE id = 1;
                                   COMMIT;
SELECT balance FROM accounts
WHERE id = 1;
-- Видит: 3000 — значение ИЗМЕНИЛОСЬ!
COMMIT;
```

**Уровень READ COMMITTED** допускает non-repeatable read (это уровень по умолчанию
в PostgreSQL). **REPEATABLE READ** и выше — не допускают.

**QA-сценарий:** тест формирует отчёт, который читает данные дважды (например, сумма
в header и детализация). Если между чтениями данные изменились, отчёт будет
несогласованным. Такие баги ловятся при нагрузочном тестировании.

---

### Phantom Read (Фантомное чтение)

Два одинаковых `SELECT` в одной транзакции возвращают **разное количество строк**,
потому что другая транзакция вставила или удалила строки.

```
Транзакция A                        Транзакция B
───────────────                    ───────────────
BEGIN;
SELECT COUNT(*) FROM orders
WHERE status = 'PENDING';
-- Видит: 10
                                   BEGIN;
                                   INSERT INTO orders (status)
                                   VALUES ('PENDING');
                                   COMMIT;
SELECT COUNT(*) FROM orders
WHERE status = 'PENDING';
-- Видит: 11 — появилась «фантомная» строка!
COMMIT;
```

**SERIALIZABLE** — единственный уровень, полностью исключающий phantom read
(в стандарте SQL; в PostgreSQL REPEATABLE READ тоже защищает).

---

### Установка уровня изоляции

```sql
-- Для текущей транзакции
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- ... операции ...
COMMIT;

-- Глобально для сессии
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

В Java через JDBC:

```java
Connection conn = DriverManager.getConnection(url, user, password);
conn.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED);
conn.setAutoCommit(false);

// Работа с данными...

conn.commit();  // или conn.rollback();
```

Доступные константы:
```java
Connection.TRANSACTION_READ_UNCOMMITTED  // 1
Connection.TRANSACTION_READ_COMMITTED    // 2
Connection.TRANSACTION_REPEATABLE_READ   // 4
Connection.TRANSACTION_SERIALIZABLE      // 8
```

---

## Тестирование конкурентных операций

### Пример: двойное списание

```java
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

@Test
void shouldNotAllowDoubleDebit() throws Exception {
    // Подготовка: баланс = 1000, пытаемся списать 800 дважды одновременно
    db.executeUpdate("UPDATE accounts SET balance = 1000 WHERE account_id = 1");

    int threadCount = 2;
    CountDownLatch startLatch = new CountDownLatch(1);       // Синхронизация старта
    CountDownLatch finishLatch = new CountDownLatch(threadCount);
    var errors = new java.util.concurrent.CopyOnWriteArrayList<Exception>();

    ExecutorService executor = Executors.newFixedThreadPool(threadCount);

    for (int i = 0; i < threadCount; i++) {
        executor.submit(() -> {
            try {
                startLatch.await();  // Ждём сигнала (одновременный старт)

                try (Connection conn = DbConnection.getConnection()) {
                    conn.setAutoCommit(false);
                    conn.setTransactionIsolation(Connection.TRANSACTION_SERIALIZABLE);

                    // Читаем баланс
                    try (var pstmt = conn.prepareStatement(
                            "SELECT balance FROM accounts WHERE account_id = ? FOR UPDATE")) {
                        pstmt.setInt(1, 1);
                        var rs = pstmt.executeQuery();
                        rs.next();
                        int balance = rs.getInt("balance");

                        if (balance >= 800) {
                            // Списываем
                            try (var update = conn.prepareStatement(
                                    "UPDATE accounts SET balance = balance - 800 WHERE account_id = ?")) {
                                update.setInt(1, 1);
                                update.executeUpdate();
                            }
                        }
                    }

                    conn.commit();
                }
            } catch (Exception e) {
                errors.add(e);
            } finally {
                finishLatch.countDown();
            }
        });
    }

    startLatch.countDown();   // Стартуем оба потока одновременно
    finishLatch.await();      // Ждём завершения
    executor.shutdown();

    // Проверка: баланс не должен быть отрицательным
    int finalBalance = db.executeScalar(
        "SELECT balance FROM accounts WHERE account_id = ?", 1
    );
    assertThat(finalBalance).isGreaterThanOrEqualTo(0);

    // Одна операция должна списать 800, вторая — отказать или откатиться
    // Ожидаемый баланс: 200 (одно списание) или 1000 (оба откатились)
    assertThat(finalBalance).isIn(200, 1000);
}
```

### Пример: lost update (потерянное обновление)

```java
@Test
void shouldDetectLostUpdate() throws Exception {
    // Начальное значение: likes = 0
    db.executeUpdate("UPDATE posts SET likes = 0 WHERE post_id = 1");

    int threadCount = 100;
    ExecutorService executor = Executors.newFixedThreadPool(10);
    CountDownLatch latch = new CountDownLatch(threadCount);

    for (int i = 0; i < threadCount; i++) {
        executor.submit(() -> {
            try {
                // Каждый поток увеличивает likes на 1
                // Если не использовать атомарный UPDATE, будут потерянные обновления
                db.executeUpdate(
                    "UPDATE posts SET likes = likes + 1 WHERE post_id = ?", 1
                );
            } finally {
                latch.countDown();
            }
        });
    }

    latch.await();
    executor.shutdown();

    // Если всё корректно, likes должен быть 100
    int likes = db.executeScalar("SELECT likes FROM posts WHERE post_id = ?", 1);
    assertThat(likes).isEqualTo(100);
}
```

---

## @Transactional в Spring-тестах

### Как работает

Аннотация `@Transactional` на тестовом методе (или классе):
1. **Открывает транзакцию** перед тестом.
2. **Выполняет тест** внутри этой транзакции.
3. **Откатывает транзакцию** после теста (по умолчанию).

Это означает автоматическую очистку данных — никакого мусора в БД.

```java
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.transaction.annotation.Transactional;

@SpringBootTest
@Transactional  // Каждый тест оборачивается в транзакцию и откатывается
class UserServiceTransactionalTest {

    @Autowired
    private UserService userService;

    @Autowired
    private UserRepository userRepository;

    @Test
    void shouldCreateUser() {
        userService.createUser("tx_test@test.com", "TxTest");

        // Данные видны внутри транзакции теста
        var user = userRepository.findByEmail("tx_test@test.com");
        assertThat(user).isPresent();
    }
    // После теста — ROLLBACK. Запись "tx_test@test.com" удалена из БД.

    @Test
    void shouldDeleteUser() {
        // Этот тест не видит данных из предыдущего — они откатились
        var user = userRepository.findByEmail("tx_test@test.com");
        assertThat(user).isEmpty();
    }
}
```

### Ловушки @Transactional

#### 1. Тест не видит данные, создаваемые приложением в другой транзакции

```java
@Test
@Transactional
void problematicTest() {
    // Тест работает в Транзакции A

    // API-вызов создаёт данные в Транзакции B (собственная транзакция приложения)
    restTemplate.postForEntity("/api/users", requestBody, Void.class);

    // Транзакция B зафиксирована, но Транзакция A (теста) не видит эти данные
    // из-за изоляции транзакций! (READ COMMITTED по умолчанию)
    var user = userRepository.findByEmail("new@test.com");
    // user может быть ПУСТЫМ — flaky test!
}
```

**Решение:** для интеграционных тестов с реальными HTTP-вызовами **не используйте**
`@Transactional`. Используйте явную очистку данных в `@AfterEach`.

#### 2. LazyInitializationException маскируется

```java
@Test
@Transactional  // Hibernate-сессия открыта на протяжении всего теста
void maskedLazyLoading() {
    User user = userRepository.findById(1L).orElseThrow();
    // В прод-коде user.getOrders() вызвал бы LazyInitializationException
    // Но в тесте с @Transactional сессия открыта — всё работает!
    List<Order> orders = user.getOrders();  // Работает в тесте, падает в проде!
}
```

**Решение:** не полагайтесь на `@Transactional` в тестах для проверки lazy-загрузки.
Тестируйте через реальные HTTP-вызовы.

#### 3. Явная фиксация

```java
@Test
@Transactional
@Rollback(false)  // Или @Commit — данные НЕ откатываются
void shouldPersistData() {
    userService.createUser("persist@test.com", "Persist");
    // Данные останутся в БД после теста!
}
```

---

## Транзакции и flaky-тесты

### Почему тесты становятся flaky из-за транзакций

1. **Race condition в тестовых данных.** Два теста параллельно создают пользователя с одним email.
   Один из них получает unique constraint violation.

2. **Тест зависит от порядка выполнения.** Тест A создаёт данные, тест B на них рассчитывает.
   При параллельном запуске тест B может выполниться раньше.

3. **Timeout при блокировке.** Тест A блокирует строку (`SELECT ... FOR UPDATE`), тест B
   ждёт освобождения и получает timeout.

4. **Незафиксированные данные.** Тест вставляет данные, но не фиксирует транзакцию.
   Другой тест (в другом соединении) не видит эти данные.

### Стратегии борьбы с flaky-тестами

```java
// 1. Уникальные данные для каждого теста
String uniqueEmail = "test_" + UUID.randomUUID() + "@test.com";

// 2. Идемпотентная подготовка данных
@BeforeEach
void setup() {
    db.executeUpdate("DELETE FROM orders WHERE customer_id = ?", TEST_ID);
    db.executeUpdate(
        "INSERT INTO orders (order_id, customer_id, status) VALUES (?, ?, ?) " +
        "ON CONFLICT (order_id) DO NOTHING",
        TEST_ORDER_ID, TEST_ID, "PENDING"
    );
}

// 3. Retry при ожидаемых конфликтах
Awaitility.await()
    .atMost(Duration.ofSeconds(5))
    .pollInterval(Duration.ofMillis(200))
    .ignoreExceptions()
    .until(() -> {
        var rows = db.executeQuery("SELECT status FROM orders WHERE order_id = ?", orderId);
        return !rows.isEmpty() && "PROCESSED".equals(rows.get(0).get("status"));
    });
```

---

## Связь с тестированием

| Задача QA                                      | Связь с транзакциями                             |
|------------------------------------------------|--------------------------------------------------|
| Тестирование отката при ошибке                 | Проверка Atomicity: частичные данные не остаются |
| Тестирование параллельных покупок              | Проверка Isolation: нет двойного списания        |
| Flaky-тесты при параллельном запуске           | Часто вызваны race conditions между транзакциями |
| Автоочистка данных после теста                 | `@Transactional` + rollback в Spring             |
| Тестирование отчётов при активной нагрузке     | Non-repeatable read / phantom read               |
| Проверка устойчивости к сбоям                  | Durability: данные не теряются после crash        |

---

## Типичные ошибки

1. **Не понимать, что `@Transactional` откатывает данные** — удивляться, что тестовые данные
   «исчезают» после теста. Это intended behavior.

2. **Использовать `@Transactional` в интеграционных тестах с HTTP-вызовами** — тест и приложение
   работают в разных транзакциях. Тест не видит данные приложения.

3. **Не использовать `FOR UPDATE` при тестировании конкурентности** — без блокировки два потока
   могут прочитать одно и то же значение и оба его перезаписать (lost update).

4. **Забыть про `autoCommit`** — по умолчанию JDBC-соединение работает в режиме auto-commit:
   каждый `executeUpdate()` — отдельная транзакция. Для группировки операций нужно
   `conn.setAutoCommit(false)`.

5. **Тестировать только «happy path» транзакций** — проверять, что commit работает,
   но не проверять rollback при ошибке. Нужно явно провоцировать сбой и проверять откат.

6. **Hardcoded `Thread.sleep()` вместо polling** — ожидание асинхронной транзакции через
   фиксированный sleep ненадёжно и медленно. Используйте Awaitility.

7. **Не учитывать deadlock** — при тестировании конкурентных операций два потока могут
   заблокировать друг друга. Тест должен обрабатывать `SQLTransactionRollbackException`.

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Что такое транзакция? Приведите пример из реальной жизни.
2. Расшифруйте ACID. Что означает каждое свойство?
3. Что произойдёт, если в середине транзакции возникнет ошибка?
4. Что такое `COMMIT` и `ROLLBACK`?
5. Зачем QA-инженеру знать про транзакции?

### 🟡 Средний уровень
6. Какие уровни изоляции транзакций существуют? Какой используется по умолчанию в PostgreSQL?
7. Объясните разницу между dirty read, non-repeatable read и phantom read.
8. Как `@Transactional` работает в Spring-тестах? Зачем используется?
9. Почему `@Transactional` в тесте может маскировать реальные баги?
10. Как тестировать, что система корректно откатывает транзакцию при ошибке?

### 🔴 Продвинутый уровень
11. Как протестировать race condition при конкурентном доступе к данным?
12. Почему `@Transactional` в интеграционном тесте с HTTP-вызовами не работает как ожидается?
13. Что такое lost update? Как его протестировать и предотвратить?
14. Как уровень изоляции влияет на производительность и корректность? Как выбрать подходящий?
15. Что такое deadlock? Как тестировать систему на устойчивость к deadlock?

---

## Практические задания

### Задание 1. Проверка атомарности
Напишите тест: попробуйте перевести деньги на несуществующий счёт. Убедитесь, что
баланс исходного счёта не изменился.

<details>
<summary>Решение</summary>

```java
@Test
void transferToNonExistentAccount_shouldRollback() throws Exception {
    // Начальный баланс
    int initialBalance = db.executeScalar(
        "SELECT balance FROM accounts WHERE account_id = ?", 1
    );

    try (Connection conn = DbConnection.getConnection()) {
        conn.setAutoCommit(false);
        try {
            // Списание
            try (var pstmt = conn.prepareStatement(
                    "UPDATE accounts SET balance = balance - 500 WHERE account_id = ?")) {
                pstmt.setInt(1, 1);
                pstmt.executeUpdate();
            }

            // Зачисление на несуществующий счёт — вызовет ошибку FK
            try (var pstmt = conn.prepareStatement(
                    "UPDATE accounts SET balance = balance + 500 WHERE account_id = ?")) {
                pstmt.setInt(1, 99999);
                int updated = pstmt.executeUpdate();
                if (updated == 0) {
                    throw new RuntimeException("Целевой счёт не найден");
                }
            }

            conn.commit();
        } catch (Exception e) {
            conn.rollback();  // Откат: списание отменяется
        }
    }

    // Баланс не должен измениться
    int finalBalance = db.executeScalar(
        "SELECT balance FROM accounts WHERE account_id = ?", 1
    );
    assertThat(finalBalance).isEqualTo(initialBalance);
}
```

</details>

### Задание 2. Тестирование конкурентных операций
Напишите тест, который проверяет, что 50 одновременных попыток купить последний
товар на складе не приведут к отрицательному остатку.

<details>
<summary>Решение</summary>

```java
@Test
void concurrentPurchase_shouldNotOversell() throws Exception {
    // Подготовка: 1 единица товара на складе
    db.executeUpdate("UPDATE products SET stock = 1 WHERE product_id = 1");

    int threadCount = 50;
    CountDownLatch startLatch = new CountDownLatch(1);
    CountDownLatch finishLatch = new CountDownLatch(threadCount);
    var successCount = new java.util.concurrent.atomic.AtomicInteger(0);

    ExecutorService executor = Executors.newFixedThreadPool(threadCount);

    for (int i = 0; i < threadCount; i++) {
        executor.submit(() -> {
            try {
                startLatch.await();
                try (Connection conn = DbConnection.getConnection()) {
                    conn.setAutoCommit(false);
                    try (var pstmt = conn.prepareStatement(
                            "UPDATE products SET stock = stock - 1 " +
                            "WHERE product_id = 1 AND stock > 0")) {
                        int updated = pstmt.executeUpdate();
                        if (updated > 0) {
                            successCount.incrementAndGet();
                        }
                    }
                    conn.commit();
                }
            } catch (Exception e) {
                // Конфликт — ожидаемо
            } finally {
                finishLatch.countDown();
            }
        });
    }

    startLatch.countDown();
    finishLatch.await();
    executor.shutdown();

    // Только одна покупка должна быть успешной
    assertThat(successCount.get()).isEqualTo(1);

    // Остаток не отрицательный
    int stock = db.executeScalar("SELECT stock FROM products WHERE product_id = ?", 1);
    assertThat(stock).isEqualTo(0);
}
```

</details>

### Задание 3. @Transactional в Spring-тесте
Объясните, почему следующий тест может быть flaky, и предложите исправление.

```java
@Test
@Transactional
void shouldCreateOrderViaApi() {
    restTemplate.postForEntity("/api/orders", request, Void.class);
    var order = orderRepository.findByCustomerId(customerId);
    assertThat(order).isNotEmpty();  // Иногда пустой!
}
```

<details>
<summary>Решение</summary>

Проблема: тест работает в Транзакции A, а HTTP-вызов `/api/orders` создаёт данные
в Транзакции B (собственная транзакция приложения). Транзакция A не видит данные
Транзакции B из-за изоляции (READ COMMITTED).

Исправление — убрать `@Transactional` и делать очистку явно:

```java
@Test
void shouldCreateOrderViaApi() {
    restTemplate.postForEntity("/api/orders", request, Void.class);
    var order = orderRepository.findByCustomerId(customerId);
    assertThat(order).isNotEmpty();
}

@AfterEach
void cleanup() {
    orderRepository.deleteByCustomerId(customerId);
}
```

</details>

---

## Дополнительные ресурсы

- [PostgreSQL: Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html) — официальная документация по уровням изоляции
- [Vlad Mihalcea: A beginner's guide to database locking](https://vladmihalcea.com/a-beginners-guide-to-database-locking-and-the-lost-update-phenomena/) — подробно о блокировках и lost update
- [Spring @Transactional in Tests](https://docs.spring.io/spring-framework/reference/testing/testcontext-framework/tx.html) — как Spring управляет транзакциями в тестах
- [JCIP (Java Concurrency in Practice)](https://jcip.net/) — классическая книга о конкурентности в Java
- [Awaitility](https://github.com/awaitility/awaitility) — библиотека для ожидания асинхронных условий в тестах
- [Baeldung: Transaction Isolation Levels](https://www.baeldung.com/cs/transaction-isolation-levels) — наглядное объяснение уровней изоляции
