# Многопоточность — обзор для QA

## Обзор

Многопоточность (multithreading) — это способность программы выполнять несколько потоков (threads) одновременно. Для QA-инженера эта тема критически важна не столько для написания многопоточного кода, сколько для **понимания, почему тесты падают при параллельном запуске**. Когда вы запускаете 50 UI-тестов параллельно и каждый использует свой WebDriver, без понимания потокобезопасности вы получите хаотичные падения: один тест закрывает браузер другого, общие переменные перезаписываются, данные «утекают» между тестами. Эта глава даёт QA-инженеру необходимый минимум для понимания проблем параллельного выполнения и их решения.

---

## Thread и Runnable — основы

### Что такое поток (Thread)

Поток — это независимая единица выполнения внутри процесса. Каждая Java-программа начинается с одного потока (`main`). Можно создать дополнительные потоки для параллельного выполнения задач.

```
Процесс (JVM)
├── main thread (основной поток)
├── Thread-1 (тест A — Chrome)
├── Thread-2 (тест B — Chrome)
└── Thread-3 (тест C — Chrome)
```

### Создание потоков

```java
// Способ 1: Наследование от Thread
class TestRunner extends Thread {
    private final String testName;

    public TestRunner(String testName) {
        this.testName = testName;
    }

    @Override
    public void run() {
        // Код, выполняемый в отдельном потоке
        System.out.println("Запуск теста: " + testName
            + " в потоке: " + Thread.currentThread().getName());
    }
}

// Запуск
TestRunner runner = new TestRunner("LoginTest");
runner.start(); // start() создаёт новый поток и вызывает run()
// Внимание: runner.run() — НЕ создаст новый поток! Выполнится в текущем.

// Способ 2: Реализация Runnable (предпочтительный)
Runnable task = () -> {
    System.out.println("Выполняется в потоке: "
        + Thread.currentThread().getName());
};

Thread thread = new Thread(task);
thread.start();

// Способ 3: Через Callable (возвращает результат)
Callable<String> testTask = () -> {
    // Выполнение теста
    return "PASSED";
};
```

### Почему Runnable лучше наследования от Thread

| Критерий | `extends Thread` | `implements Runnable` |
|----------|------------------|----------------------|
| Наследование | Занимает единственное наследование | Класс может наследовать другой |
| Переиспользование | Жёсткая привязка к Thread | Можно передать в ExecutorService |
| Гибкость | Менее гибкий | Более гибкий |

---

## Проблемы многопоточности

### Race Condition (состояние гонки)

Возникает, когда два потока одновременно обращаются к общему ресурсу и хотя бы один из них его изменяет. Результат непредсказуем.

```java
// ПРОБЛЕМА: общий счётчик без синхронизации
public class TestCounter {
    private int passedTests = 0; // Общий ресурс

    public void incrementPassed() {
        passedTests++; // НЕ атомарная операция!
        // На самом деле: 1) прочитать значение 2) увеличить 3) записать
        // Если два потока выполнят это одновременно:
        // Поток 1: читает 5
        // Поток 2: читает 5
        // Поток 1: записывает 6
        // Поток 2: записывает 6 (а должно быть 7!)
    }

    public int getPassedTests() {
        return passedTests;
    }
}
```

**Реальный пример в тестах:** несколько тестов параллельно пишут в общий отчёт или обновляют общий статус — данные теряются или перезаписываются.

### Deadlock (взаимная блокировка)

Два потока ждут друг друга и ни один не может продолжить выполнение.

```java
// ПРОБЛЕМА: deadlock
Object lockA = new Object();
Object lockB = new Object();

// Поток 1
new Thread(() -> {
    synchronized (lockA) {
        // Захватил lockA, пытается захватить lockB
        synchronized (lockB) {
            System.out.println("Поток 1 работает");
        }
    }
}).start();

// Поток 2
new Thread(() -> {
    synchronized (lockB) {
        // Захватил lockB, пытается захватить lockA
        synchronized (lockA) {
            System.out.println("Поток 2 работает");
        }
    }
}).start();

// Поток 1 ждёт lockB, который захвачен Потоком 2
// Поток 2 ждёт lockA, который захвачен Потоком 1
// Оба потока заблокированы навсегда!
```

