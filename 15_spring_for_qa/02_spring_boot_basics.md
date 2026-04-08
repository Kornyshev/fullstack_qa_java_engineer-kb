# Spring Boot для QA

## Обзор

Spring Boot — это надстройка над Spring Framework, которая радикально упрощает создание и конфигурацию
приложений. Для QA-инженера Spring Boot важен по нескольким причинам: он определяет, как приложение
конфигурируется (properties, profiles), как запускается (embedded server), и предоставляет встроенные
инструменты мониторинга (Actuator), которые полезны при тестировании. Понимание Spring Boot позволяет
QA быстрее разворачивать тестовые окружения, диагностировать проблемы и писать эффективные интеграционные
тесты.

---

## Что добавляет Spring Boot поверх Spring

### Spring vs Spring Boot

| Аспект | Spring Framework | Spring Boot |
|--------|-----------------|-------------|
| Конфигурация | Ручная, многословная (XML / Java Config) | Автоматическая (auto-configuration) |
| Зависимости | Вручную подбираете совместимые версии | Starters — готовые наборы зависимостей |
| Сервер | Внешний (Tomcat, Jetty нужно настраивать) | Встроенный (embedded Tomcat/Jetty/Undertow) |
| Запуск | WAR-файл → деплой на сервер | JAR-файл → `java -jar app.jar` |
| Properties | Базовая поддержка | Мощная система с профилями и приоритетами |
| Мониторинг | Нет из коробки | Actuator — health, metrics, info |

### Ключевая философия: Convention over Configuration

Spring Boot следует принципу «соглашение важнее конфигурации» — большинство настроек имеют
разумные значения по умолчанию. Для QA это значит, что приложение может работать с минимальной
конфигурацией, а отклонения от стандарта находятся в `application.properties` / `application.yml`.

---

## Starters — готовые наборы зависимостей

Starters — это Maven/Gradle-зависимости, которые подтягивают всё необходимое для конкретной
функциональности.

### Основные starters, которые QA встречает в проектах

| Starter | Что включает | Значение для QA |
|---------|-------------|-----------------|
| `spring-boot-starter-web` | Spring MVC, embedded Tomcat, Jackson | REST API — основной объект тестирования |
| `spring-boot-starter-data-jpa` | Hibernate, Spring Data JPA | Работа с БД — нужны тестовые данные |
| `spring-boot-starter-test` | JUnit 5, Mockito, MockMvc, AssertJ | Основной набор для написания тестов |
| `spring-boot-starter-security` | Spring Security, аутентификация | Тестирование авторизации и ролей |
| `spring-boot-starter-actuator` | Health checks, metrics, info | Мониторинг приложения при тестировании |
| `spring-boot-starter-validation` | Bean Validation (Hibernate Validator) | Валидация входных данных |

### Пример: что подтягивает spring-boot-starter-test

```xml
<!-- Один starter вместо 5+ отдельных зависимостей -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

Автоматически включает:
- **JUnit 5** (Jupiter) — запуск тестов
- **Mockito** — моки и стабы
- **AssertJ** — fluent-assertions
- **Spring Test** — `@SpringBootTest`, `MockMvc`
- **JSONassert** — проверка JSON
- **JsonPath** — навигация по JSON
- **Hamcrest** — дополнительные matchers

---

## Auto-Configuration

Auto-configuration — механизм, при котором Spring Boot автоматически настраивает bean на основе
доступных зависимостей и properties.

### Как это работает

```
┌──────────────────────────────────────────────────────┐
│  Classpath содержит spring-boot-starter-data-jpa     │
│              +                                       │
│  application.yml содержит spring.datasource.url      │
│              =                                       │
│  Spring Boot автоматически создаёт:                  │
│    ├── DataSource (подключение к БД)                 │
│    ├── EntityManagerFactory (Hibernate)              │
│    ├── TransactionManager                            │
│    └── JpaRepositories (реализации репозиториев)     │
└──────────────────────────────────────────────────────┘
```

### Почему QA это важно

```yaml
# application.yml — если URL БД неправильный, приложение не стартует
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: app_user
    password: secret
