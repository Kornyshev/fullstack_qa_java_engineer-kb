# Параллельный запуск тестов

## Введение

Последовательный запуск 200 UI-тестов может занимать 2-3 часа. Параллельный запуск
сокращает это время в несколько раз — пропорционально количеству потоков.
Однако параллельность вносит сложности: потокобезопасность, изоляция данных,
управление ресурсами. В этом руководстве разберём настройку параллельного запуска
для JUnit 5 и TestNG, обеспечение потокобезопасности и использование Selenoid
для масштабирования.

> **Цель:** Настроить параллельный запуск тестов, обеспечить потокобезопасность
> WebDriver через ThreadLocal, подключить Selenoid для параллельных браузеров
> и измерить прирост производительности.

---

## Часть 1: JUnit 5 — параллельное выполнение

### Шаг 1: Файл конфигурации junit-platform.properties

Файл: `src/test/resources/junit-platform.properties`

```properties
# Включаем параллельное выполнение тестов
junit.jupiter.execution.parallel.enabled=true

# Стратегия по умолчанию для классов — CONCURRENT (параллельно)
junit.jupiter.execution.parallel.mode.default=concurrent

# Стратегия для классов верхнего уровня — тоже параллельно
junit.jupiter.execution.parallel.mode.classes.default=concurrent

# Стратегия конфигурации количества потоков
junit.jupiter.execution.parallel.config.strategy=fixed

# Количество параллельных потоков
junit.jupiter.execution.parallel.config.fixed.parallelism=4

# Альтернатива: динамическое количество потоков (множитель к количеству ядер CPU)
# junit.jupiter.execution.parallel.config.strategy=dynamic
# junit.jupiter.execution.parallel.config.dynamic.factor=1.5
```

### Шаг 2: Аннотации для управления параллельностью

```java
import org.junit.jupiter.api.parallel.Execution;
import org.junit.jupiter.api.parallel.ExecutionMode;
import org.junit.jupiter.api.parallel.ResourceLock;

// Все тесты в классе выполняются параллельно
@Execution(ExecutionMode.CONCURRENT)
public class ParallelLoginTest extends BaseTest {

    @Test
    void testLoginWithValidCredentials() {
        // Этот тест запустится параллельно с другими
        loginPage.loginAs("user1@test.com", "password");
        assertTrue(homePage.isUserLoggedIn("user1"));
    }

    @Test
    void testLoginWithInvalidPassword() {
        loginPage.loginAs("user1@test.com", "wrongpassword");
        loginPage.verifyErrorMessage("email or password is invalid");
    }
}

// Тесты в этом классе выполняются последовательно
// (например, если они зависят от общего состояния)
@Execution(ExecutionMode.SAME_THREAD)
public class SequentialArticleTest extends BaseTest {

    @Test
    @Order(1)
    void testCreateArticle() {
        // Создаём статью
    }

    @Test
    @Order(2)
    void testEditArticle() {
        // Редактируем созданную статью — зависит от предыдущего теста
    }
}

// ResourceLock — для тестов, которые работают с общим ресурсом
public class SharedResourceTest extends BaseTest {

    // Эти тесты не будут выполняться одновременно,
    // потому что оба блокируют ресурс "ADMIN_USER"
    @Test
    @ResourceLock("ADMIN_USER")
    void testAdminAction1() {
        // Использует единственного admin-пользователя
    }

    @Test
    @ResourceLock("ADMIN_USER")
    void testAdminAction2() {
        // Тоже использует admin-пользователя — дождётся завершения первого теста
    }
}
```

---

## Часть 2: TestNG — параллельное выполнение

### Файл: testng.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE suite SYSTEM "https://testng.org/testng-1.0.dtd">

<!--
    parallel="methods"   — параллельно выполняются тестовые методы
    parallel="classes"   — параллельно выполняются тестовые классы
    parallel="tests"     — параллельно выполняются теги <test>
    parallel="instances" — параллельно выполняются экземпляры классов

    thread-count — количество параллельных потоков
-->
<suite name="Regression Suite" parallel="classes" thread-count="4">

    <!-- Smoke-тесты — быстрый набор -->
    <test name="Smoke Tests" parallel="methods" thread-count="2">
        <classes>
            <class name="tests.LoginTest"/>
            <class name="tests.HomePageTest"/>
        </classes>
    </test>

    <!-- API-тесты — можно больше параллельности, не нужен браузер -->
    <test name="API Tests" parallel="methods" thread-count="8">
        <packages>
            <package name="tests.api"/>
        </packages>
    </test>

    <!-- UI-тесты — ограничиваем параллельность из-за нагрузки на браузеры -->
    <test name="UI Tests" parallel="classes" thread-count="4">
        <packages>
            <package name="tests.ui"/>
        </packages>
    </test>

