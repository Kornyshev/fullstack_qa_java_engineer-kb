# Порождающие паттерны (Creational Patterns)

## Обзор

Порождающие паттерны (Creational Patterns) абстрагируют процесс создания объектов, делая систему
независимой от способа создания, композиции и представления объектов. Вместо того чтобы инстанцировать
классы напрямую через `new`, порождающие паттерны предоставляют гибкие механизмы создания, которые
повышают переиспользуемость и упрощают сопровождение кода.

Для QA Automation Engineer три ключевых порождающих паттерна:

- **Singleton** — единственный экземпляр (конфигурация, менеджер WebDriver).
- **Factory Method** — создание объектов без указания конкретного класса (драйверы для разных браузеров).
- **Builder** — пошаговое конструирование сложных объектов (тестовые данные, тела HTTP-запросов).

---

## Singleton

### Назначение

Singleton гарантирует, что класс имеет **ровно один экземпляр**, и предоставляет глобальную точку
доступа к нему.

### ASCII-диаграмма

```
┌───────────────────────────────┐
│         Singleton             │
├───────────────────────────────┤
│ - instance: Singleton         │
│ - config: Properties          │
├───────────────────────────────┤
│ - Singleton()                 │
│ + getInstance(): Singleton    │
│ + getProperty(key): String    │
└───────────────────────────────┘
        │
        │  Единственный экземпляр
        │  создаётся при первом вызове
        v
   ┌──────────┐
   │ instance │ ← хранится в static-поле
   └──────────┘
```

### Применение в тестировании

1. **Хранение конфигурации** — параметры окружения (URL, credentials, timeouts) загружаются один раз.
2. **Менеджер WebDriver** — управление жизненным циклом драйвера.
3. **Пул соединений** — единый пул для работы с БД в тестах.

### Пример: ConfigHolder

```java
/**
 * Singleton для хранения тестовой конфигурации.
 * Загружает properties-файл один раз и предоставляет доступ ко всем параметрам.
 */
public final class ConfigHolder {

    // volatile гарантирует корректную работу в многопоточной среде
    private static volatile ConfigHolder instance;
    private final Properties properties;

    private ConfigHolder() {
        // Приватный конструктор — запрещаем создание извне
        properties = new Properties();
        try (var stream = getClass().getClassLoader()
                .getResourceAsStream("test.properties")) {
            if (stream == null) {
                throw new IllegalStateException("Файл конфигурации не найден");
            }
            properties.load(stream);
        } catch (IOException e) {
            throw new UncheckedIOException("Ошибка загрузки конфигурации", e);
        }
    }

    /**
     * Потокобезопасный метод получения единственного экземпляра.
     * Используется Double-Checked Locking.
     */
    public static ConfigHolder getInstance() {
        if (instance == null) {
            synchronized (ConfigHolder.class) {
                if (instance == null) {
                    instance = new ConfigHolder();
                }
            }
        }
        return instance;
    }

    public String getProperty(String key) {
        return properties.getProperty(key);
    }

    public String getBaseUrl() {
        return getProperty("base.url");
    }

    public int getTimeout() {
        return Integer.parseInt(getProperty("timeout.seconds"));
    }
}
```

### Пример: WebDriverManager (ThreadLocal Singleton)

```java
/**
 * Менеджер WebDriver с поддержкой параллельного запуска тестов.
 * Каждый поток получает свой экземпляр WebDriver через ThreadLocal.
 */
public final class WebDriverManager {

    // ThreadLocal гарантирует изоляцию драйверов между потоками
    private static final ThreadLocal<WebDriver> driverThreadLocal = new ThreadLocal<>();

    private WebDriverManager() {
        // Запрещаем инстанцирование
    }

    public static WebDriver getDriver() {
        if (driverThreadLocal.get() == null) {
            // Создаём драйвер при первом обращении в данном потоке
            WebDriver driver = createDriver();
            driverThreadLocal.set(driver);
        }
        return driverThreadLocal.get();
    }

    public static void quitDriver() {
        WebDriver driver = driverThreadLocal.get();
        if (driver != null) {
            driver.quit();
            driverThreadLocal.remove(); // Очищаем ThreadLocal во избежание утечки памяти
        }
    }

    private static WebDriver createDriver() {
        String browser = ConfigHolder.getInstance().getProperty("browser");
        return switch (browser.toLowerCase()) {
            case "chrome" -> new ChromeDriver();
            case "firefox" -> new FirefoxDriver();
            case "edge" -> new EdgeDriver();
            default -> throw new IllegalArgumentException(
                "Неподдерживаемый браузер: " + browser);
        };
    }
}
```