```

> **Для QA**: если приложение не запускается на тестовом стенде — первым делом проверьте
> `application.properties` / `application.yml`. Auto-configuration пытается подключиться
> к указанным ресурсам (БД, Kafka, Redis) при старте.

### Отладка auto-configuration

```properties
# Включить отчёт об auto-configuration в логах
debug=true
```

В логах появится:
```
============================
CONDITIONS EVALUATION REPORT
============================

Positive matches:
-----------------
   DataSourceAutoConfiguration matched

Negative matches:
-----------------
   MongoAutoConfiguration:
      Did not match:
         - @ConditionalOnClass did not find required class 'com.mongodb.client.MongoClient'
```

---

## Конфигурация: application.properties и application.yml

### Два формата — одна цель

```properties
# application.properties — плоский формат
server.port=8080
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
spring.datasource.username=admin
logging.level.root=INFO
```

```yaml
# application.yml — иерархический формат (чаще используется)
server:
  port: 8080

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: admin

logging:
  level:
    root: INFO
```

### Ключевые настройки, которые QA должен знать

| Настройка | Описание | Зачем QA |
|-----------|----------|----------|
| `server.port` | Порт приложения | Знать, куда слать запросы |
| `spring.datasource.url` | URL базы данных | Проверить подключение тестового стенда |
| `spring.jpa.show-sql=true` | Показывать SQL-запросы в логах | Отладка проблем с данными |
| `logging.level.org.springframework=DEBUG` | Уровень логирования Spring | Диагностика проблем |
| `spring.profiles.active` | Активный профиль | Определяет конфигурацию окружения |
| `management.endpoints.web.exposure.include` | Открытые Actuator-эндпоинты | Доступ к мониторингу |

### Приоритеты конфигурации (от высшего к низшему)

```
1. Аргументы командной строки:     java -jar app.jar --server.port=9090
2. Переменные окружения:           SERVER_PORT=9090
3. application-{profile}.yml:      application-prod.yml
4. application.yml:                Базовые настройки
5. Значения по умолчанию:         Spring Boot defaults
```

> **Для QA**: если настройка не работает как ожидается — проверьте, не перекрывается ли
> она на более высоком уровне приоритета (переменная окружения, аргумент командной строки).

### Использование @Value для чтения конфигурации

```java
@Service
public class NotificationService {

    // Значение берётся из application.yml
    @Value("${notification.email.enabled:true}")
    private boolean emailEnabled;

    @Value("${notification.retry.count:3}")
    private int retryCount;

    public void sendNotification(String message) {
        if (!emailEnabled) {
            return; // В тестовом окружении может быть выключено
        }
        // Отправка уведомления с retryCount попытками
    }
}
```

---

## Profiles — управление окружениями

Profiles позволяют иметь разную конфигурацию для разных окружений (dev, test, staging, prod).

### Структура файлов конфигурации

```
src/main/resources/
├── application.yml              # Общие настройки для всех профилей
├── application-dev.yml          # Настройки для разработки
├── application-test.yml         # Настройки для тестирования
├── application-staging.yml      # Настройки для staging
└── application-prod.yml         # Настройки для production
```

### Пример конфигурации по профилям

```yaml
# application.yml — общие настройки
spring:
  application:
    name: order-service

---
# application-dev.yml — разработка
server:
  port: 8080
spring:
  datasource:
    url: jdbc:h2:mem:devdb        # In-memory БД для быстрого старта
  jpa:
    show-sql: true
logging:
  level:
    root: DEBUG

---
# application-test.yml — тестирование
server:
  port: 0                         # Случайный порт — избежать конфликтов
spring:
  datasource:
    url: jdbc:h2:mem:testdb
  jpa:
    hibernate:
      ddl-auto: create-drop       # Пересоздавать схему при каждом запуске