</suite>
```

### Запуск TestNG с параллельностью через Maven

```xml
<!-- pom.xml — настройка maven-surefire-plugin для TestNG -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.5</version>
    <configuration>
        <suiteXmlFiles>
            <suiteXmlFile>testng.xml</suiteXmlFile>
        </suiteXmlFiles>
        <!-- Альтернатива testng.xml — параметры через CLI -->
        <properties>
            <property>
                <name>parallel</name>
                <value>methods</value>
            </property>
            <property>
                <name>threadCount</name>
                <value>4</value>
            </property>
        </properties>
    </configuration>
</plugin>
```

---

## Часть 3: ThreadLocal — потокобезопасный WebDriver

### Проблема

При параллельном запуске каждый поток должен иметь свой экземпляр WebDriver.
Если все тесты используют одну переменную `driver`, они будут конфликтовать:
один тест открывает страницу, а другой в этот момент кликает не туда.

### Решение: ThreadLocal

```java
public class DriverManager {

    // Каждый поток получит свой экземпляр WebDriver
    private static final ThreadLocal<WebDriver> driverThreadLocal = new ThreadLocal<>();

    // Создание драйвера для текущего потока
    public static void initDriver() {
        if (driverThreadLocal.get() != null) {
            // Драйвер уже существует для этого потока — не создаём повторно
            return;
        }

        WebDriver driver;
        String browser = ConfigReader.getBrowser();

        switch (browser.toLowerCase()) {
            case "chrome":
                ChromeOptions chromeOptions = new ChromeOptions();
                if (ConfigReader.isHeadless()) {
                    chromeOptions.addArguments("--headless=new");
                }
                // Каждый поток получает отдельный экземпляр Chrome
                driver = new ChromeDriver(chromeOptions);
                break;

            case "firefox":
                FirefoxOptions firefoxOptions = new FirefoxOptions();
                if (ConfigReader.isHeadless()) {
                    firefoxOptions.addArguments("--headless");
                }
                driver = new FirefoxDriver(firefoxOptions);
                break;

            default:
                throw new IllegalArgumentException(
                    "Неподдерживаемый браузер: " + browser);
        }

        driver.manage().window().maximize();
        driver.manage().timeouts().implicitlyWait(
            Duration.ofSeconds(ConfigReader.getImplicitTimeout()));

        // Сохраняем драйвер в ThreadLocal текущего потока
        driverThreadLocal.set(driver);
    }

    // Получение драйвера текущего потока
    public static WebDriver getDriver() {
        WebDriver driver = driverThreadLocal.get();
        if (driver == null) {
            throw new IllegalStateException(
                "WebDriver не инициализирован для потока: "
                + Thread.currentThread().getName());
        }
        return driver;
    }

    // Закрытие драйвера и очистка ThreadLocal
    public static void quitDriver() {
        WebDriver driver = driverThreadLocal.get();
        if (driver != null) {
            driver.quit();
            // Обязательно удаляем ссылку — иначе утечка памяти
            driverThreadLocal.remove();
        }
    }
}
```

### BaseTest с ThreadLocal

```java
public abstract class BaseTest {

    protected LoginPage loginPage;
    protected ArticlePage articlePage;
    protected EditorPage editorPage;

    @BeforeEach
    void setUp() {
        // Каждый тест в своём потоке создаёт свой браузер
        DriverManager.initDriver();

        // Page Objects используют драйвер текущего потока
        WebDriver driver = DriverManager.getDriver();
        loginPage = new LoginPage(driver);
        articlePage = new ArticlePage(driver);
        editorPage = new EditorPage(driver);
    }

    @AfterEach
    void tearDown() {
        // Закрываем браузер текущего потока
        DriverManager.quitDriver();
    }

    // Для доступа из Extension (скриншоты при падении)
    public WebDriver getDriver() {
        return DriverManager.getDriver();
    }
}
```

---

## Часть 4: Потокобезопасность тестовых данных

### Проблема: конфликт данных

```java
// АНТИПАТТЕРН: общие данные для параллельных тестов
public class ArticleTest extends BaseTest {