**Реальный пример:** два теста одновременно блокируют общие ресурсы (файл отчёта + подключение к БД) в разном порядке.

### Visibility Problem (проблема видимости)

Изменения переменной в одном потоке могут быть не видны другому потоку из-за кэширования процессором.

```
Поток 1: записывает flag = true  → сохраняет в кэше CPU
Поток 2: читает flag             → видит false (читает из своего кэша)
```

---

## Synchronized — базовая синхронизация

Ключевое слово `synchronized` гарантирует, что только один поток может выполнять блок кода одновременно.

### Синхронизированный метод

```java
public class ThreadSafeCounter {
    private int count = 0;

    // Только один поток может выполнять этот метод одновременно
    public synchronized void increment() {
        count++;
    }

    public synchronized int getCount() {
        return count;
    }
}
```

### Синхронизированный блок

```java
public class TestResultCollector {
    private final List<String> results = new ArrayList<>();
    private final Object lock = new Object();

    public void addResult(String result) {
        // Синхронизируем только критическую секцию
        synchronized (lock) {
            results.add(result);
        }
        // Код вне synchronized может выполняться параллельно
        System.out.println("Добавлен результат: " + result);
    }

    public List<String> getResults() {
        synchronized (lock) {
            return new ArrayList<>(results); // Возвращаем копию
        }
    }
}
```

**Правило для QA:** если несколько тестовых потоков обращаются к общему ресурсу (список результатов, файл, счётчик), доступ нужно синхронизировать.

---

## Volatile — видимость между потоками

`volatile` гарантирует, что значение переменной всегда читается из основной памяти, а не из кэша потока.

```java
public class TestExecutionFlag {
    // Без volatile другие потоки могут не увидеть изменение
    private volatile boolean shouldStop = false;

    public void stopExecution() {
        shouldStop = true; // Запись сразу видна всем потокам
    }

    public void runTests() {
        while (!shouldStop) { // Чтение из основной памяти
            // Выполняем следующий тест
            executeNextTest();
        }
        System.out.println("Выполнение остановлено");
    }
}
```

### Volatile vs Synchronized

| Критерий | `volatile` | `synchronized` |
|----------|-----------|---------------|
| Гарантирует видимость | Да | Да |
| Гарантирует атомарность | Нет | Да |
| Блокирует потоки | Нет | Да |
| Применяется к | Только к переменным | К методам и блокам |
| Подходит для | Флаги, простые чтения/записи | Составные операции (read-modify-write) |

**Для QA:** `volatile` достаточен для простых флагов (запуск/остановка), `synchronized` нужен для счётчиков и коллекций.

---

## ThreadLocal — ключевой инструмент для параллельных тестов

`ThreadLocal` — это переменная, у которой **у каждого потока своя копия**. Это главный инструмент для обеспечения потокобезопасности в тестовых фреймворках.

### Проблема: общий WebDriver

```java
// ПРОБЛЕМА: один WebDriver на все потоки
public class BaseTest {
    // ВСЕ потоки используют один и тот же драйвер!
    private static WebDriver driver;

    @BeforeEach
    void setUp() {
        driver = new ChromeDriver(); // Перезаписывает для всех потоков!
    }

    @AfterEach
    void tearDown() {
        driver.quit(); // Закрывает браузер другого теста!
    }
}
```

### Решение: ThreadLocal WebDriver

```java
// РЕШЕНИЕ: каждый поток имеет свой WebDriver
public class BaseTest {
    // У каждого потока СВОЯ копия WebDriver
    private static final ThreadLocal<WebDriver> driverThreadLocal = new ThreadLocal<>();

    @BeforeEach
    void setUp() {
        WebDriver driver = new ChromeDriver();
        driverThreadLocal.set(driver); // Сохраняем драйвер для текущего потока
    }

    protected WebDriver getDriver() {
        return driverThreadLocal.get(); // Возвращает драйвер текущего потока
    }

    @AfterEach
    void tearDown() {
        WebDriver driver = driverThreadLocal.get();
        if (driver != null) {
            driver.quit();
            driverThreadLocal.remove(); // Очищаем, чтобы избежать утечки памяти
        }
    }
}

// Каждый тест получает свой браузер
class LoginTest extends BaseTest {
    @Test
    void shouldLogin() {
        getDriver().get("https://example.com/login");
        // Работает только с СВОИМ браузером
    }
}
```

