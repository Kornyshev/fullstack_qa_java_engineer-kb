# Параллельный запуск тестов

## Обзор

По мере роста тестового набора последовательный запуск становится узким местом: 500 UI-тестов
по 10 секунд каждый — это почти полтора часа. Параллельный запуск позволяет сократить это время
в несколько раз, но требует тщательной подготовки: тесты должны быть независимыми, данные —
изолированными, а ресурсы (WebDriver, БД-соединения) — потокобезопасными. В этом разделе
разберём все подходы к параллельному запуску: JUnit 5, TestNG, Maven Surefire, а также решение
типичных проблем thread safety.

---

## Зачем запускать параллельно

| Последовательно | Параллельно (4 потока) |
|----------------|----------------------|
| 500 тестов × 10 сек = **83 мин** | 500 тестов / 4 × 10 сек = **~21 мин** |
| 1000 тестов × 10 сек = **167 мин** | 1000 тестов / 4 × 10 сек = **~42 мин** |

Быстрая обратная связь — ключевой принцип CI/CD. Если тесты идут 3 часа, разработчики не ждут
результатов и мержат без зелёных тестов.

---

## JUnit 5: параллельный запуск

### Конфигурация через junit-platform.properties

Файл `src/test/resources/junit-platform.properties`:

```properties
# Включение параллельного выполнения
junit.jupiter.execution.parallel.enabled=true

# Режим по умолчанию для тестовых классов
# CONCURRENT — параллельно, SAME_THREAD — последовательно
junit.jupiter.execution.parallel.mode.default=concurrent

# Режим для классов верхнего уровня
junit.jupiter.execution.parallel.mode.classes.default=concurrent

# Стратегия определения количества потоков
# fixed — фиксированное число, dynamic — на основе доступных ядер
junit.jupiter.execution.parallel.config.strategy=fixed

# Количество потоков при fixed-стратегии
junit.jupiter.execution.parallel.config.fixed.parallelism=4

# Или динамическая стратегия: factor × количество ядер
# junit.jupiter.execution.parallel.config.strategy=dynamic
# junit.jupiter.execution.parallel.config.dynamic.factor=1
```

### Аннотация @Execution

Позволяет управлять параллельностью на уровне класса или метода:

```java
import org.junit.jupiter.api.parallel.Execution;
import org.junit.jupiter.api.parallel.ExecutionMode;

// Все тесты в этом классе выполняются параллельно
@Execution(ExecutionMode.CONCURRENT)
public class CatalogTest extends BaseUiTest {

    @Test
    void searchReturnsResults() { /* ... */ }

    @Test
    void filterByPriceWorks() { /* ... */ }

    @Test
    void sortByNameWorks() { /* ... */ }
}

// Тесты в этом классе — строго последовательно
// (например, зависят от общего состояния)
@Execution(ExecutionMode.SAME_THREAD)
public class OrderFlowTest extends BaseUiTest {

    @Test
    void step1_addToCart() { /* ... */ }

    @Test
    void step2_checkout() { /* ... */ }
}
```

### @ResourceLock — защита общих ресурсов

```java
import org.junit.jupiter.api.parallel.ResourceLock;
import org.junit.jupiter.api.parallel.ResourceAccessMode;

public class UserTest {

    // Эти тесты не будут запускаться одновременно,
    // потому что используют один и тот же ресурс с WRITE-доступом
    @Test
    @ResourceLock(value = "ADMIN_USER", mode = ResourceAccessMode.READ_WRITE)
    void adminCanDeleteUser() { /* ... */ }

    @Test
    @ResourceLock(value = "ADMIN_USER", mode = ResourceAccessMode.READ_WRITE)
    void adminCanBlockUser() { /* ... */ }

    // Этот тест может работать параллельно с любыми другими
    @Test
    void regularUserCanViewProfile() { /* ... */ }
}
```

---

## TestNG: параллельный запуск

### Конфигурация через testng.xml

```xml
<!DOCTYPE suite SYSTEM "https://testng.org/testng-1.0.dtd">
<suite name="Test Suite" parallel="methods" thread-count="4">

    <!-- parallel="methods" — каждый тестовый метод в отдельном потоке -->
    <!-- parallel="classes" — каждый тестовый класс в отдельном потоке -->
    <!-- parallel="tests" — каждый блок <test> в отдельном потоке -->

    <test name="UI Tests">
        <packages>
            <package name="com.company.project.tests.ui"/>
        </packages>
    </test>

    <test name="API Tests">
        <packages>
            <package name="com.company.project.tests.api"/>
        </packages>
    </test>
</suite>
```