    // Один и тот же email — при параллельном запуске оба теста
    // попытаются зарегистрировать одного пользователя
    private static final String EMAIL = "test@example.com";

    @Test
    void testCreateArticle() {
        // Оба теста используют EMAIL — конфликт!
        loginPage.loginAs(EMAIL, "password");
        // ...
    }

    @Test
    void testDeleteArticle() {
        loginPage.loginAs(EMAIL, "password");
        // ...
    }
}
```

### Решение: уникальные данные для каждого теста

```java
public class ArticleTest extends BaseTest {

    @Test
    void testCreateArticle() {
        // Каждый тест генерирует уникальные данные
        String email = DataGenerator.uniqueEmail();
        String password = DataGenerator.strongPassword();
        String username = DataGenerator.uniqueUsername();

        // Создаём пользователя через API — быстро и изолированно
        ApiClient.createUser(email, password, username);

        loginPage.loginAs(email, password);
        editorPage.open();
        // Заголовок тоже уникальный
        String title = "Article " + System.nanoTime();
        editorPage.createArticle(title, "desc", "body", "tag");

        assertEquals(title, articlePage.getTitle());
    }

    @Test
    void testDeleteArticle() {
        // Полностью независимые данные — никаких конфликтов
        String email = DataGenerator.uniqueEmail();
        String password = DataGenerator.strongPassword();
        String username = DataGenerator.uniqueUsername();

        String token = ApiClient.createUserAndGetToken(email, password, username);
        String slug = ApiClient.createArticle(token, "Delete Me " + System.nanoTime(),
                                              "desc", "body");

        loginPage.loginAs(email, password);
        articlePage.open(slug);
        articlePage.deleteArticle();

        articlePage.verifyArticleDeleted();
    }
}
```

### ThreadLocal для вспомогательных данных

```java
public class TestContext {

    // Контекст теста — хранит данные текущего тестового потока
    private static final ThreadLocal<Map<String, Object>> context =
        ThreadLocal.withInitial(HashMap::new);

    public static void set(String key, Object value) {
        context.get().put(key, value);
    }

    @SuppressWarnings("unchecked")
    public static <T> T get(String key) {
        return (T) context.get().get(key);
    }

    // Обязательно вызывать в @AfterEach для предотвращения утечек памяти
    public static void clear() {
        context.get().clear();
        context.remove();
    }
}

// Использование в тесте
@BeforeEach
void setUp() {
    DriverManager.initDriver();
    // Сохраняем токен в контексте потока
    String token = ApiClient.createUserAndGetToken(
        DataGenerator.uniqueEmail(), "password", DataGenerator.uniqueUsername());
    TestContext.set("token", token);
}

@AfterEach
void tearDown() {
    DriverManager.quitDriver();
    TestContext.clear();
}
```

---

## Часть 5: Selenoid — параллельные браузеры в Docker

### Что такое Selenoid

Selenoid — это сервер для запуска браузеров в Docker-контейнерах. Каждый тест получает
чистый браузер в изолированном контейнере. Преимущества:
- Неограниченная параллельность (ограничена только ресурсами сервера)
- Чистый браузер для каждого теста (нет кэша, cookies, расширений)
- Поддержка разных версий браузеров одновременно
- Запись видео каждого теста

### Установка Selenoid

```bash
# Скачиваем Configuration Manager
curl -s https://aerokube.com/cm/linux_amd64 > cm && chmod +x cm

# Запускаем Selenoid с последними версиями Chrome и Firefox
./cm selenoid start --vnc --browsers "chrome:latest;firefox:latest"

# Запускаем Selenoid UI — веб-интерфейс для наблюдения за тестами
./cm selenoid-ui start

# Selenoid доступен на http://localhost:4444
# Selenoid UI доступен на http://localhost:8080
```

### Файл: browsers.json (конфигурация Selenoid)

```json
{
  "chrome": {
    "default": "125.0",
    "versions": {
      "125.0": {
        "image": "selenoid/vnc:chrome_125.0",
        "port": "4444",
        "path": "/"
      },
      "124.0": {
        "image": "selenoid/vnc:chrome_124.0",
        "port": "4444",
        "path": "/"
      }
    }
  },
  "firefox": {
    "default": "126.0",
    "versions": {
      "126.0": {
        "image": "selenoid/vnc:firefox_126.0",
        "port": "4444",
        "path": "/"
      }
    }
  }
}
```

### Код: WebDriver для Selenoid

```java
public class DriverManager {

