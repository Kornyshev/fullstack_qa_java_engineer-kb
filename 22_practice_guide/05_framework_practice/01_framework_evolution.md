# Эволюция тестового фреймворка

## Введение

Построение тестового фреймворка — это итеративный процесс. Нельзя сразу создать идеальную архитектуру.
Каждая стадия развития фреймворка решает конкретную проблему предыдущей стадии.
В этом руководстве мы пройдём путь от хаотичного скрипта до многослойного фреймворка промышленного уровня.

> **Цель:** Понять, почему фреймворк эволюционирует, какие проблемы решает каждая стадия,
> и уметь рефакторить существующий код поэтапно.

---

## Stage 1: Тесты без Page Object Model — всё в одном файле

### Проблема

Начинающий автоматизатор обычно пишет тесты «в лоб»: все локаторы, действия и проверки
находятся прямо в тестовом методе. Это работает для одного-двух тестов, но при росте
количества тестов превращается в кошмар сопровождения.

### Пример кода (антипаттерн)

```java
public class AllInOneTest {

    WebDriver driver;

    @BeforeEach
    void setUp() {
        // Инициализация драйвера прямо в тесте
        driver = new ChromeDriver();
        driver.manage().window().maximize();
        driver.manage().timeouts().implicitlyWait(Duration.ofSeconds(10));
    }

    @AfterEach
    void tearDown() {
        if (driver != null) {
            driver.quit();
        }
    }

    @Test
    void testLogin() {
        // URL захардкожен прямо в тесте
        driver.get("http://localhost:3000/login");

        // Локаторы дублируются в каждом тесте
        driver.findElement(By.cssSelector("input[placeholder='Email']"))
              .sendKeys("user@example.com");
        driver.findElement(By.cssSelector("input[placeholder='Password']"))
              .sendKeys("password123");
        driver.findElement(By.cssSelector("button[type='submit']"))
              .click();

        // Ожидание захардкожено
        try { Thread.sleep(2000); } catch (InterruptedException e) {}

        // Проверка
        String username = driver.findElement(By.cssSelector(".nav-link[href='/profile']"))
                                .getText();
        assertEquals("testuser", username);
    }

    @Test
    void testCreateArticle() {
        // Снова логин — полное дублирование кода
        driver.get("http://localhost:3000/login");
        driver.findElement(By.cssSelector("input[placeholder='Email']"))
              .sendKeys("user@example.com");
        driver.findElement(By.cssSelector("input[placeholder='Password']"))
              .sendKeys("password123");
        driver.findElement(By.cssSelector("button[type='submit']"))
              .click();

        try { Thread.sleep(2000); } catch (InterruptedException e) {}

        // Создание статьи
        driver.get("http://localhost:3000/editor");
        driver.findElement(By.cssSelector("input[placeholder='Article Title']"))
              .sendKeys("My Article");
        driver.findElement(By.cssSelector("input[placeholder=\"What's this article about?\"]"))
              .sendKeys("Description");
        driver.findElement(By.cssSelector("textarea[placeholder='Write your article']"))
              .sendKeys("Article body text");
        driver.findElement(By.cssSelector("button[type='submit']"))
              .click();

        try { Thread.sleep(3000); } catch (InterruptedException e) {}

        String title = driver.findElement(By.cssSelector("h1")).getText();
        assertEquals("My Article", title);
    }
}
```

### Что плохо

| Проблема | Последствие |
|----------|------------|
| Дублирование локаторов | Изменение одного элемента на UI — правка в десятках мест |
| Захардкоженные данные | Невозможно запустить на другом окружении |
| Thread.sleep вместо ожиданий | Нестабильные тесты (flaky tests) |
| Логика логина скопирована | Изменение формы логина ломает все тесты |
| Нет разделения ответственности | Тест одновременно знает о локаторах, данных и логике |

---

## Stage 2: Page Object Model

### Проблема Stage 1

Локаторы и действия размазаны по тестам. При изменении UI нужно править каждый тест.