### Уровни параллелизма TestNG

| parallel= | Поведение |
|-----------|-----------|
| `methods` | Каждый `@Test` метод — в своём потоке. Максимальный параллелизм. |
| `classes` | Методы одного класса — последовательно, классы — параллельно. |
| `tests` | Каждый `<test>` блок — в своём потоке. |
| `instances` | Экземпляры классов — параллельно. |

### Аннотации TestNG для потоков

```java
// Указание thread pool для конкретного теста
@Test(threadPoolSize = 3, invocationCount = 10, timeOut = 10000)
public void loadTestLogin() {
    // Метод выполнится 10 раз в 3 потоках
    apiClient.login("user", "pass");
}
```

---

## Maven Surefire Plugin: параллельный запуск через fork

### Конфигурация pom.xml

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.5</version>
    <configuration>
        <!-- Количество JVM-процессов для параллельного запуска -->
        <forkCount>4</forkCount>

        <!-- true — повторное использование JVM-процессов (экономия памяти) -->
        <!-- false — новая JVM для каждого тестового класса (полная изоляция) -->
        <reuseForks>true</reuseForks>

        <!-- Передача системных свойств в forked JVM -->
        <systemPropertyVariables>
            <env>${env}</env>
            <browser>${browser}</browser>
        </systemPropertyVariables>

        <!-- Включение конкретных тестов -->
        <includes>
            <include>**/*Test.java</include>
        </includes>
    </configuration>
</plugin>
```

### Разница между fork и thread

| Аспект | Fork (Surefire) | Thread (JUnit 5 / TestNG) |
|--------|----------------|--------------------------|
| Изоляция | Полная — отдельная JVM | Общая JVM, разные потоки |
| Память | Больше (каждый fork — отдельная JVM) | Меньше |
| Static-поля | Изолированы между fork-ами | Общие для всех потоков |
| Скорость старта | Медленнее (запуск JVM) | Быстрее |
| Когда использовать | Когда тесты не thread-safe | Когда тесты thread-safe |

### Комбинирование подходов

```xml
<!-- 2 JVM-процесса, в каждом — JUnit 5 параллелизм с 4 потоками -->
<!-- Итого: 8 тестов одновременно -->
<configuration>
    <forkCount>2</forkCount>
    <reuseForks>true</reuseForks>
</configuration>
```

Плюс `junit-platform.properties` с `parallelism=4`.

---

## Проблемы thread safety и их решение

### Проблема 1: Общий WebDriver

```java
// ПЛОХО — один WebDriver на все потоки
public class BaseUiTest {
    protected static WebDriver driver; // static = shared!

    @BeforeAll
    static void setup() {
        driver = new ChromeDriver(); // Все потоки используют один браузер!
    }
}
```

**Последствие:** потоки одновременно кликают в одном окне браузера — хаос.

### Решение: ThreadLocal

```java
// ХОРОШО — каждый поток получает свой WebDriver
public class DriverManager {

    // ThreadLocal хранит отдельное значение для каждого потока
    private static final ThreadLocal<WebDriver> driverThreadLocal = new ThreadLocal<>();

    public static WebDriver getDriver() {
        return driverThreadLocal.get();
    }

    public static void setDriver(WebDriver driver) {
        driverThreadLocal.set(driver);
    }

    public static void closeDriver() {
        WebDriver driver = driverThreadLocal.get();
        if (driver != null) {
            driver.quit();
            driverThreadLocal.remove(); // Предотвращаем утечку памяти
        }
    }
}
```

```java
public class BaseUiTest {

    @BeforeEach
    void setup() {
        WebDriver driver = WebDriverFactory.createDriver(CONFIG.browser());
        DriverManager.setDriver(driver);
    }

    @AfterEach
    void teardown() {
        DriverManager.closeDriver();
    }
}
```

> **Примечание:** Selenide уже использует ThreadLocal для WebDriver внутри,
> поэтому с Selenide параллельный запуск работает «из коробки».

### Проблема 2: Общие тестовые данные

```java
// ПЛОХО — все потоки используют одного пользователя
private static final String TEST_USER = "testuser@mail.com";

@Test
void userCanUpdateProfile() {
    // Поток 1 обновляет имя на "Иван"
    // Поток 2 обновляет имя на "Пётр"
    // Кто выиграет? Непредсказуемо!
    apiClient.updateUser(TEST_USER, newName);
}
```

**Решение:** каждый тест создаёт уникальные данные:

```java
@Test
void userCanUpdateProfile() {
    // Каждый поток создаёт своего уникального пользователя
    User user = apiClient.createUser(UserFactory.randomUser());
    apiClient.updateUser(user.getId(), Map.of("firstName", "Иван"));
    // Проверяем — никаких конфликтов
    User updated = apiClient.getUserById(user.getId());
    assertThat(updated.getFirstName()).isEqualTo("Иван");
}
```

### Проблема 3: Общий конфиг (mutable state)

```java
// ПЛОХО — мутабельный Singleton
public class Config {
    private static String baseUrl = "https://dev.myshop.com";

    public static void setBaseUrl(String url) {
        baseUrl = url; // Поток 1 ставит "dev", поток 2 — "staging"
    }
}
```

**Решение:** конфигурация должна быть immutable (неизменяемой) после инициализации.

### Проблема 4: Порты и файлы

```java
// ПЛОХО — все потоки пишут в один файл
FileWriter writer = new FileWriter("test-results.log");
writer.write(testResult); // Гонка данных!
```

**Решение:** уникальные файлы для каждого потока или потокобезопасные структуры:

```java
// Потокобезопасный лог
private static final ConcurrentLinkedQueue<String> results = new ConcurrentLinkedQueue<>();

results.add(Thread.currentThread().getName() + ": " + testResult);
```

---

## Selenoid для параллельных браузерных тестов

Selenoid — это легковесная замена Selenium Grid, запускающая браузеры в Docker-контейнерах.

### docker-compose.yml

```yaml
version: '3'
services:
  selenoid:
    image: aerokube/selenoid:latest
    ports:
      - "4444:4444"
    volumes:
      - ./config/browsers.json:/etc/selenoid/browsers.json
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - OVERRIDE_VIDEO_OUTPUT_DIR=/opt/selenoid/video

  selenoid-ui:
    image: aerokube/selenoid-ui:latest
    ports:
      - "8080:8080"
    depends_on:
      - selenoid
    command: ["--selenoid-uri", "http://selenoid:4444"]
```

### browsers.json

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

### Настройка Selenide для работы с Selenoid

```java
@BeforeEach
void setup() {
    if (CONFIG.isRemote()) {
        Configuration.remote = CONFIG.selenoidUrl(); // http://selenoid:4444/wd/hub
        DesiredCapabilities capabilities = new DesiredCapabilities();
        capabilities.setCapability("selenoid:options", Map.of(
            "enableVNC", true,       // Доступ к экрану через VNC
            "enableVideo", true,     // Запись видео теста
            "enableLog", true        // Сохранение логов браузера
        ));
        Configuration.browserCapabilities = capabilities;
    }
    Configuration.browser = CONFIG.browser();
    Configuration.browserSize = CONFIG.browserSize();
}
```

### Преимущества Selenoid для параллельного запуска

- **Изоляция** — каждый тест работает в своём Docker-контейнере.
- **Масштабирование** — можно запустить десятки браузеров одновременно.
- **Чистота** — контейнер уничтожается после теста, не остаётся «мусора».
- **VNC** — можно подключиться к браузеру и наблюдать за тестом.
- **Видео** — автоматическая запись для отладки.

---

## Полная конфигурация параллельного запуска

### JUnit 5 + Maven Surefire + Selenide + Selenoid

**pom.xml:**

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.5</version>
    <configuration>
        <forkCount>2</forkCount>
        <reuseForks>true</reuseForks>
        <systemPropertyVariables>
            <env>${env}</env>
            <selenoid.url>http://localhost:4444/wd/hub</selenoid.url>
        </systemPropertyVariables>
    </configuration>
</plugin>
```

**junit-platform.properties:**

```properties
junit.jupiter.execution.parallel.enabled=true
junit.jupiter.execution.parallel.mode.default=concurrent
junit.jupiter.execution.parallel.mode.classes.default=concurrent
junit.jupiter.execution.parallel.config.strategy=fixed
junit.jupiter.execution.parallel.config.fixed.parallelism=4
```

**Запуск:**

```bash
# Запуск с 4 потоками в 2 fork-ах = 8 параллельных тестов
mvn test -Denv=staging -Dfork.count=2

# Только smoke-тесты параллельно
mvn test -Dgroups=smoke
```

---

## Чек-лист готовности к параллельному запуску

Перед включением параллелизма убедитесь:

- [ ] Каждый тест **независим** — не зависит от порядка выполнения и результата других тестов.
- [ ] Каждый тест создаёт **уникальные данные** — нет общих пользователей, заказов.
- [ ] WebDriver в **ThreadLocal** или используется Selenide (ThreadLocal встроен).
- [ ] Конфигурация **immutable** после инициализации.
- [ ] Нет `static mutable` полей, используемых для передачи данных между тестами.
- [ ] Файловые операции **изолированы** (уникальные имена файлов).
- [ ] Cleanup **не удаляет чужие данные** (удаление по ID, не TRUNCATE).
- [ ] **Нет общих внешних ресурсов** (портов, файлов, email-ящиков).
- [ ] Тесты прошли **5 прогонов подряд** без flaky-падений.

---

## Связь с тестированием

Параллельный запуск критичен для:

- **CI/CD** — быстрая обратная связь по каждому коммиту.
- **Масштабирования** — переход от 100 к 1000 тестов без увеличения времени в 10 раз.
- **Release confidence** — полный набор тестов за 15 минут вместо 3 часов.
- **Выявления flaky-тестов** — параллелизм обнажает скрытые зависимости между тестами.

---

## Типичные ошибки

1. **Включить параллелизм без подготовки** — тесты начинают случайно падать.
2. **Shared WebDriver** — все потоки кликают в одном браузере.
3. **Общие тестовые данные** — потоки перезаписывают данные друг друга.
4. **`Thread.sleep()` вместо ожиданий** — в параллельном режиме работает ещё хуже.
5. **TRUNCATE TABLE в setup** — один тест удаляет данные другого.
6. **Зависимость от порядка выполнения** — `@Order(1)`, `@Order(2)` в параллельных тестах.
7. **Забыть `ThreadLocal.remove()`** — утечка памяти в thread pool.
8. **Слишком много потоков** — система не справляется, тесты медленнее, чем последовательно.

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Зачем запускать тесты параллельно?
2. Как включить параллельный запуск в JUnit 5?
3. Что такое `forkCount` в Maven Surefire?
4. Какая разница между fork и thread в контексте параллельного запуска?
5. Что такое ThreadLocal и зачем он нужен?

### 🟡 Средний уровень
6. Как обеспечить thread safety для WebDriver при параллельных тестах?
7. Как изолировать тестовые данные при параллельном запуске?
8. Какая разница между `parallel="methods"` и `parallel="classes"` в TestNG?
9. Что такое Selenoid и как он помогает с параллельным запуском?
10. Как определить оптимальное количество потоков для параллельного запуска?

### 🔴 Продвинутый уровень
11. Как отлаживать flaky-тесты, возникающие только при параллельном запуске?
12. Как организовать параллельный запуск с общей базой данных?
13. Как совместить `@ResourceLock` в JUnit 5 с максимальным параллелизмом?
14. Какие подходы к шардингу тестов между CI/CD runner-ами вы знаете?
15. Как реализовать динамическое масштабирование Selenoid-кластера?

---

## Практические задания

### Задание 1: JUnit 5 параллелизм
Возьмите проект с 10+ тестами. Настройте параллельный запуск через `junit-platform.properties`.
Замерьте время последовательного и параллельного выполнения. Сравните.

### Задание 2: ThreadLocal WebDriver
Реализуйте `DriverManager` с ThreadLocal. Напишите 5 UI-тестов, которые стабильно
проходят параллельно в 5 потоках.

### Задание 3: Selenoid
Разверните Selenoid через Docker Compose. Настройте Selenide для работы с Selenoid.
Запустите 10 тестов параллельно, наблюдая за выполнением через Selenoid UI.

### Задание 4: Поиск race condition
Создайте тест, который стабильно проходит последовательно, но падает параллельно
(используйте shared mutable state). Затем исправьте его с помощью ThreadLocal и уникальных данных.

---

## Дополнительные ресурсы

- [JUnit 5 — Parallel Execution](https://junit.org/junit5/docs/current/user-guide/#writing-tests-parallel-execution)
- [TestNG — Parallel Tests](https://testng.org/doc/documentation-main.html#parallel-tests)
- [Maven Surefire — Fork Options](https://maven.apache.org/surefire/maven-surefire-plugin/examples/fork-options-and-parallel-execution.html)
- [Selenoid — Quick Start](https://aerokube.com/selenoid/latest/)
- [Baeldung — ThreadLocal in Java](https://www.baeldung.com/java-threadlocal)