    private static final ThreadLocal<WebDriver> driverThreadLocal = new ThreadLocal<>();

    public static void initDriver() {
        WebDriver driver;
        String runMode = ConfigReader.getRunMode(); // "local" или "remote"

        if ("remote".equals(runMode)) {
            driver = createRemoteDriver();
        } else {
            driver = createLocalDriver();
        }

        driver.manage().window().maximize();
        driverThreadLocal.set(driver);
    }

    // Создание RemoteWebDriver для Selenoid
    private static WebDriver createRemoteDriver() {
        String selenoidUrl = ConfigReader.getSelenoidUrl(); // http://localhost:4444/wd/hub
        String browser = ConfigReader.getBrowser();

        ChromeOptions options = new ChromeOptions();
        options.setCapability("browserVersion", "125.0");

        // Selenoid-специфичные capabilities
        Map<String, Object> selenoidOptions = new HashMap<>();
        selenoidOptions.put("enableVNC", true);     // Включить VNC для просмотра экрана
        selenoidOptions.put("enableVideo", true);    // Записывать видео теста
        selenoidOptions.put("enableLog", true);      // Логировать действия
        selenoidOptions.put("videoName", Thread.currentThread().getName() + ".mp4");
        // Имя сессии — для идентификации в Selenoid UI
        selenoidOptions.put("name", "Test: " + Thread.currentThread().getName());
        // Таймаут бездействия — если тест зависнет, контейнер будет остановлен
        selenoidOptions.put("sessionTimeout", "5m");

        options.setCapability("selenoid:options", selenoidOptions);

        try {
            return new RemoteWebDriver(new URL(selenoidUrl), options);
        } catch (MalformedURLException e) {
            throw new RuntimeException(
                "Некорректный URL Selenoid: " + selenoidUrl, e);
        }
    }

    private static WebDriver createLocalDriver() {
        ChromeOptions options = new ChromeOptions();
        if (ConfigReader.isHeadless()) {
            options.addArguments("--headless=new");
        }
        return new ChromeDriver(options);
    }
}
```

---

## Часть 6: Замер производительности — до и после

### Скрипт для замера времени

```bash
#!/bin/bash
# Файл: measure_performance.sh
# Скрипт для сравнения последовательного и параллельного запуска

echo "=== Замер последовательного запуска ==="
SEQUENTIAL_START=$(date +%s)
mvn clean test \
  -Djunit.jupiter.execution.parallel.enabled=false \
  -Drun.mode=remote \
  2>&1 | tail -5
SEQUENTIAL_END=$(date +%s)
SEQUENTIAL_TIME=$((SEQUENTIAL_END - SEQUENTIAL_START))
echo "Последовательный запуск: ${SEQUENTIAL_TIME} секунд"

echo ""
echo "=== Замер параллельного запуска (4 потока) ==="
PARALLEL_START=$(date +%s)
mvn clean test \
  -Djunit.jupiter.execution.parallel.enabled=true \
  -Djunit.jupiter.execution.parallel.config.fixed.parallelism=4 \
  -Drun.mode=remote \
  2>&1 | tail -5
PARALLEL_END=$(date +%s)
PARALLEL_TIME=$((PARALLEL_END - PARALLEL_START))
echo "Параллельный запуск (4 потока): ${PARALLEL_TIME} секунд"

echo ""
echo "=== Замер параллельного запуска (8 потоков) ==="
PARALLEL8_START=$(date +%s)
mvn clean test \
  -Djunit.jupiter.execution.parallel.enabled=true \
  -Djunit.jupiter.execution.parallel.config.fixed.parallelism=8 \
  -Drun.mode=remote \
  2>&1 | tail -5
PARALLEL8_END=$(date +%s)
PARALLEL8_TIME=$((PARALLEL8_END - PARALLEL8_START))
echo "Параллельный запуск (8 потоков): ${PARALLEL8_TIME} секунд"