### Решение

Вынести знание о странице (локаторы + действия) в отдельный класс — Page Object.
Тест оперирует бизнес-действиями, а не CSS-селекторами.

### Структура проекта

```
src/test/java/
├── pages/
│   ├── BasePage.java
│   ├── LoginPage.java
│   ├── ArticlePage.java
│   └── EditorPage.java
└── tests/
    ├── BaseTest.java
    ├── LoginTest.java
    └── ArticleTest.java
```

### Код: BasePage

```java
public abstract class BasePage {

    protected WebDriver driver;
    protected WebDriverWait wait;

    public BasePage(WebDriver driver) {
        this.driver = driver;
        // Явное ожидание — замена Thread.sleep
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(10));
    }

    // Общие методы для всех страниц
    protected WebElement waitForElement(By locator) {
        return wait.until(ExpectedConditions.visibilityOfElementLocated(locator));
    }

    protected void click(By locator) {
        waitForElement(locator).click();
    }

    protected void type(By locator, String text) {
        WebElement element = waitForElement(locator);
        element.clear();
        element.sendKeys(text);
    }
}
```

### Код: LoginPage

```java
public class LoginPage extends BasePage {

    // Локаторы собраны в одном месте
    private final By emailInput = By.cssSelector("input[placeholder='Email']");
    private final By passwordInput = By.cssSelector("input[placeholder='Password']");
    private final By submitButton = By.cssSelector("button[type='submit']");

    public LoginPage(WebDriver driver) {
        super(driver);
    }

    // Бизнес-метод — скрывает детали реализации
    public void loginAs(String email, String password) {
        driver.get("http://localhost:3000/login");
        type(emailInput, email);
        type(passwordInput, password);
        click(submitButton);
    }
}
```

### Код: Тест после рефакторинга

```java
public class LoginTest extends BaseTest {

    @Test
    void testSuccessfulLogin() {
        // Тест читается как бизнес-сценарий
        loginPage.loginAs("user@example.com", "password123");
        assertTrue(homePage.isUserLoggedIn("testuser"));
    }
}
```

### Что улучшилось

- Локатор меняется в одном месте — LoginPage
- Тесты читаются как пользовательские сценарии
- Повторное использование: loginAs() вызывается из любого теста

---

## Stage 3: Externalization конфигурации

### Проблема Stage 2

URL, учётные данные, таймауты захардкожены в коде. Нельзя переключить окружение
без пересборки проекта.

### Решение

Вынести всю конфигурацию в файлы properties/YAML и читать через конфигурационный класс.

### Файл: src/test/resources/config.properties

```properties
# Базовый URL приложения — меняется для каждого окружения
base.url=http://localhost:3000
# Учётные данные для тестового пользователя
test.user.email=user@example.com
test.user.password=password123
# Настройки браузера
browser=chrome
headless=false
# Таймауты (в секундах)
timeout.implicit=10
timeout.explicit=15
timeout.page.load=30
```

### Файл: src/test/resources/config-staging.properties

```properties
# Staging-окружение — переопределяем только то, что отличается
base.url=https://staging.myapp.com
test.user.email=staging-user@example.com
test.user.password=stagingPass456
headless=true
```

### Код: ConfigReader

```java
public class ConfigReader {

    private static final Properties properties = new Properties();

    static {
        // Определяем окружение из системного свойства или переменной среды
        String env = System.getProperty("env", "config");
        String fileName = env + ".properties";

        try (InputStream input = ConfigReader.class.getClassLoader()
                .getResourceAsStream(fileName)) {
            if (input == null) {
                throw new RuntimeException(
                    "Файл конфигурации не найден: " + fileName);
            }
            properties.load(input);
        } catch (IOException e) {
            throw new RuntimeException(
                "Ошибка загрузки конфигурации: " + e.getMessage());
        }
    }

    public static String getBaseUrl() {
        return properties.getProperty("base.url");
    }

    public static String getBrowser() {
        return properties.getProperty("browser", "chrome");
    }

    public static boolean isHeadless() {
        return Boolean.parseBoolean(
            properties.getProperty("headless", "false"));
    }

    public static int getExplicitTimeout() {
        return Integer.parseInt(
            properties.getProperty("timeout.explicit", "15"));
    }
}
```

