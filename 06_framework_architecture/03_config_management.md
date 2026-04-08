# Управление конфигурацией

## Обзор

Управление конфигурацией — критически важный аспект тестового фреймворка. Жёстко прописанные URL, логины,
параметры браузера и прочие настройки делают фреймворк негибким и привязанным к конкретному окружению.
Правильный подход — вынести всю конфигурацию во внешние файлы, переменные окружения и системные свойства
с чёткой иерархией приоритетов. В этом разделе мы разберём все подходы к управлению конфигурацией,
библиотеку Owner, профили окружений и лучшие практики.

---

## Зачем выносить конфигурацию

Хардкод конфигурации — одна из самых частых ошибок начинающих автоматизаторов:

```java
// ПЛОХО — хардкод в тесте
@BeforeEach
void setup() {
    Configuration.baseUrl = "https://dev.myshop.com";  // А если staging?
    Configuration.browser = "chrome";                    // А если firefox?
    RestAssured.baseURI = "https://api.dev.myshop.com"; // Дублирование URL
}
```

Проблемы с таким подходом:

| Проблема | Последствие |
|----------|------------|
| Смена окружения | Нужно менять код в десятках файлов |
| CI/CD | Невозможно параметризировать без перекомпиляции |
| Безопасность | Пароли и ключи попадают в Git |
| Командная работа | У каждого свои локальные настройки |
| Дублирование | Один URL указан в 50 местах |

---

## Источники конфигурации

### 1. Property-файлы (`.properties`)

Самый распространённый способ. Файлы хранятся в `src/test/resources/config/`:

```properties
# application.properties — общие настройки
browser=chrome
browser.size=1920x1080
browser.headless=false
timeout=10000
allure.results.dir=allure-results
```

```properties
# dev.properties — настройки dev-окружения
base.url=https://dev.myshop.com
api.base.url=https://api.dev.myshop.com
db.url=jdbc:postgresql://dev-db:5432/shop
db.username=test_user
db.password=test_pass
```

```properties
# staging.properties — настройки staging
base.url=https://staging.myshop.com
api.base.url=https://api.staging.myshop.com
db.url=jdbc:postgresql://staging-db:5432/shop
db.username=staging_user
db.password=staging_pass
```

Чтение вручную:

```java
import java.io.InputStream;
import java.util.Properties;

public class PropertiesReader {

    // Чтение property-файла из classpath
    public static Properties load(String fileName) {
        Properties properties = new Properties();
        try (InputStream input = PropertiesReader.class
                .getClassLoader()
                .getResourceAsStream(fileName)) {
            if (input == null) {
                throw new IllegalArgumentException(
                    "Файл не найден: " + fileName);
            }
            properties.load(input);
        } catch (Exception e) {
            throw new RuntimeException(
                "Ошибка чтения конфигурации: " + fileName, e);
        }
        return properties;
    }
}
```

### 2. Переменные окружения (Environment Variables)

Идеальны для CI/CD и секретов:

```java
// Чтение переменной окружения с fallback на значение по умолчанию
String baseUrl = System.getenv("BASE_URL");
if (baseUrl == null) {
    baseUrl = "https://dev.myshop.com";
}
```

Установка в CI/CD (GitHub Actions):

```yaml
env:
  BASE_URL: https://staging.myshop.com
  API_TOKEN: ${{ secrets.API_TOKEN }}
  BROWSER: chrome
```

### 3. Системные свойства (System Properties)

Передаются через `-D` при запуске:

```bash
mvn test -Dbase.url=https://staging.myshop.com -Dbrowser=firefox
```

Чтение в коде:

```java
// Системное свойство имеет приоритет над файлом
String baseUrl = System.getProperty("base.url", "https://dev.myshop.com");
```

### 4. `.env` файлы

Популярны для локальной разработки. Хранят секреты, которые **не коммитятся в Git**:

```
# .env — НЕ коммитить! Добавить в .gitignore
DB_PASSWORD=secret123
API_TOKEN=eyJhbGciOiJIUzI1NiJ9...
ADMIN_PASSWORD=admin_secret
```