logging:
  level:
    root: WARN                    # Меньше шума в тестах

---
# application-prod.yml — production
server:
  port: 443
spring:
  datasource:
    url: jdbc:postgresql://prod-db:5432/orders
  jpa:
    show-sql: false               # Не логировать SQL в production
logging:
  level:
    root: ERROR
```

### Активация профиля

```bash
# Через переменную окружения
export SPRING_PROFILES_ACTIVE=staging

# Через аргумент командной строки
java -jar app.jar --spring.profiles.active=prod

# Через application.yml
spring:
  profiles:
    active: dev
```

### Профили в тестах

```java
// Активация тестового профиля — использует application-test.yml
@SpringBootTest
@ActiveProfiles("test")
class OrderServiceIntegrationTest {

    @Test
    void shouldCreateOrder() {
        // Тест работает с H2 in-memory, порт случайный
    }
}
```

> **Для QA**: всегда проверяйте, какой профиль активен на тестовом стенде. Баг может
> воспроизводиться только на staging, потому что конфигурация отличается от dev.

---

## Embedded Server

Spring Boot включает встроенный веб-сервер (по умолчанию Tomcat).

### Как это упрощает тестирование

```java
// Spring Boot поднимает настоящий HTTP-сервер на случайном порту
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ApiIntegrationTest {

    @LocalServerPort
    private int port; // Spring подставит реальный порт

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void shouldReturnUsers() {
        ResponseEntity<String> response = restTemplate
                .getForEntity("http://localhost:" + port + "/api/users", String.class);

        assertEquals(HttpStatus.OK, response.getStatusCode());
    }
}
```

### Режимы WebEnvironment

| Режим | Описание | Когда использовать |
|-------|----------|-------------------|
| `MOCK` (по умолчанию) | Без реального сервера, MockMvc | Юнит-тесты контроллеров |
| `RANDOM_PORT` | Реальный сервер на случайном порту | Интеграционные тесты API |
| `DEFINED_PORT` | Реальный сервер на порту из properties | Когда порт важен |
| `NONE` | Без веб-окружения | Тесты сервисов без HTTP |

---

## Spring Boot Actuator

Actuator предоставляет production-ready эндпоинты для мониторинга и управления приложением.
Для QA это незаменимый инструмент при тестировании.

### Основные эндпоинты

| Эндпоинт | URL | Что показывает | Применение для QA |
|----------|-----|----------------|-------------------|
| `health` | `/actuator/health` | Статус приложения и зависимостей | Проверить, что приложение live |
| `info` | `/actuator/info` | Версия, описание приложения | Убедиться, что задеплоена нужная версия |
| `metrics` | `/actuator/metrics` | JVM, HTTP, custom метрики | Мониторинг производительности |
| `env` | `/actuator/env` | Конфигурация, переменные окружения | Диагностика проблем конфигурации |
| `beans` | `/actuator/beans` | Все зарегистрированные bean | Понимание структуры приложения |
| `mappings` | `/actuator/mappings` | Все HTTP-маршруты | Знать все эндпоинты для тестирования |
| `loggers` | `/actuator/loggers` | Уровни логирования | Динамически включить DEBUG |

### Конфигурация Actuator

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,env,mappings  # Какие эндпоинты доступны
      base-path: /actuator                          # Базовый путь (по умолчанию)
  endpoint:
    health:
      show-details: always    # Показывать детали (db, disk, redis)
  info:
    env:
      enabled: true           # Показывать info.* из properties
```

### Пример ответа /actuator/health

```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "PostgreSQL",
        "validationQuery": "isValid()"
      }
    },
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 107374182400,
        "free": 85899345920,
        "threshold": 10485760
      }
    },
    "redis": {
      "status": "DOWN",
      "details": {
        "error": "Connection refused"
      }
    }
  }
}
```