### Использование в тестах

```java
class LoginTest {

    @BeforeEach
    void setUp() {
        // Драйвер создаётся автоматически при первом вызове
        WebDriverManager.getDriver()
            .get(ConfigHolder.getInstance().getBaseUrl());
    }

    @AfterEach
    void tearDown() {
        WebDriverManager.quitDriver();
    }

    @Test
    void shouldLoginSuccessfully() {
        var loginPage = new LoginPage(WebDriverManager.getDriver());
        var dashboard = loginPage.loginAs("admin", "secret");
        assertEquals("Dashboard", dashboard.getTitle());
    }
}
```

---

## Factory Method

### Назначение

Factory Method определяет интерфейс для создания объекта, но **позволяет подклассам решить**,
какой конкретный класс создавать. Фабричный метод отделяет код создания объектов от кода,
который их использует.

### ASCII-диаграмма

```
         ┌──────────────────────┐
         │  WebDriverFactory    │ (абстрактный создатель)
         │  «interface»         │
         ├──────────────────────┤
         │ + createDriver():    │
         │        WebDriver     │
         └──────────┬───────────┘
                    │
       ┌────────────┼────────────┐
       │            │            │
       v            v            v
┌────────────┐┌────────────┐┌────────────┐
│ ChromeDriver││ FirefoxDriver││ EdgeDriver │
│  Factory   ││  Factory   ││  Factory   │
├────────────┤├────────────┤├────────────┤
│+createDriver││+createDriver││+createDriver│
└────────────┘└────────────┘└────────────┘
       │            │            │
       v            v            v
   ChromeDriver FirefoxDriver EdgeDriver
```

### Применение в тестировании

1. **Кроссбраузерное тестирование** — создание нужного драйвера по конфигурации.
2. **Создание тестовых данных** — фабрика пользователей с разными ролями.
3. **Выбор окружения** — создание API-клиента для dev/staging/prod.

### Пример: WebDriverFactory

```java
/**
 * Интерфейс фабрики для создания WebDriver.
 * Каждая реализация знает, как создать драйвер конкретного браузера.
 */
public interface WebDriverFactory {
    WebDriver createDriver();
}

/**
 * Фабрика для Chrome с настроенными опциями.
 */
public class ChromeDriverFactory implements WebDriverFactory {

    @Override
    public WebDriver createDriver() {
        var options = new ChromeOptions();
        options.addArguments("--start-maximized");
        options.addArguments("--disable-notifications");

        // Если запуск в CI — включаем headless-режим
        if (Boolean.parseBoolean(System.getenv("CI"))) {
            options.addArguments("--headless=new");
            options.addArguments("--no-sandbox");
            options.addArguments("--disable-dev-shm-usage");
        }
        return new ChromeDriver(options);
    }
}

/**
 * Фабрика для Firefox.
 */
public class FirefoxDriverFactory implements WebDriverFactory {

    @Override
    public WebDriver createDriver() {
        var options = new FirefoxOptions();
        options.addPreference("dom.webnotifications.enabled", false);

        if (Boolean.parseBoolean(System.getenv("CI"))) {
            options.addArguments("--headless");
        }
        return new FirefoxDriver(options);
    }
}

/**
 * Фабрика для удалённого запуска через Selenium Grid.
 */
public class RemoteDriverFactory implements WebDriverFactory {

    private final String gridUrl;
    private final Capabilities capabilities;

    public RemoteDriverFactory(String gridUrl, Capabilities capabilities) {
        this.gridUrl = gridUrl;
        this.capabilities = capabilities;
    }

    @Override
    public WebDriver createDriver() {
        try {
            return new RemoteWebDriver(new URL(gridUrl), capabilities);
        } catch (MalformedURLException e) {
            throw new IllegalArgumentException("Некорректный URL Grid: " + gridUrl, e);
        }
    }
}
```