### ThreadLocal для других тестовых ресурсов

```java
// ThreadLocal для хранения тестовых данных, специфичных для потока
public class TestContext {
    private static final ThreadLocal<Map<String, Object>> context =
        ThreadLocal.withInitial(HashMap::new);

    public static void set(String key, Object value) {
        context.get().put(key, value);
    }

    @SuppressWarnings("unchecked")
    public static <T> T get(String key) {
        return (T) context.get().get(key);
    }

    public static void clear() {
        context.get().clear();
        context.remove(); // Обязательно вызывать для предотвращения утечки
    }
}

// Использование в тестах
@BeforeEach
void setUp() {
    TestContext.set("authToken", getAuthToken());
    TestContext.set("userId", createTestUser());
}

@Test
void shouldGetUserProfile() {
    String token = TestContext.get("authToken");
    Long userId = TestContext.get("userId");
    // Данные принадлежат только текущему потоку
}

@AfterEach
void tearDown() {
    TestContext.clear(); // Очистка после теста
}
```

### Важно: утечка памяти в ThreadLocal

```
ThreadLocal → Thread → ThreadLocalMap → значение

Если не вызвать remove(), значение остаётся в памяти,
пока поток жив (в thread pool потоки живут долго!).

ВСЕГДА вызывайте threadLocal.remove() в @AfterEach!
```

---

## ExecutorService — управление пулом потоков

`ExecutorService` — это абстракция для управления потоками. Вместо создания потоков вручную вы отправляете задачи в пул.

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

// Создаём пул из 4 потоков
ExecutorService executor = Executors.newFixedThreadPool(4);

// Отправляем задачи
List<Future<String>> futures = new ArrayList<>();
for (String testName : testNames) {
    Future<String> future = executor.submit(() -> {
        // Каждая задача выполняется в одном из 4 потоков
        return runTest(testName);
    });
    futures.add(future);
}

// Получаем результаты
for (Future<String> future : futures) {
    String result = future.get(); // Ожидаем завершения задачи
    System.out.println("Результат: " + result);
}

// Завершаем работу пула
executor.shutdown();
```

### Типы ExecutorService

| Тип | Описание | Когда использовать |
|-----|---------|-------------------|
| `newFixedThreadPool(n)` | Фиксированный пул из n потоков | Параллельное выполнение тестов с ограничением |
| `newCachedThreadPool()` | Динамический пул, создаёт потоки по запросу | Непредсказуемое количество задач |
| `newSingleThreadExecutor()` | Один поток, задачи в очереди | Последовательное выполнение с очередью |
| `newScheduledThreadPool(n)` | Пул для отложенного и периодического выполнения | Таймеры, периодические проверки |

**Для QA:** JUnit 5 и TestNG используют thread pools для параллельного выполнения тестов. Понимание ExecutorService помогает настроить `junit-platform.properties`:

```properties
# Параллельное выполнение тестов в JUnit 5
junit.jupiter.execution.parallel.enabled=true
junit.jupiter.execution.parallel.mode.default=concurrent
junit.jupiter.execution.parallel.config.strategy=fixed
junit.jupiter.execution.parallel.config.fixed.parallelism=4
```

---

## ConcurrentHashMap vs HashMap

### Проблема с HashMap в параллельных тестах

```java
// ПРОБЛЕМА: HashMap НЕ потокобезопасен
private static Map<String, String> testResults = new HashMap<>();

// Если несколько потоков одновременно добавляют результаты:
// - ConcurrentModificationException
// - Потеря данных
// - Бесконечный цикл (в старых версиях Java)
```

### Решение: ConcurrentHashMap

```java
// РЕШЕНИЕ: ConcurrentHashMap — потокобезопасен
private static final Map<String, String> testResults = new ConcurrentHashMap<>();

