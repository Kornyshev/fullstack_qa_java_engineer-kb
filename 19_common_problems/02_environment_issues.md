# Проблемы окружений: нестабильные среды тестирования

## Обзор

Тестовое окружение (test environment) — это инфраструктура, в которой запускаются тесты:
серверы приложений, базы данных, внешние сервисы, сетевая конфигурация. Проблемы с окружениями —
вторая по частоте причина ложных падений тестов после flaky-тестов.

По статистике, QA-инженеры тратят до 30-40% рабочего времени на борьбу с проблемами окружений:
недоступность стендов, рассинхронизация данных, нехватка ресурсов, дрейф конфигураций.
Понимание типичных проблем и умение их решать — критически важный навык, который часто
проверяют на интервью для Middle и Senior QA-позиций.

---

## Типичная архитектура тестовых окружений

```
┌─────────────────────────────────────────────────────────────────┐
│                    Lifecycle окружений                          │
│                                                                 │
│  DEV → QA/TEST → STAGING → PRE-PROD → PRODUCTION               │
│                                                                 │
│  Разработка  Тестирование  Интеграция  Приёмка    Продакшен    │
│  и отладка   (ручное/авто) с внешними  (UAT)      (реальные    │
│                             системами              пользователи)│
└─────────────────────────────────────────────────────────────────┘
```

| Окружение | Назначение | Типичные проблемы |
|-----------|------------|-------------------|
| DEV | Разработка и дебаг | Нестабильно, часто ломается |
| QA/TEST | Автотесты и ручное тестирование | Устаревшие данные, нехватка ресурсов |
| STAGING | Интеграция с внешними системами | Несоответствие конфигурации с продом |
| PRE-PROD | Приёмочное тестирование | Ограниченный доступ, медленный деплой |

---

## Основные проблемы окружений

### 1. Unstable Test Environments (нестабильные стенды)

Тестовые стенды периодически «падают» или работают с ошибками, не связанными с тестируемым
функционалом.

**Симптомы:**
- Тесты массово падают с `Connection refused` или `500 Internal Server Error`
- Приложение на стенде зависает и не отвечает на запросы
- Микросервисы частично недоступны (один из 10 сервисов лежит)

**Причины:**
- Недостаточно ресурсов (CPU, RAM, disk space)
- Сервис упал и не был автоматически перезапущен
- Deployment нового билда сломал стенд
- Утечки памяти в приложении — стенд «деградирует» за несколько дней

```java
/**
 * Health Check — проверка доступности окружения перед запуском тестов.
 * Выполняется как JUnit 5 Extension.
 */
public class EnvironmentHealthCheck implements BeforeAllCallback {

    private static final List<String> REQUIRED_SERVICES = List.of(
        "http://test-env:8080/actuator/health",    // Основное приложение
        "http://test-env:8081/actuator/health",    // Сервис авторизации
        "http://test-env:5432"                      // База данных
    );

    @Override
    public void beforeAll(ExtensionContext context) {
        List<String> unavailable = REQUIRED_SERVICES.stream()
            .filter(url -> !isServiceHealthy(url))
            .collect(Collectors.toList());

        if (!unavailable.isEmpty()) {
            throw new TestAbortedException(
                "Окружение недоступно. Недоступные сервисы: " + unavailable
            );
        }
    }

    private boolean isServiceHealthy(String url) {
        try {
            HttpClient client = HttpClient.newBuilder()
                .connectTimeout(Duration.ofSeconds(5))
                .build();
            HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(url))
                .timeout(Duration.ofSeconds(5))
                .GET()
                .build();
            HttpResponse<String> response = client.send(
                request, HttpResponse.BodyHandlers.ofString()
            );
            return response.statusCode() == 200;
        } catch (Exception e) {
            return false;
        }
    }
}

// Использование
@ExtendWith(EnvironmentHealthCheck.class)
class ApiTests {
    @Test
    void testSomething() {
        // Тест запустится только если окружение доступно
    }
}
```

### 2. Data Sync Issues (рассинхронизация данных)

Данные между окружениями рассинхронизированы: тесты ожидают определённые данные,
а на стенде их нет или они отличаются.

**Типичные сценарии:**
- Справочники (catalogs) обновлены в продакшене, но не в тестовом окружении
- Миграция БД применена в DEV, но не в QA
- Тестовые данные были удалены при обновлении стенда
- Fixture-данные устарели и не соответствуют текущей схеме