### Фабричный провайдер (Static Factory)

```java
/**
 * Статическая фабрика, выбирающая нужную реализацию по конфигурации.
 */
public final class DriverFactoryProvider {

    private DriverFactoryProvider() {}

    public static WebDriverFactory getFactory() {
        String browser = ConfigHolder.getInstance().getProperty("browser");
        String executionMode = ConfigHolder.getInstance().getProperty("execution.mode");

        if ("remote".equalsIgnoreCase(executionMode)) {
            String gridUrl = ConfigHolder.getInstance().getProperty("grid.url");
            return new RemoteDriverFactory(gridUrl, getCapabilities(browser));
        }

        return switch (browser.toLowerCase()) {
            case "chrome"  -> new ChromeDriverFactory();
            case "firefox" -> new FirefoxDriverFactory();
            default -> throw new IllegalArgumentException(
                "Неизвестный браузер: " + browser);
        };
    }

    private static Capabilities getCapabilities(String browser) {
        return switch (browser.toLowerCase()) {
            case "chrome"  -> new ChromeOptions();
            case "firefox" -> new FirefoxOptions();
            default -> throw new IllegalArgumentException(
                "Неизвестный браузер: " + browser);
        };
    }
}
```

---

## Builder

### Назначение

Builder отделяет конструирование сложного объекта от его представления, позволяя одному и тому же
процессу конструирования создавать **различные представления**. Особенно полезен, когда объект
имеет множество параметров, часть из которых необязательна.

### ASCII-диаграмма

```
┌─────────────────────────────────┐
│          UserBuilder            │
├─────────────────────────────────┤
│ - name: String                  │
│ - email: String                 │
│ - role: Role                    │
│ - age: Integer                  │
│ - active: boolean               │
├─────────────────────────────────┤
│ + name(String): UserBuilder     │
│ + email(String): UserBuilder    │
│ + role(Role): UserBuilder       │
│ + age(Integer): UserBuilder     │
│ + active(boolean): UserBuilder  │
│ + build(): User                 │
└─────────────────────────────────┘
         │
         │ build()
         v
┌─────────────────────────────────┐
│            User                 │
├─────────────────────────────────┤
│ - name, email, role, age, active│
└─────────────────────────────────┘
```

### Применение в тестировании

1. **Создание тестовых данных** — пользователи, заказы, продукты с множеством полей.
2. **Формирование тел HTTP-запросов** — JSON-объекты для REST API тестов.
3. **Конфигурация тестов** — параметры запуска с множеством опций.

### Пример: Builder для тестовых данных

```java
/**
 * Модель пользователя с Builder-ом для удобного создания в тестах.
 */
public class User {

    private final String name;
    private final String email;
    private final Role role;
    private final Integer age;
    private final boolean active;

    private User(Builder builder) {
        this.name = builder.name;
        this.email = builder.email;
        this.role = builder.role;
        this.age = builder.age;
        this.active = builder.active;
    }

    // Геттеры
    public String getName() { return name; }
    public String getEmail() { return email; }
    public Role getRole() { return role; }
    public Integer getAge() { return age; }
    public boolean isActive() { return active; }

    // Точка входа в Builder
    public static Builder builder() {
        return new Builder();
    }

    public static class Builder {
        private String name;
        private String email;
        private Role role = Role.USER; // Значение по умолчанию
        private Integer age;
        private boolean active = true; // Значение по умолчанию

        public Builder name(String name) {
            this.name = name;
            return this; // Возвращаем this для цепочки вызовов
        }

        public Builder email(String email) {
            this.email = email;
            return this;
        }

        public Builder role(Role role) {
            this.role = role;
            return this;
        }

        public Builder age(Integer age) {
            this.age = age;
            return this;
        }

        public Builder active(boolean active) {
            this.active = active;
            return this;
        }

        /**
         * Финальный метод — собирает объект.
         * Здесь можно добавить валидацию обязательных полей.
         */
        public User build() {
            if (name == null || name.isBlank()) {
                throw new IllegalStateException("Имя пользователя обязательно");
            }
            if (email == null || email.isBlank()) {
                throw new IllegalStateException("Email обязателен");
            }
            return new User(this);
        }
    }
}
```