// Несколько потоков могут безопасно работать с этой Map
testResults.put("LoginTest", "PASSED");   // Поток 1
testResults.put("OrderTest", "FAILED");   // Поток 2 — безопасно!
```

### Сравнение коллекций

| Коллекция | Потокобезопасность | Производительность | Когда использовать |
|-----------|-------------------|-------------------|-------------------|
| `HashMap` | Нет | Высокая | Однопоточные тесты |
| `ConcurrentHashMap` | Да | Высокая (сегментная блокировка) | Параллельные тесты, общие данные |
| `Collections.synchronizedMap()` | Да | Средняя (полная блокировка) | Редко — ConcurrentHashMap лучше |
| `ArrayList` | Нет | Высокая | Однопоточные тесты |
| `CopyOnWriteArrayList` | Да | Низкая при записи, высокая при чтении | Много чтений, мало записей |
| `ConcurrentLinkedQueue` | Да | Высокая | Очередь задач между потоками |

---

## Потокобезопасность в тестовом фреймворке

### Полный пример: потокобезопасный базовый тестовый класс

```java
import lombok.extern.slf4j.Slf4j;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;

@Slf4j
public abstract class BaseTest {

    // У каждого потока свой WebDriver
    private static final ThreadLocal<WebDriver> driverHolder = new ThreadLocal<>();

    // Общий счётчик тестов — потокобезопасный
    private static final AtomicInteger testCounter = new AtomicInteger(0);

    // Общие результаты — потокобезопасная коллекция
    private static final Map<String, String> results = new ConcurrentHashMap<>();

    @BeforeEach
    void setUp(TestInfo testInfo) {
        // Создаём WebDriver для текущего потока
        WebDriver driver = new ChromeDriver();
        driverHolder.set(driver);

        int testNumber = testCounter.incrementAndGet();
        log.info("[Поток {}] Тест #{}: {} — запуск",
            Thread.currentThread().getName(), testNumber,
            testInfo.getDisplayName());
    }

    protected WebDriver getDriver() {
        return driverHolder.get();
    }

    @AfterEach
    void tearDown(TestInfo testInfo) {
        try {
            WebDriver driver = driverHolder.get();
            if (driver != null) {
                driver.quit();
            }
        } finally {
            driverHolder.remove(); // Предотвращаем утечку памяти
            log.info("[Поток {}] Тест: {} — завершён",
                Thread.currentThread().getName(),
                testInfo.getDisplayName());
        }
    }

    protected void saveResult(String testName, String result) {
        results.put(testName, result); // ConcurrentHashMap — потокобезопасно
    }
}
```

### Потокобезопасный REST Assured

```java
@Slf4j
public abstract class BaseApiTest {

    // ThreadLocal для хранения auth-токена каждого потока
    private static final ThreadLocal<String> authTokenHolder = new ThreadLocal<>();

    @BeforeEach
    void setUp() {
        // Каждый поток получает свой auth-токен
        String token = AuthService.getNewToken();
        authTokenHolder.set(token);
        log.info("[{}] Получен токен для потока", Thread.currentThread().getName());
    }

    protected RequestSpecification authorizedRequest() {
        return given()
            .header("Authorization", "Bearer " + authTokenHolder.get())
            .contentType(ContentType.JSON);
    }

    @AfterEach
    void tearDown() {
        authTokenHolder.remove();
    }
}
```

---

## Типичные проблемы параллельных тестов

### Проблема 1: Shared State (общее состояние)

```java
// ПРОБЛЕМНЫЙ код
class UserTest {
    private static String createdUserId; // Общее поле для всех тестов!

    @Test
    void shouldCreateUser() {
        createdUserId = api.createUser("Иван"); // Поток 1 записал "user-1"
    }

    @Test
    void shouldDeleteUser() {
        api.deleteUser(createdUserId); // Поток 2 прочитал "user-1", хотя создавал "user-2"
    }
}
```

**Решение:** каждый тест должен создавать свои данные и не зависеть от других тестов.

### Проблема 2: Database Conflicts

```java
// ПРОБЛЕМА: тесты используют одни и те же данные в БД
@Test
void shouldUpdateUserEmail() {
    // Оба потока пытаются обновить одного и того же пользователя
    api.updateUser(1, "new-email@test.com"); // Конфликт!
}
```

**Решение:** использовать уникальные данные для каждого теста:
```java
@Test
void shouldUpdateUserEmail() {
    // Создаём уникального пользователя для ЭТОГО теста
    Long userId = api.createUser("user_" + UUID.randomUUID());
    api.updateUser(userId, "new-email@test.com"); // Нет конфликта
}
```

### Проблема 3: File System Conflicts

```java
// ПРОБЛЕМА: все потоки пишут в один файл
FileWriter writer = new FileWriter("test-results.txt");
writer.write(testResult); // Данные перемешиваются или теряются
```

**Решение:** уникальные файлы или потокобезопасная запись:
```java
// Уникальный файл для каждого потока
String fileName = "test-results-" + Thread.currentThread().getName() + ".txt";