echo ""
echo "=== РЕЗУЛЬТАТЫ ==="
echo "Последовательно:   ${SEQUENTIAL_TIME}с"
echo "4 потока:          ${PARALLEL_TIME}с (ускорение x$(echo "scale=1; $SEQUENTIAL_TIME/$PARALLEL_TIME" | bc))"
echo "8 потоков:          ${PARALLEL8_TIME}с (ускорение x$(echo "scale=1; $SEQUENTIAL_TIME/$PARALLEL8_TIME" | bc))"
```

### Типичные результаты

| Конфигурация | Тестов | Время | Ускорение |
|-------------|--------|-------|-----------|
| Последовательно | 50 | 25 мин | 1x |
| 4 потока (local Chrome) | 50 | 8 мин | 3.1x |
| 4 потока (Selenoid) | 50 | 7 мин | 3.6x |
| 8 потоков (Selenoid) | 50 | 4 мин | 6.3x |
| 16 потоков (Selenoid) | 50 | 3 мин | 8.3x |

> **Примечание:** Ускорение не линейное. Узкие места — CPU, память, сеть, сам SUT.
> Оптимальное количество потоков обычно равно количеству ядер CPU * 1.5-2.

---

## Часть 7: Типичные проблемы и решения

### Проблема 1: Flaky-тесты при параллельном запуске

```java
// АНТИПАТТЕРН: тесты зависят от порядка выполнения
public class ArticleTest {
    static String createdSlug; // Общая переменная — гонка данных!

    @Test
    void testCreate() {
        createdSlug = createArticle(); // Поток 1 записывает
    }

    @Test
    void testRead() {
        openArticle(createdSlug); // Поток 2 читает — но значение может быть null!
    }
}

// РЕШЕНИЕ: каждый тест создаёт свои данные
public class ArticleTest {
    @Test
    void testCreate() {
        String slug = createArticle();
        verifyArticleExists(slug);
    }

    @Test
    void testRead() {
        // Предусловие: создаём статью через API
        String slug = ApiClient.createArticle(token, "Title", "desc", "body");
        openArticle(slug);
        verifyArticleDisplayed();
    }
}
```

### Проблема 2: Утечка WebDriver

```java
// АНТИПАТТЕРН: забыли вызвать remove()
@AfterEach
void tearDown() {
    DriverManager.getDriver().quit();
    // ThreadLocal не очищен — утечка памяти при длительном прогоне!
}

// РЕШЕНИЕ: всегда вызывать remove()
@AfterEach
void tearDown() {
    DriverManager.quitDriver(); // Внутри: driver.quit() + threadLocal.remove()
}
```

### Проблема 3: Исчерпание ресурсов

```properties
# Ограничиваем количество потоков в зависимости от ресурсов
# Для CI-сервера с 4 CPU и 8GB RAM:
junit.jupiter.execution.parallel.config.fixed.parallelism=4

# Для Selenoid — ограничение в docker-compose
# selenoid:
#   environment:
#     - OVERRIDE_VIDEO_OUTPUT_DIR=/opt/selenoid/video/
#   deploy:
#     resources:
#       limits:
#         memory: 8G
```

---

## Практическое задание

### Задание 1: Настройка параллельного запуска (JUnit 5)

1. Создайте `junit-platform.properties` с `parallelism=4`
2. Убедитесь, что все тесты используют ThreadLocal для WebDriver
3. Запустите тесты и проверьте, что они выполняются параллельно
4. Проверьте Timeline в Allure-отчёте — тесты должны перекрываться по времени

### Задание 2: Подключение Selenoid

1. Установите Selenoid через Configuration Manager
2. Модифицируйте DriverManager для поддержки RemoteWebDriver
3. Запустите тесты с `-Drun.mode=remote`
4. Откройте Selenoid UI и наблюдайте за параллельным выполнением

### Задание 3: Замер производительности

1. Выполните скрипт `measure_performance.sh`
2. Зафиксируйте время для последовательного и параллельного запуска
3. Постройте таблицу сравнения (количество потоков vs время)
4. Определите оптимальное количество потоков для вашего оборудования

**Критерии оценки:**
- WebDriver потокобезопасен (ThreadLocal + remove)
- Тестовые данные изолированы между потоками
- Параллельный запуск работает стабильно (3 последовательных прогона без flaky)
- Замер показывает реальное ускорение в 2-4 раза

---

## Чек-лист самопроверки

- [ ] junit-platform.properties настроен для параллельного запуска
- [ ] WebDriver обёрнут в ThreadLocal
- [ ] ThreadLocal.remove() вызывается в @AfterEach
- [ ] Каждый тест генерирует уникальные тестовые данные
- [ ] Нет общих mutable-переменных между тестами
- [ ] Selenoid установлен и запущен
- [ ] RemoteWebDriver настроен с Selenoid capabilities
- [ ] Замер производительности проведён и задокументирован
- [ ] Timeline в Allure показывает параллельное выполнение
- [ ] 3 подряд запуска прошли без flaky-тестов