### Пример: Builder для REST API тела запроса

```java
/**
 * Builder для создания тела запроса на создание заказа.
 * Используется в API-тестах для формирования JSON.
 */
public class OrderRequest {

    private final String productId;
    private final int quantity;
    private final String deliveryAddress;
    private final String couponCode;
    private final boolean expressDelivery;

    private OrderRequest(Builder builder) {
        this.productId = builder.productId;
        this.quantity = builder.quantity;
        this.deliveryAddress = builder.deliveryAddress;
        this.couponCode = builder.couponCode;
        this.expressDelivery = builder.expressDelivery;
    }

    public static Builder builder() {
        return new Builder();
    }

    // Геттеры...

    public static class Builder {
        private String productId;
        private int quantity = 1;
        private String deliveryAddress;
        private String couponCode;
        private boolean expressDelivery = false;

        public Builder productId(String productId) {
            this.productId = productId;
            return this;
        }

        public Builder quantity(int quantity) {
            this.quantity = quantity;
            return this;
        }

        public Builder deliveryAddress(String address) {
            this.deliveryAddress = address;
            return this;
        }

        public Builder couponCode(String code) {
            this.couponCode = code;
            return this;
        }

        public Builder expressDelivery(boolean express) {
            this.expressDelivery = express;
            return this;
        }

        public OrderRequest build() {
            Objects.requireNonNull(productId, "productId обязателен");
            Objects.requireNonNull(deliveryAddress, "Адрес доставки обязателен");
            if (quantity <= 0) {
                throw new IllegalArgumentException("Количество должно быть положительным");
            }
            return new OrderRequest(this);
        }
    }
}
```

### Использование Builder-ов в тестах

```java
class OrderApiTest {

    @Test
    void shouldCreateOrderWithCoupon() {
        // Читаемое, декларативное создание тестовых данных
        var order = OrderRequest.builder()
            .productId("PROD-001")
            .quantity(3)
            .deliveryAddress("ул. Тестовая, д. 42")
            .couponCode("DISCOUNT10")
            .expressDelivery(true)
            .build();

        var response = apiClient.createOrder(order);

        assertEquals(201, response.getStatusCode());
        assertNotNull(response.getBody().getOrderId());
    }

    @Test
    void shouldRejectOrderWithZeroQuantity() {
        // Builder с невалидными данными — проверяем валидацию
        assertThrows(IllegalArgumentException.class, () ->
            OrderRequest.builder()
                .productId("PROD-001")
                .quantity(0) // Невалидное значение
                .deliveryAddress("ул. Тестовая, д. 42")
                .build()
        );
    }
}
```

---

## Связь с тестированием

| Паттерн       | Проблема в тестировании              | Решение                                          |
|---------------|--------------------------------------|--------------------------------------------------|
| **Singleton** | Дублирование загрузки конфигурации   | Одна точка доступа к настройкам                   |
| **Factory**   | Хардкод конкретного браузера в тестах | Выбор драйвера определяется конфигурацией         |
| **Builder**   | Конструкторы с 10+ параметрами       | Пошаговое, читаемое создание объектов             |

Порождающие паттерны особенно важны на этапе **настройки тестового окружения** (test setup).
Правильное применение этих паттернов делает тесты:

- **Независимыми** от конкретной реализации (браузера, окружения).
- **Читаемыми** — создание тестовых данных выглядит как спецификация.
- **Поддерживаемыми** — изменения в процессе создания не затрагивают тесты.