> **Для QA**: `/actuator/health` — первое, что нужно проверить, если приложение ведёт себя
> странно. Если Redis показывает `DOWN` — кэш не работает, и это может объяснить баг.

### Полезные метрики для QA

```bash
# Количество HTTP-запросов
curl http://localhost:8080/actuator/metrics/http.server.requests

# Время ответа
curl http://localhost:8080/actuator/metrics/http.server.requests?tag=uri:/api/users

# Использование памяти JVM
curl http://localhost:8080/actuator/metrics/jvm.memory.used

# Количество активных потоков
curl http://localhost:8080/actuator/metrics/jvm.threads.live
```

### Использование /actuator/mappings для тест-дизайна

Эндпоинт `/actuator/mappings` показывает все зарегистрированные маршруты:

```json
{
  "handler": "com.example.UserController#getUser(Long)",
  "predicate": "{GET /api/users/{id}}",
  "details": {
    "produces": ["application/json"]
  }
}
```

> **Для QA**: используйте `/actuator/mappings`, чтобы увидеть все доступные API-эндпоинты
> приложения. Это помогает составить тест-план и убедиться, что все эндпоинты покрыты тестами.

---

## Связь с тестированием

### Тестовые properties

```yaml
# src/test/resources/application-test.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb         # In-memory БД вместо PostgreSQL
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create-drop         # Чистая схема при каждом запуске

# Отключаем внешние зависимости для изоляции тестов
notification:
  email:
    enabled: false
  sms:
    enabled: false
```

### Testcontainers — реальные зависимости в тестах

```java
// Запускаем настоящий PostgreSQL в Docker для тестов
@SpringBootTest
@ActiveProfiles("test")
@Testcontainers
class OrderRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        // Переопределяем URL БД на контейнер
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void shouldSaveAndFindOrder() {
        Order order = new Order("product-1", 2, BigDecimal.valueOf(99.99));
        Order saved = orderRepository.save(order);

        Optional<Order> found = orderRepository.findById(saved.getId());
        assertTrue(found.isPresent());
        assertEquals("product-1", found.get().getProductId());
    }
}
```

---

## Типичные ошибки

### 1. Неправильный профиль на тестовом стенде

```bash
# Стенд запущен с профилем dev вместо staging
# Результат: приложение подключается к dev-базе, тестовые данные отсутствуют
java -jar app.jar --spring.profiles.active=dev  # Ошибка!
java -jar app.jar --spring.profiles.active=staging  # Правильно
```

### 2. Игнорирование Actuator при диагностике

```bash
# Вместо того чтобы гадать, почему приложение медленное:
curl http://localhost:8080/actuator/health
# Видим: Redis DOWN → кэш не работает → каждый запрос идёт в БД
```

### 3. Жёстко зашитый порт в тестах

```java
// ОШИБКА: порт зашит — тест упадёт, если порт занят
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
class ApiTest {
    // Всегда 8080 — конфликт с другими тестами или приложениями
}

// ПРАВИЛЬНО: случайный порт
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ApiTest {
    @LocalServerPort
    private int port; // Каждый запуск — уникальный порт
}
```

### 4. Тестовый application.yml не в том месте

```
# ОШИБКА: положили в src/main/resources — влияет на production
src/main/resources/application-test.yml   ← Неправильно!

# ПРАВИЛЬНО: тестовые настройки в тестовых ресурсах
src/test/resources/application-test.yml   ← Правильно!
```

### 5. Забыли открыть Actuator-эндпоинты

```yaml
# По умолчанию открыт только /actuator/health
# Чтобы видеть все — нужно явно указать
management:
  endpoints:
    web:
      exposure:
        include: "*"  # Открыть всё (только для dev/test!)
```

---

## Вопросы на интервью

### 🟢 Базовый уровень

1. **Чем Spring Boot отличается от Spring Framework?**
   Spring Boot — надстройка над Spring, добавляющая auto-configuration, starters, embedded server
   и Actuator. Упрощает конфигурацию и запуск приложений.