### Запуск с разными окружениями

```bash
# Локальный запуск (по умолчанию config.properties)
mvn test

# Запуск на staging-окружении
mvn test -Denv=config-staging

# Запуск в headless-режиме с переопределением через CLI
mvn test -Dheadless=true -Dbrowser=firefox
```

---

## Stage 4: Утилиты — генерация данных, работа с файлами

### Проблема Stage 3

Тестовые данные создаются вручную и часто конфликтуют между запусками.
Нет переиспользуемых утилит для типовых операций.

### Решение

Создать пакет утилит: генерация данных, работа с файлами, форматирование дат, retry-логика.

### Структура

```
src/test/java/
├── utils/
│   ├── DataGenerator.java      // Генерация тестовых данных
│   ├── FileUtils.java          // Чтение/запись файлов
│   ├── DateUtils.java          // Работа с датами
│   ├── RetryUtils.java         // Повторные попытки для нестабильных операций
│   └── JsonUtils.java          // Сериализация/десериализация JSON
```

### Код: DataGenerator

```java
public class DataGenerator {

    private static final Faker faker = new Faker(new Locale("en"));
    private static final Random random = new Random();

    // Уникальный email — никогда не совпадёт с существующим
    public static String uniqueEmail() {
        return "test_" + System.currentTimeMillis()
               + "_" + random.nextInt(1000) + "@test.com";
    }

    // Уникальное имя пользователя
    public static String uniqueUsername() {
        return "user_" + faker.name().firstName().toLowerCase()
               + "_" + System.currentTimeMillis();
    }

    // Генерация статьи с реалистичными данными
    public static Map<String, String> articleData() {
        Map<String, String> data = new HashMap<>();
        data.put("title", faker.book().title() + " " + System.currentTimeMillis());
        data.put("description", faker.lorem().sentence(10));
        data.put("body", faker.lorem().paragraphs(3)
                .stream().collect(Collectors.joining("\n\n")));
        data.put("tag", faker.programmingLanguage().name().toLowerCase());
        return data;
    }

    // Пароль, соответствующий типичным требованиям безопасности
    public static String strongPassword() {
        return "Test_" + faker.internet().password(8, 12, true, true) + "1!";
    }
}
```

### Код: RetryUtils

```java
public class RetryUtils {

    // Повторная попытка выполнения действия с указанным количеством попыток
    public static <T> T retry(int maxAttempts, Duration delay, Supplier<T> action) {
        Exception lastException = null;

        for (int attempt = 1; attempt <= maxAttempts; attempt++) {
            try {
                return action.get();
            } catch (Exception e) {
                lastException = e;
                if (attempt < maxAttempts) {
                    try {
                        Thread.sleep(delay.toMillis());
                    } catch (InterruptedException ie) {
                        Thread.currentThread().interrupt();
                        throw new RuntimeException(ie);
                    }
                }
            }
        }
        throw new RuntimeException(
            "Действие не выполнено после " + maxAttempts + " попыток",
            lastException
        );
    }
}
```

### Использование в тесте

```java
@Test
void testCreateArticleWithGeneratedData() {
    // Данные генерируются автоматически — каждый запуск уникален
    var data = DataGenerator.articleData();

    loginPage.loginAs(ConfigReader.getTestEmail(), ConfigReader.getTestPassword());
    editorPage.createArticle(data.get("title"), data.get("description"),
                             data.get("body"), data.get("tag"));

    assertEquals(data.get("title"), articlePage.getTitle());
}
```

---

## Stage 5: Многослойный фреймворк — UI + API + DB

### Проблема Stage 4

Все тесты работают через UI — это медленно и ненадёжно. Предусловия создаются через UI,
хотя быстрее и надёжнее делать это через API или напрямую в базе данных.