```java
/**
 * Утилита для проверки и подготовки тестовых данных.
 * Выполняется перед test suite, чтобы гарантировать наличие необходимых данных.
 */
public class TestDataValidator {

    private final JdbcTemplate jdbcTemplate;

    public TestDataValidator(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    // Проверка наличия обязательных справочных данных
    public void validateReferenceData() {
        List<String> missingTables = new ArrayList<>();

        // Проверяем, что справочники заполнены
        Map<String, Integer> expectedCounts = Map.of(
            "currencies", 3,       // Минимум 3 валюты
            "countries", 50,       // Минимум 50 стран
            "product_categories", 5 // Минимум 5 категорий
        );

        expectedCounts.forEach((table, minCount) -> {
            Integer actual = jdbcTemplate.queryForObject(
                "SELECT COUNT(*) FROM " + table, Integer.class
            );
            if (actual == null || actual < minCount) {
                missingTables.add(
                    String.format("%s: ожидается >= %d, фактически %d",
                        table, minCount, actual)
                );
            }
        });

        if (!missingTables.isEmpty()) {
            throw new IllegalStateException(
                "Отсутствуют справочные данные:\n" +
                String.join("\n", missingTables)
            );
        }
    }

    // Подготовка тестовых данных из SQL-скрипта
    public void seedTestData(String scriptPath) {
        try {
            String sql = Files.readString(Path.of(scriptPath));
            jdbcTemplate.execute(sql);
        } catch (Exception e) {
            throw new RuntimeException(
                "Не удалось загрузить тестовые данные: " + scriptPath, e
            );
        }
    }
}
```

### 3. VPN и Access Problems (проблемы с доступом)

Тестовые окружения часто находятся за VPN или корпоративным файрволом, что создаёт проблемы
для CI/CD и удалённой работы.

**Типичные проблемы:**
- CI-сервер не имеет доступа к тестовому стенду
- VPN-соединение обрывается во время длительного test run
- Сертификаты устарели — HTTPS-подключения отклоняются
- Доступ по IP ограничен, а CI-сервер использует динамические IP

**Решения:**
- Разместить CI-runner в той же сети, что и тестовое окружение
- Настроить VPN на уровне CI-сервера (а не пользователя)
- Автоматическое обновление сертификатов
- Использовать service mesh (Istio, Linkerd) для управления сетевым доступом

```java
// Конфигурация RestAssured для работы с самоподписанными сертификатами
// (типичная ситуация для тестовых окружений)
public class ApiClientConfig {

    public static RequestSpecification getBaseSpec() {
        return new RequestSpecBuilder()
            .setBaseUri(Config.get("base.url"))
            .setContentType(ContentType.JSON)
            // Разрешить самоподписанные сертификаты (ТОЛЬКО для тестов!)
            .setRelaxedHTTPSValidation()
            // Таймауты для нестабильных сетей
            .setConfig(RestAssuredConfig.config()
                .httpClient(HttpClientConfig.httpClientConfig()
                    .setParam("http.connection.timeout", 10000)
                    .setParam("http.socket.timeout", 30000)
                )
            )
            .build();
    }
}
```

### 4. Resource Shortage (нехватка ресурсов)

Тестовые окружения часто получают меньше ресурсов, чем продакшен, что приводит
к таймаутам, медленной работе и ложным падениям.

**Признаки нехватки ресурсов:**

| Ресурс | Симптомы | Диагностика |
|--------|----------|-------------|
| CPU | Медленные ответы, таймауты | `top`, `htop`, мониторинг |
| RAM | OOM-ошибки, свопинг | `free -m`, метрики JVM heap |
| Disk | Ошибки записи, потеря логов | `df -h`, алерты на заполнение |
| Network | Медленные API-вызовы | `ping`, `traceroute`, latency |
| DB connections | Pool exhausted | Мониторинг connection pool |

```java
/**
 * Утилита для проверки ресурсов тестового окружения.
 * Запускается перед test suite для раннего выявления проблем.
 */
public class ResourceChecker {

    // Проверка доступного дискового пространства
    public static void checkDiskSpace(Path path, long minFreeMB) {
        long freeSpace = path.toFile().getFreeSpace() / (1024 * 1024);
        if (freeSpace < minFreeMB) {
            throw new TestAbortedException(
                String.format("Недостаточно дискового пространства: %d MB (нужно %d MB)",
                    freeSpace, minFreeMB)
            );
        }
    }

    // Проверка доступной памяти JVM
    public static void checkJvmMemory(long minFreeMB) {
        Runtime runtime = Runtime.getRuntime();
        long freeMemory = runtime.freeMemory() / (1024 * 1024);
        long maxMemory = runtime.maxMemory() / (1024 * 1024);
        long usedMemory = (runtime.totalMemory() - runtime.freeMemory()) / (1024 * 1024);

        if (freeMemory < minFreeMB) {
            System.out.printf(
                "ПРЕДУПРЕЖДЕНИЕ: JVM — используется %d MB, свободно %d MB, максимум %d MB%n",
                usedMemory, freeMemory, maxMemory
            );
        }
    }
}
```