---

## Типичные ошибки

1. **Singleton без потокобезопасности** — в многопоточных тестах (параллельный запуск) это
   приводит к race condition и непредсказуемым падениям.
2. **Singleton с изменяемым состоянием** — если тесты модифицируют состояние Singleton-а,
   это нарушает изоляцию тестов.
3. **Factory без значений по умолчанию** — заставляет каждый тест полностью конфигурировать
   создаваемый объект, увеличивая объём boilerplate-кода.
4. **Builder без валидации в `build()`** — позволяет создавать невалидные объекты, что приводит
   к ошибкам далеко от места создания.
5. **Забытый `ThreadLocal.remove()`** — утечка памяти при использовании пула потоков.
6. **God Factory** — одна фабрика создаёт все типы объектов, нарушая Single Responsibility.

---

## Вопросы на интервью

### Уровень Junior

- Что такое Singleton? Приведите пример из тестирования.
- Зачем нужен паттерн Builder? Когда его стоит использовать?
- Как создать WebDriver для разных браузеров без `if-else` в тестах?
- Что такое `ThreadLocal` и зачем он нужен при параллельном запуске тестов?

### Уровень Middle

- Как обеспечить потокобезопасность Singleton? Опишите Double-Checked Locking.
- Чем Factory Method отличается от Abstract Factory? Приведите примеры из тестирования.
- Как реализовать Builder с обязательными и опциональными полями?
- Какие проблемы возникают при использовании Singleton в тестах? Как их избежать?
- Как бы вы организовали создание тестовых данных для сложной доменной модели?

### Уровень Senior

- Когда Singleton — это антипаттерн? Какие альтернативы существуют (DI-контейнер)?
- Как реализовать фабрику с использованием Java Service Loader?
- Сравните подходы: Builder, Telescoping Constructor, JavaBeans. Когда какой выбрать?
- Как протестировать сам Singleton на корректность работы в многопоточной среде?
- Как вы масштабируете создание тестовых данных при росте количества сущностей в системе?

---

## Практические задания

1. **Singleton: ConfigHolder.** Реализуйте потокобезопасный Singleton для хранения тестовой
   конфигурации. Загрузите данные из `test.properties`. Добавьте метод для получения свойства
   с дефолтным значением.

2. **Factory: Multi-browser.** Создайте фабрику WebDriver с поддержкой Chrome, Firefox и
   удалённого запуска. Браузер должен определяться системным свойством `browser`.

3. **Builder: TestData.** Реализуйте Builder для сущности `Employee` со следующими полями:
   `name` (обязательное), `email` (обязательное), `department`, `position`, `salary`, `hireDate`.
   Добавьте валидацию в метод `build()`.

4. **Комбинация паттернов.** Создайте mini-фреймворк, который:
   - Использует Singleton для конфигурации.
   - Использует Factory для создания WebDriver.
   - Использует Builder для создания тестовых пользователей.
   Напишите один тест, демонстрирующий работу всех трёх паттернов вместе.

5. **Рефакторинг.** Перепишите следующий код, применив порождающие паттерны:
   ```java
   @Test
   void test() {
       WebDriver driver = new ChromeDriver();
       driver.get("http://localhost:8080");
       // Создание пользователя через конструктор с 8 параметрами
       User user = new User("John", "john@test.com", "ADMIN", 30,
                            true, "IT", "Senior QA", 100000);
   }
   ```

---

## Дополнительные ресурсы

- **Refactoring Guru — Creational Patterns** — наглядные диаграммы и примеры на Java.
- **Effective Java, Item 2** — Joshua Bloch — обоснование Builder как предпочтительного подхода.
- **Effective Java, Item 3** — Joshua Bloch — реализация Singleton через enum.
- **Selenium WebDriver Architecture** — как Selenium использует Factory внутри.
- **Project Lombok `@Builder`** — автогенерация Builder-ов для уменьшения boilerplate.
- **MapStruct** — альтернатива Builder для маппинга между DTO и доменными объектами.