```
# .env.example — коммитить как шаблон
DB_PASSWORD=your_password_here
API_TOKEN=your_token_here
ADMIN_PASSWORD=your_admin_password_here
```

Для чтения `.env` в Java можно использовать библиотеку `dotenv-java`:

```xml
<dependency>
    <groupId>io.github.cdimascio</groupId>
    <artifactId>dotenv-java</artifactId>
    <version>3.0.0</version>
    <scope>test</scope>
</dependency>
```

```java
import io.github.cdimascio.dotenv.Dotenv;

Dotenv dotenv = Dotenv.configure()
        .ignoreIfMissing()  // Не падать, если файла нет (CI/CD)
        .load();

String dbPassword = dotenv.get("DB_PASSWORD");
```

---

## Библиотека Owner

**Owner** — элегантная библиотека для управления конфигурацией через Java-интерфейсы. Это стандарт де-факто
в тестовых фреймворках на Java.

### Подключение

```xml
<dependency>
    <groupId>org.aeonbits.owner</groupId>
    <artifactId>owner</artifactId>
    <version>1.0.12</version>
    <scope>test</scope>
</dependency>
```

### Базовое использование

Создаём интерфейс, описывающий конфигурацию:

```java
package com.company.project.config;

import org.aeonbits.owner.Config;
import org.aeonbits.owner.Config.Sources;
import org.aeonbits.owner.Config.LoadPolicy;
import org.aeonbits.owner.Config.LoadType;

@LoadPolicy(LoadType.MERGE)
@Sources({
    "system:properties",                          // 1-й приоритет: -D параметры
    "system:env",                                  // 2-й приоритет: переменные окружения
    "classpath:config/${env}.properties",          // 3-й приоритет: файл окружения
    "classpath:config/application.properties"      // 4-й приоритет: общий файл
})
public interface ProjectConfig extends Config {

    // --- Браузер ---

    @Key("browser")
    @DefaultValue("chrome")
    String browser();

    @Key("browser.size")
    @DefaultValue("1920x1080")
    String browserSize();

    @Key("browser.headless")
    @DefaultValue("false")
    boolean headless();

    // --- URL ---

    @Key("base.url")
    String baseUrl();

    @Key("api.base.url")
    String apiBaseUrl();

    // --- Таймауты ---

    @Key("timeout")
    @DefaultValue("10000")
    int timeout();

    @Key("polling.interval")
    @DefaultValue("500")
    int pollingInterval();

    // --- База данных ---

    @Key("db.url")
    String dbUrl();

    @Key("db.username")
    String dbUsername();

    @Key("db.password")
    String dbPassword();

    // --- Пользователи ---

    @Key("default.username")
    @DefaultValue("testuser")
    String defaultUsername();

    @Key("default.password")
    String defaultPassword();

    // --- Окружение ---

    @Key("env")
    @DefaultValue("dev")
    String env();
}
```

### Создание экземпляра конфигурации (Singleton)

```java
package com.company.project.config;

import org.aeonbits.owner.ConfigFactory;

// Единая точка доступа к конфигурации
public class ConfigProvider {

    private static volatile ProjectConfig config;

    // Thread-safe Singleton через double-checked locking
    public static ProjectConfig getConfig() {
        if (config == null) {
            synchronized (ConfigProvider.class) {
                if (config == null) {
                    config = ConfigFactory.create(
                        ProjectConfig.class,
                        System.getProperties(),
                        System.getenv()
                    );
                }
            }
        }
        return config;
    }

    // Запрет создания экземпляров
    private ConfigProvider() {
        throw new UnsupportedOperationException(
            "Утилитарный класс, не создавайте экземпляры");
    }
}
```

### Использование в тестах

```java
public class BaseUiTest {

    protected static final ProjectConfig CONFIG = ConfigProvider.getConfig();

    @BeforeEach
    void setup() {
        Configuration.baseUrl = CONFIG.baseUrl();
        Configuration.browser = CONFIG.browser();
        Configuration.browserSize = CONFIG.browserSize();
        Configuration.timeout = CONFIG.timeout();
        Configuration.headless = CONFIG.headless();
    }
}
```

---

## Иерархия и приоритеты конфигурации

Порядок приоритетов (от высшего к низшему):