// Или потокобезопасная запись
synchronized (fileLock) {
    Files.writeString(path, testResult + "\n", StandardOpenOption.APPEND);
}
```

---

## Связь с тестированием

1. **Параллельное выполнение тестов** — главная причина, по которой QA должен понимать многопоточность. JUnit 5 и TestNG поддерживают параллельный запуск, и без потокобезопасного кода тесты будут нестабильными.

2. **ThreadLocal для WebDriver** — стандартное решение в Selenium-фреймворках. Каждый тестовый поток получает свой экземпляр браузера.

3. **Flaky tests** — нестабильные тесты часто являются результатом проблем с многопоточностью: race conditions при доступе к общим данным, неправильная изоляция тестовых данных.

4. **CI/CD pipeline** — параллельный запуск тестов значительно ускоряет пайплайн. Без потокобезопасного фреймворка параллелизация невозможна.

5. **Тестирование многопоточного приложения** — QA должен понимать, что race conditions и deadlocks — это типы багов, которые нужно уметь воспроизводить и описывать.

---

## Типичные ошибки

1. **Используют `static` поля для хранения данных между тестами.** При параллельном запуске `static` поле — это общий ресурс для всех потоков. Результат: тесты влияют друг на друга, хаотичные падения.

2. **Забывают `threadLocal.remove()` в `@AfterEach`.** Это приводит к утечке памяти: значения остаются привязанными к потокам пула, которые переиспользуются.

3. **Используют `HashMap` вместо `ConcurrentHashMap`** для хранения общих результатов тестов. При параллельной записи получают `ConcurrentModificationException` или потерю данных.

4. **Тесты зависят от порядка выполнения.** В параллельном режиме порядок не гарантирован. Каждый тест должен быть полностью независимым: создавать свои данные, не полагаться на результаты других тестов.

---

## Вопросы на интервью

- 🟢 **Q:** Что такое Thread и как создать поток в Java?
- **A:** Thread — единица выполнения внутри процесса. Создать можно: наследование от `Thread` и переопределение `run()`, реализация `Runnable` интерфейса, через `ExecutorService`. Предпочтительный способ — `Runnable` или `ExecutorService`.

- 🟢 **Q:** Что такое race condition? Приведите пример из тестирования.
- **A:** Race condition — ситуация, когда результат зависит от порядка выполнения потоков. Пример: два теста параллельно создают пользователя с одним email — один получает 409 Conflict. Или два потока увеличивают общий счётчик — результат непредсказуем.

- 🟢 **Q:** Зачем нужен ThreadLocal в тестовом фреймворке?
- **A:** `ThreadLocal` хранит отдельную копию переменной для каждого потока. Используется для WebDriver в параллельных UI-тестах: каждый поток работает со своим браузером. Без `ThreadLocal` один тест может закрыть браузер другого.

- 🟡 **Q:** В чём разница между `synchronized` и `volatile`?
- **A:** `synchronized` блокирует доступ к блоку кода (один поток за раз), гарантирует атомарность и видимость. `volatile` только гарантирует видимость (чтение из основной памяти), но не атомарность. Для счётчиков нужен `synchronized` или `AtomicInteger`, для флагов достаточно `volatile`.

- 🟡 **Q:** Чем `ConcurrentHashMap` отличается от `HashMap`? Когда что использовать в тестах?
- **A:** `HashMap` — не потокобезопасен, при параллельной записи может дать `ConcurrentModificationException` или потерю данных. `ConcurrentHashMap` — потокобезопасен с сегментной блокировкой, высокая производительность. В параллельных тестах для общих данных — только `ConcurrentHashMap`.

- 🟡 **Q:** Почему тесты нестабильны при параллельном запуске? Как это исправить?
- **A:** Основные причины: shared state (общие `static` поля), один WebDriver на все потоки, зависимость от порядка выполнения, конфликты в БД. Решения: `ThreadLocal` для изоляции, `ConcurrentHashMap` для общих коллекций, уникальные тестовые данные, независимость каждого теста.

- 🟡 **Q:** Что такое deadlock? Может ли он возникнуть в тестах?
- **A:** Deadlock — два потока блокируют ресурсы, ожидая друг друга. В тестах: два теста одновременно блокируют разные записи в БД в обратном порядке, или тестовый фреймворк неправильно использует синхронизацию. Диагностика: thread dump, анализ стектрейсов.

- 🔴 **Q:** Как настроить параллельное выполнение тестов в JUnit 5?
- **A:** Через `junit-platform.properties`: включить `parallel.enabled=true`, настроить `mode.default=concurrent`, выбрать стратегию (`fixed` или `dynamic`), указать количество потоков. Код должен быть потокобезопасным: `ThreadLocal` для состояния, `ConcurrentHashMap` для общих данных.

- 🔴 **Q:** Объясните, как устроен ThreadLocal. Почему важно вызывать `remove()`?
- **A:** Каждый `Thread` содержит `ThreadLocalMap`, где ключ — `ThreadLocal`-объект, значение — данные. `remove()` удаляет запись из карты. Без `remove()` в thread pool (где потоки переиспользуются) значения «утекают» — следующий тест в том же потоке может прочитать чужие данные, плюс утечка памяти.

---

## Практические задания

### Задание 1 (базовое): Воспроизведение race condition

Создайте класс `SharedCounter` с методом `increment()` (без синхронизации). Запустите 10 потоков, каждый увеличивает счётчик 1000 раз. Проверьте, что итоговое значение не равно 10000. Затем добавьте `synchronized` и убедитесь, что значение корректно.

```java
class SharedCounter {
    private int count = 0;