### Решение

Разделить фреймворк на слои. Каждый слой имеет свою ответственность:
- **UI Layer** — только проверка визуального поведения
- **API Layer** — быстрое создание предусловий, проверка бизнес-логики
- **DB Layer** — верификация данных в базе, подготовка тестовых данных

### Архитектура

```
src/test/java/
├── api/
│   ├── ApiClient.java            // Базовый REST-клиент
│   ├── AuthApi.java              // Эндпоинты авторизации
│   ├── ArticleApi.java           // Эндпоинты статей
│   └── models/                   // POJO для API
│       ├── UserRequest.java
│       ├── UserResponse.java
│       └── ArticleResponse.java
├── db/
│   ├── DatabaseClient.java       // Подключение к БД
│   ├── UserRepository.java       // Запросы к таблице пользователей
│   └── ArticleRepository.java    // Запросы к таблице статей
├── ui/
│   ├── pages/                    // Page Objects
│   └── tests/                    // UI-тесты
├── integration/
│   └── tests/                    // Тесты, использующие все три слоя
└── utils/
```

### Код: ApiClient (API Layer)

```java
public class ApiClient {

    private static final String BASE_URL = ConfigReader.getBaseUrl();

    // Создание пользователя через API — в 10 раз быстрее, чем через UI
    public static String createUserAndGetToken(String email, String password,
                                                String username) {
        String body = String.format(
            "{\"user\":{\"email\":\"%s\",\"password\":\"%s\",\"username\":\"%s\"}}",
            email, password, username
        );

        return given()
            .contentType(ContentType.JSON)
            .body(body)
        .when()
            .post(BASE_URL + "/api/users")
        .then()
            .statusCode(200)
            .extract()
            .path("user.token");
    }

    // Создание статьи через API — предусловие для UI-теста
    public static String createArticle(String token, String title,
                                        String description, String body) {
        String requestBody = String.format(
            "{\"article\":{\"title\":\"%s\",\"description\":\"%s\"," +
            "\"body\":\"%s\",\"tagList\":[\"test\"]}}",
            title, description, body
        );

        return given()
            .contentType(ContentType.JSON)
            .header("Authorization", "Token " + token)
            .body(requestBody)
        .when()
            .post(BASE_URL + "/api/articles")
        .then()
            .statusCode(200)
            .extract()
            .path("article.slug");
    }
}
```

### Код: DatabaseClient (DB Layer)

```java
public class DatabaseClient {

    private static final String DB_URL = ConfigReader.getDbUrl();
    private static final String DB_USER = ConfigReader.getDbUser();
    private static final String DB_PASSWORD = ConfigReader.getDbPassword();

    // Проверка, что запись создана в базе данных
    public static boolean articleExistsInDb(String slug) {
        String sql = "SELECT COUNT(*) FROM articles WHERE slug = ?";

        try (Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD);
             PreparedStatement stmt = conn.prepareStatement(sql)) {

            stmt.setString(1, slug);
            ResultSet rs = stmt.executeQuery();
            rs.next();
            return rs.getInt(1) > 0;

        } catch (SQLException e) {
            throw new RuntimeException(
                "Ошибка проверки статьи в базе данных: " + e.getMessage());
        }
    }

    // Очистка тестовых данных после прогона
    public static void cleanupTestData(String emailPattern) {
        String sql = "DELETE FROM users WHERE email LIKE ?";

        try (Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD);
             PreparedStatement stmt = conn.prepareStatement(sql)) {

            stmt.setString(1, emailPattern + "%");
            int deleted = stmt.executeUpdate();
            System.out.println("Удалено тестовых пользователей: " + deleted);

        } catch (SQLException e) {
            throw new RuntimeException(
                "Ошибка очистки тестовых данных: " + e.getMessage());
        }
    }
}
```

### Код: Многослойный тест