```
1. System Properties (-Dkey=value)      ← Самый высокий приоритет
2. Environment Variables                  ← CI/CD секреты
3. .env файл                             ← Локальные секреты
4. {env}.properties                       ← Настройки окружения
5. application.properties                 ← Общие настройки
6. @DefaultValue в интерфейсе             ← Значения по умолчанию
```

Это значит, что `mvn test -Dbrowser=firefox` **перезапишет** значение из `application.properties`,
даже если там указано `browser=chrome`.

### Переключение окружений

```bash
# Запуск на dev (по умолчанию)
mvn test

# Запуск на staging
mvn test -Denv=staging

# Запуск на prod с headless Chrome
mvn test -Denv=prod -Dbrowser.headless=true

# CI/CD — всё через переменные окружения
export ENV=staging
export BASE_URL=https://staging.myshop.com
mvn test
```

### Профили Maven

Альтернативный способ переключения окружений через Maven profiles:

```xml
<profiles>
    <profile>
        <id>dev</id>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
        <properties>
            <env>dev</env>
        </properties>
    </profile>
    <profile>
        <id>staging</id>
        <properties>
            <env>staging</env>
        </properties>
    </profile>
    <profile>
        <id>prod</id>
        <properties>
            <env>prod</env>
        </properties>
    </profile>
</profiles>

<!-- Передача свойства в Surefire -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <systemPropertyVariables>
            <env>${env}</env>
        </systemPropertyVariables>
    </configuration>
</plugin>
```

```bash
# Запуск с профилем staging
mvn test -Pstaging
```

---

## Продвинутые возможности Owner

### Конвертация типов

```java
public interface ProjectConfig extends Config {

    // Owner автоматически конвертирует строки в нужные типы
    @Key("timeout")
    int timeout();                    // "10000" → 10000

    @Key("browser.headless")
    boolean headless();               // "true" → true

    @Key("retry.count")
    long retryCount();                // "3" → 3L

    // Список значений
    @Key("supported.browsers")
    @Separator(",")
    List<String> supportedBrowsers(); // "chrome,firefox,edge" → List

    // Enum
    @Key("log.level")
    LogLevel logLevel();              // "INFO" → LogLevel.INFO
}
```

### Переменные в значениях

```properties
# В property-файле можно ссылаться на другие значения
base.url=https://${env}.myshop.com
api.base.url=https://api.${env}.myshop.com
```

### Горячая перезагрузка (Hot Reload)

```java
@HotReload(value = 5, unit = TimeUnit.SECONDS)
@Sources("file:config/application.properties")
public interface ProjectConfig extends Config {
    // Конфигурация автоматически перечитывается каждые 5 секунд
}
```

---

## Безопасность конфигурации

### Что НЕ должно попадать в Git

```gitignore
# .gitignore
.env
**/secrets.properties
**/credentials.properties
*.key
*.pem
```

### Хранение секретов

| Среда | Способ хранения |
|-------|----------------|
| Локально | `.env` файл (в `.gitignore`) |
| CI/CD | Encrypted Secrets (GitHub Actions, GitLab CI) |
| Enterprise | Vault (HashiCorp), AWS Secrets Manager |

### Маскирование в логах

```java
// Никогда не логируйте пароли и токены
log.info("Подключение к БД: url={}, user={}", config.dbUrl(), config.dbUsername());
// НЕ логировать: config.dbPassword(), config.apiToken()
```

---

## Полный пример конфигурации

### Файловая структура

```
src/test/resources/
├── config/
│   ├── application.properties
│   ├── dev.properties
│   ├── staging.properties
│   └── prod.properties
├── allure.properties
└── junit-platform.properties
```

### application.properties

```properties
# Браузер
browser=chrome
browser.size=1920x1080
browser.headless=false

# Таймауты
timeout=10000
polling.interval=500
page.load.timeout=30000

# Повторные попытки
retry.count=2

# Отчёты
screenshots.on.failure=true
video.on.failure=false

# Логирование
log.level=INFO
```

### dev.properties