    // Без synchronized — race condition
    public void increment() {
        count++;
    }

    public int getCount() {
        return count;
    }
}

@Test
void shouldDemonstrateRaceCondition() throws InterruptedException {
    SharedCounter counter = new SharedCounter();
    int threadCount = 10;
    int incrementsPerThread = 1000;

    // Создаём и запускаем потоки
    List<Thread> threads = new ArrayList<>();
    for (int i = 0; i < threadCount; i++) {
        Thread t = new Thread(() -> {
            for (int j = 0; j < incrementsPerThread; j++) {
                counter.increment();
            }
        });
        threads.add(t);
        t.start();
    }

    // Ждём завершения всех потоков
    for (Thread t : threads) {
        t.join();
    }

    // Без synchronized результат скорее всего < 10000
    System.out.println("Результат: " + counter.getCount());
    // Попробуйте добавить synchronized к increment() и сравните
}
```

### Задание 2 (среднее): ThreadLocal WebDriver Manager

Реализуйте класс `DriverManager` с `ThreadLocal<WebDriver>`. Создайте метод `getDriver()`, который создаёт WebDriver при первом обращении из потока (lazy initialization). Добавьте `removeDriver()` для очистки. Напишите базовый тестовый класс, использующий этот менеджер.

### Задание 3 (продвинутое): Потокобезопасный тестовый фреймворк

Создайте мини-фреймворк для параллельного запуска API-тестов:
- `ThreadLocal` для хранения auth-токена
- `ConcurrentHashMap` для сбора результатов
- `AtomicInteger` для подсчёта пройденных/упавших тестов
- `ExecutorService` для запуска тестов в пуле из N потоков
- Метод для вывода итогового отчёта

---

## Дополнительные ресурсы

- [Baeldung — Java Concurrency](https://www.baeldung.com/java-concurrency) — полный гайд по многопоточности
- [Baeldung — ThreadLocal](https://www.baeldung.com/java-threadlocal) — подробно о ThreadLocal
- [Baeldung — ConcurrentHashMap](https://www.baeldung.com/java-concurrent-map) — потокобезопасные коллекции
- [JUnit 5 — Parallel Execution](https://junit.org/junit5/docs/current/user-guide/#writing-tests-parallel-execution) — параллельные тесты в JUnit 5
- [Baeldung — ExecutorService](https://www.baeldung.com/java-executor-service-tutorial) — управление пулом потоков
- [Selenium — Thread-safe WebDriver](https://www.selenium.dev/documentation/webdriver/drivers/options/) — документация Selenium
- [Baeldung — Java synchronized](https://www.baeldung.com/java-synchronized) — синхронизация в Java