### 5. Environment Drift (дрейф окружений)

Со временем конфигурация тестового окружения расходится с продакшеном. Версии ОС, библиотек,
настройки Java, конфигурации сервисов начинают отличаться, и тесты перестают отражать
реальное поведение системы.

**Примеры дрейфа:**
- В продакшене Java 17.0.10, на тестовом стенде — Java 17.0.2
- В продакшене nginx с custom-конфигурацией, на стенде — стандартная
- В продакшене Redis cluster, на стенде — standalone Redis
- В продакшене PostgreSQL 16, на стенде — PostgreSQL 14

**Стратегии предотвращения:**
- Infrastructure as Code (IaC): Terraform, Ansible
- Единые Docker-образы для всех окружений
- Регулярный аудит конфигураций
- Configuration Management (Consul, etcd)

### 6. Environment Provisioning (развёртывание окружений)

Создание и обновление тестовых окружений — трудоёмкий процесс, который часто занимает
дни или недели.

**Проблемы:**
- Ручное создание окружений — долго и подвержено ошибкам
- Зависимость от DevOps-команды — очередь на настройку стендов
- Нет документации по развёртыванию — «только Вася знает, как настроить»
- Невозможно быстро откатить сломанное окружение

---

## Docker-based решения

Docker радикально упрощает работу с тестовыми окружениями, обеспечивая воспроизводимость,
изоляцию и быстрое развёртывание.

### Docker Compose для тестового окружения

```yaml
# docker-compose.test.yml — окружение для интеграционных тестов
version: '3.8'

services:
  # Тестируемое приложение
  app:
    image: my-app:${APP_VERSION:-latest}
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=test
      - DB_URL=jdbc:postgresql://db:5432/testdb
      - DB_USERNAME=testuser
      - DB_PASSWORD=testpass
      - REDIS_HOST=redis
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 10s
      timeout: 5s
      retries: 5

  # База данных
  db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_DB=testdb
      - POSTGRES_USER=testuser
      - POSTGRES_PASSWORD=testpass
    volumes:
      - ./sql/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U testuser -d testdb"]
      interval: 5s
      timeout: 3s
      retries: 5

  # Кэш
  redis:
    image: redis:7-alpine

  # Мок внешних сервисов
  wiremock:
    image: wiremock/wiremock:3.3.1
    ports:
      - "8089:8080"
    volumes:
      - ./wiremock/mappings:/home/wiremock/mappings
      - ./wiremock/__files:/home/wiremock/__files
```

### Testcontainers — программное управление контейнерами из тестов

```java
/**
 * Базовый класс для интеграционных тестов с Testcontainers.
 * Каждый test suite получает свой набор контейнеров — полная изоляция.
 */
@Testcontainers
public abstract class BaseIntegrationTest {

    // PostgreSQL контейнер — поднимается автоматически
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test")
        .withInitScript("db/init.sql");

    // Redis контейнер
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);

    // WireMock для мокирования внешних API
    @Container
    static WireMockContainer wiremock = new WireMockContainer("wiremock/wiremock:3.3.1")
        .withMappingFromResource("payment-api-stub.json");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        // Динамические свойства — порты и хосты контейнеров
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.redis.host", redis::getHost);
        registry.add("spring.redis.port", () -> redis.getMappedPort(6379));
        registry.add("payment.api.url", () ->
            "http://" + wiremock.getHost() + ":" + wiremock.getMappedPort(8080)
        );
    }
}

// Конкретный тест
class PaymentServiceTest extends BaseIntegrationTest {

    @Autowired
    private PaymentService paymentService;

    @Test
    void shouldProcessPaymentSuccessfully() {
        // Контейнеры уже запущены, данные изолированы
        PaymentResult result = paymentService.processPayment(
            new PaymentRequest("100.00", "USD")
        );
        assertEquals(PaymentStatus.SUCCESS, result.getStatus());
    }
}
```

### Ephemeral Environments (эфемерные окружения)

Создание отдельного окружения на каждый Pull Request:

```yaml
# GitHub Actions — создание окружения для PR
name: PR Environment
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  deploy-pr-env:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Развернуть окружение для PR
        run: |
          docker compose -f docker-compose.test.yml up -d
          # Ожидание готовности
          ./scripts/wait-for-services.sh

      - name: Запуск тестов
        run: mvn test -Denv.url=http://localhost:8080

      - name: Остановить окружение
        if: always()
        run: docker compose -f docker-compose.test.yml down -v
```

---

## Как отстаивать стабильные тестовые окружения

### Аргументация для менеджмента

Нестабильные окружения стоят компании денег. Вот как это обосновать:

| Проблема | Стоимость |
|----------|-----------|
| QA-инженер тратит 2 ч/день на проблемы окружений | ~25% зарплаты |
| Задержка релиза из-за недоступного стенда | Упущенная прибыль |
| Баг пропущен из-за того, что тесты не запустились | Стоимость бага в продакшене (x10-x100) |
| Перезапуск pipeline из-за нестабильного стенда | Время разработчиков × стоимость CI |

### Чек-лист стабильного тестового окружения

```
✅ Автоматическое развёртывание (IaC или Docker Compose)
✅ Health Check endpoint для каждого сервиса
✅ Мониторинг и алертинг (CPU, RAM, disk, доступность)
✅ Автоматический рестарт упавших сервисов
✅ Регулярное обновление тестовых данных
✅ Документация по архитектуре окружения
✅ Backup и быстрый restore
✅ Логирование доступно QA-команде (не только DevOps)
✅ Изоляция от других команд (отдельный namespace/стенд)
✅ SLA на доступность (например, 99% в рабочее время)
```

### Метрики для обоснования инвестиций

```java
/**
 * Сбор метрик доступности тестового окружения.
 * Данные используются для обоснования инвестиций в инфраструктуру.
 */
public class EnvironmentMetrics {

    // Расчёт uptime окружения за период
    public double calculateUptime(
            List<HealthCheckResult> checks, Instant from, Instant to) {
        long totalChecks = checks.stream()
            .filter(c -> c.getTimestamp().isAfter(from) && c.getTimestamp().isBefore(to))
            .count();
        long successChecks = checks.stream()
            .filter(c -> c.getTimestamp().isAfter(from) && c.getTimestamp().isBefore(to))
            .filter(HealthCheckResult::isHealthy)
            .count();

        return totalChecks > 0 ? (double) successChecks / totalChecks * 100 : 0;
    }

    // Расчёт потерянного времени из-за недоступности
    public Duration calculateLostTime(List<TestRunResult> runs) {
        return runs.stream()
            .filter(r -> r.getFailureReason() == FailureReason.ENVIRONMENT_UNAVAILABLE)
            .map(TestRunResult::getDuration)
            .reduce(Duration.ZERO, Duration::plus);
    }
}
```

---

## Управление конфигурациями окружений

### Externalized Configuration

```java
/**
 * Конфигурация тестового окружения.
 * Все параметры вынесены из кода — переключение между окружениями без перекомпиляции.
 */
public class EnvironmentConfig {

    private static final Properties props = new Properties();

    static {
        // Определяем окружение из системной переменной (по умолчанию — test)
        String env = System.getProperty("test.env", "test");
        String configFile = String.format("config/%s.properties", env);
        try (InputStream is = EnvironmentConfig.class
                .getClassLoader().getResourceAsStream(configFile)) {
            if (is != null) {
                props.load(is);
            } else {
                throw new RuntimeException("Файл конфигурации не найден: " + configFile);
            }
        } catch (IOException e) {
            throw new RuntimeException("Ошибка загрузки конфигурации", e);
        }
    }

    public static String getBaseUrl() {
        return props.getProperty("base.url");
    }

    public static String getDbUrl() {
        return props.getProperty("db.url");
    }

    public static int getTimeout() {
        return Integer.parseInt(props.getProperty("timeout.seconds", "30"));
    }
}
```

Файлы конфигурации для разных окружений:

```properties
# config/test.properties
base.url=http://test-server:8080
db.url=jdbc:postgresql://test-db:5432/testdb
timeout.seconds=30

# config/staging.properties
base.url=https://staging.example.com
db.url=jdbc:postgresql://staging-db:5432/stagingdb
timeout.seconds=60

# config/local.properties
base.url=http://localhost:8080
db.url=jdbc:postgresql://localhost:5432/localdb
timeout.seconds=10
```

Запуск тестов на разных окружениях:

```bash
# Запуск на тестовом стенде
mvn test -Dtest.env=test

# Запуск на staging
mvn test -Dtest.env=staging

# Запуск локально
mvn test -Dtest.env=local
```

---

## Связь с тестированием

Проблемы окружений напрямую влияют на все аспекты тестирования:

- **Достоверность результатов** — если окружение нестабильно, нельзя доверять ни PASS, ни FAIL
- **Скорость обратной связи** — недоступный стенд блокирует весь CI/CD pipeline
- **Покрытие тестами** — часть тестов невозможно запустить из-за отсутствия нужных сервисов
- **Репутация QA** — «тесты опять красные» → команда перестаёт обращать внимание
- **Стоимость тестирования** — простой из-за окружений значительно увеличивает стоимость QA

---

## Типичные ошибки

1. **Не проверять доступность окружения перед тестами** — тесты падают с непонятными ошибками
2. **Использовать одно окружение для нескольких команд** — конфликты данных и конфигураций
3. **Не документировать архитектуру окружения** — зависимость от «знающих людей»
4. **Хардкодить URL-ы и параметры в тестах** — переключение между стендами невозможно
5. **Игнорировать мониторинг тестовых стендов** — «это же не прод, зачем мониторить»
6. **Не использовать Docker/Testcontainers** — упускать возможность полной изоляции
7. **Полагаться на ручное развёртывание** — человеческий фактор порождает ошибки
8. **Не иметь SLA на доступность тестового окружения** — нет формальных обязательств

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Какие типы тестовых окружений вы знаете? Чем они отличаются?
2. Что такое Health Check и зачем он нужен в тестировании?
3. Как вы переключаетесь между разными тестовыми окружениями?
4. Зачем нужны отдельные окружения для тестирования?

### 🟡 Средний уровень
5. Что такое Testcontainers? Какие преимущества они дают?
6. Как организовать изоляцию тестовых данных между параллельными тестами?
7. Как вы диагностируете, что падение теста связано с проблемой окружения, а не с багом?
8. Что такое Environment Drift и как с ним бороться?
9. Как настроить тестовый framework для работы с несколькими окружениями?

### 🔴 Продвинутый уровень
10. Как организовать ephemeral environments для PR-based testing?
11. Как обосновать руководству необходимость инвестиций в стабильные тестовые окружения?
12. Опишите архитектуру тестового окружения для микросервисного приложения с 15 сервисами.
13. Как организовать мониторинг и алертинг для тестовых окружений?
14. Как решить проблему тестирования интеграций с внешними (third-party) сервисами?

---

## Практические задания

### Задание 1: Health Check Extension
Реализуйте JUnit 5 Extension, который:
- Перед test suite проверяет доступность всех необходимых сервисов
- При недоступности — пропускает тесты (не FAIL, а SKIP) с информативным сообщением
- Логирует результаты проверки в Allure-отчёт
- Поддерживает конфигурацию через properties-файл

### Задание 2: Docker Compose
Напишите `docker-compose.yml` для тестового окружения, включающего:
- Spring Boot приложение
- PostgreSQL с предзаполненными тестовыми данными
- Redis
- WireMock для мокирования внешнего платёжного API
- Selenoid/Selenium Grid для UI-тестов

### Задание 3: Testcontainers
Создайте базовый класс для интеграционных тестов с использованием Testcontainers:
- PostgreSQL с миграциями (Flyway/Liquibase)
- Kafka для тестирования асинхронных событий
- WireMock для мокирования внешних HTTP-сервисов
- Возможность переключения между Testcontainers (для локальной разработки)
  и реальным окружением (для CI)

### Задание 4: Метрики окружений
Спроектируйте систему сбора метрик доступности тестовых окружений:
- Периодический health check (каждые 5 минут)
- Расчёт uptime за неделю/месяц
- Корреляция с результатами тестов (какой процент падений — из-за окружений?)
- Автоматическая нотификация при деградации

---

## Дополнительные ресурсы

- [Testcontainers Documentation](https://www.testcontainers.org/) — официальная документация по Testcontainers
- [Docker Compose Specification](https://docs.docker.com/compose/compose-file/) — спецификация Docker Compose
- [Spring Boot: Externalized Configuration](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config) — управление конфигурациями
- [WireMock Documentation](https://wiremock.org/docs/) — мокирование HTTP-сервисов
- [Terraform: Infrastructure as Code](https://www.terraform.io/) — управление инфраструктурой как кодом
- Martin Fowler: "TestDouble" — паттерны подмены зависимостей в тестах