```properties
# Окружение
env=dev
base.url=https://dev.myshop.com
api.base.url=https://api.dev.myshop.com

# БД
db.url=jdbc:postgresql://localhost:5432/shop_dev
db.username=dev_user
db.password=${DB_PASSWORD}

# Пользователи
default.username=testuser@dev.com
default.password=${DEFAULT_PASSWORD}
admin.username=admin@dev.com
admin.password=${ADMIN_PASSWORD}
```

---

## Связь с тестированием

Правильное управление конфигурацией критично для:

- **Кросс-окружения** — один и тот же набор тестов запускается на dev, staging, prod.
- **CI/CD пайплайнов** — тесты параметризуются через переменные без перекомпиляции.
- **Безопасности** — секреты не попадают в репозиторий.
- **Командной работы** — каждый использует свои локальные настройки без конфликтов.
- **Стабильности** — правильные таймауты и URL для каждого окружения.
- **Отладки** — включение headless/headed режима одним параметром.

---

## Типичные ошибки

1. **Хардкод URL и паролей** — невозможно переключить окружение без изменения кода.
2. **Секреты в Git** — пароли и токены доступны всем, кто имеет доступ к репозиторию.
3. **Нет `.env.example`** — новый участник не знает, какие переменные нужно задать.
4. **Нет значений по умолчанию** — `NullPointerException` при отсутствии значения.
5. **Одинаковые настройки для всех окружений** — таймауты dev не подходят для prod.
6. **Нет валидации конфигурации** — ошибка в URL обнаруживается только при падении теста.
7. **Использование `new Properties()` везде** — вместо единого ConfigProvider.
8. **Не используют `@DefaultValue`** — тесты не запускаются без полной конфигурации.

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Зачем выносить конфигурацию в отдельные файлы?
2. Как прочитать `.properties` файл в Java?
3. Чем `System.getProperty()` отличается от `System.getenv()`?
4. Как передать параметр при запуске Maven-тестов?
5. Что такое `.env` файл и почему его нельзя коммитить?

### 🟡 Средний уровень
6. Что такое библиотека Owner? Какие преимущества она даёт?
7. Как организовать иерархию конфигурации с приоритетами?
8. Как переключать окружения (dev/staging/prod) при запуске тестов?
9. Как безопасно хранить секреты в CI/CD?
10. Что такое `@LoadPolicy(LoadType.MERGE)` в Owner?

### 🔴 Продвинутый уровень
11. Как реализовать валидацию конфигурации при старте тестов?
12. Как организовать конфигурацию для multi-module проекта?
13. Как реализовать hot reload конфигурации для длительных тестовых прогонов?
14. Как интегрировать HashiCorp Vault для хранения секретов в тестовом фреймворке?
15. Как управлять конфигурацией при запуске тестов в Kubernetes/Docker?

---

## Практические задания

### Задание 1: Настройка Owner
Создайте проект с Owner-интерфейсом для конфигурации тестов. Реализуйте:
- Чтение из property-файлов
- Переопределение через системные свойства
- Значения по умолчанию
- Переключение окружений через `-Denv=...`

### Задание 2: Безопасность
Организуйте безопасное хранение секретов:
- Создайте `.env` файл для локальных секретов
- Добавьте `.env.example` с описанием необходимых переменных
- Настройте `.gitignore`
- Продемонстрируйте, что секреты из `.env` доступны в тестах

### Задание 3: CI/CD конфигурация
Напишите GitHub Actions workflow, который:
- Запускает тесты с разными окружениями (dev, staging)
- Передаёт секреты через GitHub Secrets
- Позволяет выбрать браузер через `workflow_dispatch` input

### Задание 4: Валидация
Реализуйте проверку конфигурации при старте тестов: если обязательные значения отсутствуют
или некорректны, тесты не должны запускаться, а в лог выводится понятное сообщение.

---

## Дополнительные ресурсы

- [Owner Library — Official Documentation](http://owner.aeonbits.org/)
- [12-Factor App — Config](https://12factor.net/config)
- [GitHub Actions — Encrypted Secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
- [Maven — Resource Filtering](https://maven.apache.org/plugins/maven-resources-plugin/examples/filter.html)
- [Baeldung — Owner Library](https://www.baeldung.com/java-owner-library)