```java
@Test
@DisplayName("Создание статьи через UI с проверкой в API и БД")
void testCreateArticleEndToEnd() {
    // 1. ПРЕДУСЛОВИЕ через API — быстро создаём пользователя
    String email = DataGenerator.uniqueEmail();
    String password = DataGenerator.strongPassword();
    String username = DataGenerator.uniqueUsername();
    String token = ApiClient.createUserAndGetToken(email, password, username);

    // 2. ДЕЙСТВИЕ через UI — проверяем именно пользовательский сценарий
    loginPage.loginAs(email, password);
    editorPage.open();
    String title = "Article " + System.currentTimeMillis();
    editorPage.createArticle(title, "desc", "body", "tag");

    // 3. ПРОВЕРКА UI — статья отображается
    assertEquals(title, articlePage.getTitle());

    // 4. ПРОВЕРКА API — статья доступна через API
    String slug = title.toLowerCase().replace(" ", "-");
    Response response = given()
        .header("Authorization", "Token " + token)
        .get(ConfigReader.getBaseUrl() + "/api/articles/" + slug);
    assertEquals(200, response.getStatusCode());

    // 5. ПРОВЕРКА БД — запись существует в базе данных
    assertTrue(DatabaseClient.articleExistsInDb(slug),
               "Статья должна быть в базе данных");
}
```

---

## Сравнение стадий

| Характеристика | Stage 1 | Stage 2 | Stage 3 | Stage 4 | Stage 5 |
|---------------|---------|---------|---------|---------|---------|
| Поддерживаемость | Низкая | Средняя | Высокая | Высокая | Высокая |
| Переиспользование кода | Нет | Да | Да | Да | Да |
| Переключение окружений | Нет | Нет | Да | Да | Да |
| Уникальные тестовые данные | Нет | Нет | Нет | Да | Да |
| Скорость выполнения | Низкая | Низкая | Низкая | Низкая | Высокая |
| Надёжность проверок | Низкая | Средняя | Средняя | Средняя | Высокая |

---

## Практическое задание

### Задание 1: Рефакторинг из Stage 1 в Stage 2

1. Возьмите три теста, написанных «в лоб» (без Page Object)
2. Выделите все уникальные страницы, с которыми взаимодействуют тесты
3. Для каждой страницы создайте Page Object с локаторами и бизнес-методами
4. Перепишите тесты, используя Page Objects
5. Убедитесь, что все тесты проходят

**Ожидаемый результат:** Pull request с рефакторингом, где в описании указано:
- Сколько дублирующихся локаторов удалено
- Сколько строк кода сэкономлено
- Какие Page Objects созданы

### Задание 2: Добавьте конфигурацию (Stage 3)

1. Создайте `config.properties` и `config-staging.properties`
2. Реализуйте `ConfigReader` с поддержкой переменной `env`
3. Замените все захардкоженные значения на вызовы ConfigReader
4. Проверьте переключение окружений через `-Denv=config-staging`

### Задание 3: Многослойный тест (Stage 5)

1. Реализуйте ApiClient для создания пользователя через REST API
2. Реализуйте DatabaseClient для проверки данных в PostgreSQL
3. Напишите тест, который:
   - Создаёт предусловия через API
   - Выполняет действие через UI
   - Проверяет результат через API и DB
4. Сравните время выполнения этого теста с чисто UI-тестом

**Критерии оценки:**
- Код компилируется и тесты запускаются
- Каждый слой имеет чёткую ответственность
- Конфигурация вынесена в properties-файлы
- Тестовые данные генерируются динамически

---

## Чек-лист самопроверки

- [ ] Понимаю, почему «всё в одном файле» — антипаттерн
- [ ] Могу создать Page Object для любой страницы
- [ ] Умею выносить конфигурацию в properties-файлы
- [ ] Могу написать утилиту генерации тестовых данных
- [ ] Понимаю разницу между UI-, API- и DB-слоями фреймворка
- [ ] Могу объяснить, когда какой слой использовать для предусловий и проверок
- [ ] Способен провести рефакторинг фреймворка от Stage 1 до Stage 5