2. **Что такое starters?**
   Готовые наборы зависимостей. Например, `spring-boot-starter-web` включает Spring MVC,
   embedded Tomcat и Jackson для работы с JSON.

3. **Где находятся настройки Spring Boot приложения?**
   В файлах `application.properties` или `application.yml` в директории `src/main/resources`.

4. **Что такое профили и зачем они нужны?**
   Профили позволяют иметь разные конфигурации для разных окружений (dev, test, staging, prod).
   Активируются через `spring.profiles.active`.

### 🟡 Средний уровень

5. **Как работает auto-configuration?**
   Spring Boot анализирует classpath и properties, автоматически создавая и настраивая bean.
   Если в classpath есть JDBC-драйвер и указан `spring.datasource.url` — DataSource
   создастся автоматически.

6. **Какие Actuator-эндпоинты полезны для QA?**
   `/health` — статус компонентов, `/metrics` — метрики производительности,
   `/mappings` — все HTTP-маршруты, `/env` — конфигурация, `/info` — информация о сборке.

7. **Как переопределить настройки для тестов?**
   Создать `application-test.yml` в `src/test/resources`, использовать `@ActiveProfiles("test")`,
   или `@DynamicPropertySource` для Testcontainers.

8. **В чём разница между RANDOM_PORT и MOCK в @SpringBootTest?**
   `RANDOM_PORT` запускает реальный HTTP-сервер — подходит для интеграционных тестов.
   `MOCK` не запускает сервер — используется с `MockMvc` для тестов контроллеров.

### 🔴 Продвинутый уровень

9. **Как Spring Boot определяет порядок приоритета конфигурации?**
   Command-line args → переменные окружения → profile-specific properties → application.yml →
   defaults. Более специфичный источник перекрывает менее специфичный.

10. **Как настроить Testcontainers с DynamicPropertySource?**
    `@DynamicPropertySource` позволяет переопределить properties после старта контейнера,
    когда известен реальный порт и URL. Это необходимо, потому что контейнер стартует
    на случайном порту.

11. **Как кастомизировать health check для QA-нужд?**
    Реализовать `HealthIndicator` интерфейс с custom-логикой. Например, проверка
    доступности внешнего API или наличия тестовых данных.

---

## Практические задания

### Задание 1: Анализ конфигурации

Посмотрите на `application.yml` и ответьте: на каком порту запустится приложение? Какую БД оно
использует? Какой уровень логирования?

```yaml
server:
  port: 9090

spring:
  profiles:
    active: staging
  datasource:
    url: jdbc:postgresql://localhost:5432/app
```

```yaml
# application-staging.yml
spring:
  datasource:
    url: jdbc:postgresql://staging-db:5432/app_staging
logging:
  level:
    root: WARN
```

### Задание 2: Проверка через Actuator

Напишите набор curl-команд для проверки:
1. Что приложение запущено и все зависимости доступны
2. Какая версия приложения задеплоена
3. Какие API-эндпоинты доступны
4. Сколько памяти потребляет приложение

### Задание 3: Тестовый профиль

Создайте `application-test.yml`, который:
- Использует H2 in-memory вместо PostgreSQL
- Отключает отправку email
- Устанавливает случайный порт
- Снижает уровень логирования

### Задание 4: Диагностика

Приложение запущено, но `/api/users` возвращает 500 Internal Server Error.
Какие шаги вы предпримете для диагностики с помощью Actuator?

---

## Дополнительные ресурсы

- [Spring Boot Reference Documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/) — официальная документация
- [Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html) — руководство по Actuator
- [Common Application Properties](https://docs.spring.io/spring-boot/docs/current/reference/html/application-properties.html) — все настройки
- [Baeldung: Spring Boot Testing](https://www.baeldung.com/spring-boot-testing) — тестирование Spring Boot
- [Testcontainers](https://testcontainers.com/) — реальные зависимости в тестах
