# Контрактное тестирование

## Обзор

Контрактное тестирование (Contract Testing) — подход к тестированию, при котором взаимодействие между сервисами проверяется на соответствие формализованному контракту (договору). Контракт описывает, какой запрос отправляет потребитель (consumer) и какой ответ ожидает от поставщика (provider).

В микросервисной архитектуре сервисы развиваются независимо, и изменения в одном сервисе могут сломать другой. Контрактное тестирование позволяет обнаруживать такие проблемы **до деплоя** — без необходимости поднимать все сервисы вместе.

Для QA-инженера понимание контрактного тестирования — важный навык при работе с микросервисами, особенно в контексте CI/CD-пайплайнов.

---

## Зачем нужно контрактное тестирование

### Проблема в микросервисах

```
Service A (Consumer)  →  HTTP  →  Service B (Provider)
                                    ↑
                      Изменили формат ответа
                      (переименовали поле "userName" → "name")
                                    ↓
              Service A падает в production 💥
```

### Как традиционные тесты решают проблему

| Тип тестирования | Подход | Проблемы |
|-----------------|--------|----------|
| Unit-тесты | Мокают внешние зависимости | Не проверяют реальное взаимодействие |
| Integration-тесты (E2E) | Поднимают все сервисы | Медленно, нестабильно, сложно в настройке |
| **Контрактные тесты** | Проверяют контракт независимо | Быстро, надёжно, каждый сервис тестируется отдельно |

### Когда использовать контрактное тестирование

**Подходит:**
- Микросервисная архитектура с множеством взаимозависимых сервисов
- Команды разрабатывают сервисы независимо
- Частые деплои (CI/CD)
- API публичное или используется несколькими потребителями

**Не подходит:**
- Монолитное приложение
- Один потребитель, один поставщик, одна команда
- Крайне простое API (1-2 endpoint-а)

---

## Consumer-Driven Contracts (CDC)

### Концепция

При подходе CDC потребитель (consumer) определяет, какие данные ему нужны. Контракт формируется **со стороны потребителя**, а поставщик (provider) обязуется его выполнять.

```
Потребитель (Consumer)                    Поставщик (Provider)
┌─────────────────────┐                  ┌─────────────────────┐
│ 1. Пишет тест,      │                  │ 3. Загружает pact-  │
│    описывающий       │─── pact file ──→│    файл и проверяет  │
│    ожидаемое         │                  │    реальный API на   │
│    взаимодействие    │                  │    соответствие      │
│                      │                  │    контракту         │
│ 2. Генерирует pact-  │                  │                     │
│    файл (контракт)   │                  │ 4. Если всё ок —    │
│                      │                  │    контракт выполнен │
└─────────────────────┘                  └─────────────────────┘
```

### Процесс

1. **Consumer-тест** — потребитель описывает ожидаемое взаимодействие (запрос + ожидаемый ответ)
2. **Pact-файл** — генерируется автоматически из consumer-теста (JSON-файл с контрактом)
3. **Provider-верификация** — поставщик запускает реальный API и проверяет его против pact-файла
4. **Обмен контрактами** — через Pact Broker или файловую систему

---

## Pact Framework

Pact — наиболее популярный фреймворк для контрактного тестирования. Поддерживает множество языков: Java, JavaScript, Python, Go, .NET и др.

### Настройка (Maven)

```xml
<dependencies>
    <!-- Pact Consumer -->
    <dependency>
        <groupId>au.com.dius.pact.consumer</groupId>
        <artifactId>junit5</artifactId>
        <version>4.6.7</version>
        <scope>test</scope>
    </dependency>

    <!-- Pact Provider -->
    <dependency>
        <groupId>au.com.dius.pact.provider</groupId>
        <artifactId>junit5</artifactId>
        <version>4.6.7</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Consumer-тест (сторона потребителя)

```java
import au.com.dius.pact.consumer.dsl.DslPart;
import au.com.dius.pact.consumer.dsl.PactDslJsonBody;
import au.com.dius.pact.consumer.dsl.PactDslWithProvider;
import au.com.dius.pact.consumer.junit5.PactConsumerTestExt;
import au.com.dius.pact.consumer.junit5.PactTestFor;
import au.com.dius.pact.consumer.MockServer;
import au.com.dius.pact.core.model.V4Pact;
import au.com.dius.pact.core.model.annotations.Pact;

@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "UserService")
class UserConsumerPactTest {

    // Описание контракта: что потребитель ожидает от поставщика
    @Pact(consumer = "OrderService")
    public V4Pact getUserByIdPact(PactDslWithProvider builder) {
        return builder
            .given("Пользователь с id=1 существует")  // Начальное состояние провайдера
            .uponReceiving("Запрос пользователя по ID")  // Описание взаимодействия
                .path("/api/users/1")
                .method("GET")
                .headers("Accept", "application/json")
            .willRespondWith()
                .status(200)
                .headers(Map.of("Content-Type", "application/json"))
                .body(new PactDslJsonBody()
                    .integerType("id", 1)            // Тип integer, пример: 1
                    .stringType("name", "Иван")      // Тип string, пример: Иван
                    .stringMatcher("email",           // Regex-матчер
                        ".*@.*\\..*", "ivan@test.com")
                    .booleanType("active", true)      // Тип boolean
                )
            .toPact(V4Pact.class);
    }

    // Контракт для несуществующего пользователя
    @Pact(consumer = "OrderService")
    public V4Pact getUserNotFoundPact(PactDslWithProvider builder) {
        return builder
            .given("Пользователь с id=999 не существует")
            .uponReceiving("Запрос несуществующего пользователя")
                .path("/api/users/999")
                .method("GET")
            .willRespondWith()
                .status(404)
                .body(new PactDslJsonBody()
                    .stringType("error", "User not found")
                )
            .toPact(V4Pact.class);
    }

    // Тест: потребитель обращается к mock-серверу Pact
    @Test
    @PactTestFor(pactMethod = "getUserByIdPact")
    void shouldGetUserById(MockServer mockServer) {
        // Pact поднимает mock-сервер с описанным поведением
        User user = given()
            .baseUri(mockServer.getUrl())
            .header("Accept", "application/json")
        .when()
            .get("/api/users/1")
        .then()
            .statusCode(200)
            .extract()
            .as(User.class);

        assertEquals("Иван", user.getName());
        assertNotNull(user.getEmail());
    }

    @Test
    @PactTestFor(pactMethod = "getUserNotFoundPact")
    void shouldReturn404ForNonExistentUser(MockServer mockServer) {
        given()
            .baseUri(mockServer.getUrl())
        .when()
            .get("/api/users/999")
        .then()
            .statusCode(404)
            .body("error", equalTo("User not found"));
    }
}
```

### Результат: Pact-файл (контракт)

После выполнения consumer-теста генерируется JSON-файл (обычно в `target/pacts/`):

```json
{
  "consumer": { "name": "OrderService" },
  "provider": { "name": "UserService" },
  "interactions": [
    {
      "description": "Запрос пользователя по ID",
      "providerStates": [
        { "name": "Пользователь с id=1 существует" }
      ],
      "request": {
        "method": "GET",
        "path": "/api/users/1",
        "headers": { "Accept": "application/json" }
      },
      "response": {
        "status": 200,
        "headers": { "Content-Type": "application/json" },
        "body": {
          "id": 1,
          "name": "Иван",
          "email": "ivan@test.com",
          "active": true
        },
        "matchingRules": {
          "body": {
            "$.id": { "matchers": [{ "match": "integer" }] },
            "$.name": { "matchers": [{ "match": "type" }] },
            "$.email": { "matchers": [{ "match": "regex", "regex": ".*@.*\\..*" }] }
          }
        }
      }
    }
  ]
}
```

### Provider-верификация (сторона поставщика)

```java
import au.com.dius.pact.provider.junit5.PactVerificationContext;
import au.com.dius.pact.provider.junit5.PactVerificationInvocationContextProvider;
import au.com.dius.pact.provider.junitsupport.Provider;
import au.com.dius.pact.provider.junitsupport.State;
import au.com.dius.pact.provider.junitsupport.loader.PactFolder;

@Provider("UserService")
@PactFolder("pacts")  // Путь к pact-файлам (или @PactBroker для Pact Broker)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserProviderPactTest {

    @LocalServerPort
    private int port;

    @Autowired
    private UserRepository userRepository;

    @BeforeEach
    void setup(PactVerificationContext context) {
        context.setTarget(new HttpTestTarget("localhost", port));
    }

    // Подготовка начального состояния для каждого providerState
    @State("Пользователь с id=1 существует")
    void userExists() {
        // Создаём тестовые данные в базе
        userRepository.save(new UserEntity(1L, "Иван", "ivan@test.com", true));
    }

    @State("Пользователь с id=999 не существует")
    void userDoesNotExist() {
        // Убеждаемся, что пользователя нет
        userRepository.deleteById(999L);
    }

    // Запуск верификации — Pact воспроизводит запросы из контракта
    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void verifyPact(PactVerificationContext context) {
        context.verifyInteraction();
    }
}
```

---

## Pact Broker

Pact Broker — сервер для хранения и обмена pact-файлами между командами.

```
Consumer CI/CD                   Pact Broker                  Provider CI/CD
┌───────────┐                  ┌─────────────┐               ┌───────────┐
│ Запуск     │  публикация     │  Хранилище  │  загрузка     │ Запуск    │
│ consumer-  │───контракта───→│  контрактов  │←──контракта──│ provider- │
│ тестов     │                 │             │               │ верифика- │
│            │                 │  Визуализа- │               │ ции       │
│ Генерация  │                 │  ция связей │               │           │
│ pact-файла │                 │  между      │               │ Результат │
│            │                 │  сервисами  │               │ публику-  │
└───────────┘                  └─────────────┘               │ ется в    │
                                                             │ Broker    │
                                                             └───────────┘
```

### Команда can-i-deploy

```bash
# Проверка перед деплоем: безопасно ли деплоить данную версию?
pact-broker can-i-deploy \
  --pacticipant UserService \
  --version 1.2.3 \
  --to production
```

---

## Spring Cloud Contract (обзор)

Spring Cloud Contract — альтернатива Pact, разработанная для Spring-экосистемы.

### Ключевые отличия от Pact

| Аспект | Pact | Spring Cloud Contract |
|--------|------|----------------------|
| Подход | Consumer-driven | Provider-driven (контракт на стороне провайдера) |
| Язык контракта | JSON (генерируется из кода) | Groovy DSL, YAML, Java |
| Экосистема | Мультиязычный | Spring Boot |
| Стабы | Mock-сервер Pact | WireMock стабы (генерируются автоматически) |
| Брокер | Pact Broker | Артефакт в Maven-репозитории |

### Пример контракта (Groovy DSL)

```groovy
Contract.make {
    description "Получение пользователя по ID"

    request {
        method GET()
        url "/api/users/1"
        headers {
            accept(applicationJson())
        }
    }

    response {
        status OK()
        headers {
            contentType(applicationJson())
        }
        body([
            id   : 1,
            name : $(producer("Иван"), consumer(regex("[А-Яа-яA-Za-z]+"))),
            email: $(producer("ivan@test.com"), consumer(regex(".+@.+\\..+")))
        ])
    }
}
```

### Как это работает

1. Разработчик провайдера пишет контракт (DSL)
2. Spring Cloud Contract **генерирует тесты** для провайдера автоматически
3. Spring Cloud Contract **генерирует WireMock-стабы** для потребителей
4. Потребители используют стабы как зависимость из Maven-репозитория

---

## Контрактное тестирование vs Интеграционное тестирование

| Критерий | Контрактное тестирование | E2E / Интеграционное тестирование |
|----------|------------------------|----------------------------------|
| Скорость | Быстро (секунды) | Медленно (минуты) |
| Стабильность | Высокая | Низкая (flaky-тесты) |
| Необходимость инфраструктуры | Минимальная | Нужны все сервисы |
| Что проверяет | Совместимость API | Полный flow |
| Глубина проверки | Формат запросов/ответов | Бизнес-логика end-to-end |
| Обнаружение багов | Несовместимость между сервисами | Интеграционные баги |
| Стоимость поддержки | Низкая | Высокая |
| Когда запускать | На каждый commit | Перед релизом |
| Замена друг другу | Нет | Нет |

> **Важно:** контрактные тесты **не заменяют** интеграционные. Они **дополняют** друг друга. Контрактные тесты проверяют совместимость API, а интеграционные — бизнес-логику.

---

## Matching Rules в Pact

Pact позволяет проверять не конкретные значения, а **типы и паттерны**:

```java
new PactDslJsonBody()
    // Проверка типа (значение может быть любым)
    .integerType("id")                  // Любое целое число
    .stringType("name")                 // Любая строка
    .booleanType("active")              // Любой boolean
    .numberType("price")                // Любое число

    // Проверка по regex
    .stringMatcher("email", ".*@.*\\..*", "test@example.com")
    .stringMatcher("phone", "\\+7-\\d{3}-\\d{3}-\\d{2}-\\d{2}", "+7-999-123-45-67")

    // Проверка дат
    .date("birthDate", "yyyy-MM-dd", "1990-01-15")
    .datetime("createdAt", "yyyy-MM-dd'T'HH:mm:ss'Z'")

    // Массивы
    .minArrayLike("orders", 1)          // Минимум 1 элемент
        .integerType("orderId")
        .stringType("status")
    .closeArray()

    // Вложенные объекты
    .object("address")
        .stringType("city")
        .stringType("zipCode")
    .closeObject();
```

---

## Связь с тестированием

### Роль QA в контрактном тестировании

1. **Определение контрактов** — QA помогает определить, какие поля обязательны, какие форматы допустимы
2. **Написание consumer-тестов** — QA пишет тесты, описывающие ожидания потребителя
3. **Ревью контрактов** — проверка полноты контрактов (все endpoint-ы, все сценарии)
4. **Интеграция в CI/CD** — настройка запуска контрактных тестов в пайплайне
5. **Мониторинг Pact Broker** — отслеживание совместимости сервисов

### Пирамида тестирования с контрактами

```
         /\
        /  \         E2E-тесты (мало, медленные)
       /    \
      /------\
     /        \      Контрактные тесты (средний слой)
    /----------\
   /            \    Интеграционные тесты (компоненты)
  /--------------\
 /                \  Unit-тесты (много, быстрые)
/------------------\
```

---

## Типичные ошибки

1. **Тестирование конкретных значений вместо типов** — контракт должен проверять структуру и типы, а не конкретные данные
2. **Слишком строгие контракты** — контракт ломается при добавлении нового поля (используйте `additionalProperties: true`)
3. **Слишком слабые контракты** — контракт проверяет только статус-код без тела ответа
4. **Не используют provider states** — забывают настроить начальное состояние данных на стороне провайдера
5. **Хранят pact-файлы в коде потребителя** — нужно использовать Pact Broker для обмена
6. **Путают контрактные и E2E-тесты** — пытаются проверить бизнес-логику через контракты
7. **Не запускают can-i-deploy** — деплоят без проверки совместимости
8. **Игнорируют негативные сценарии** — контракты только для happy path

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Что такое контрактное тестирование?
2. Зачем нужно контрактное тестирование в микросервисной архитектуре?
3. Что такое consumer и provider в контексте контрактного тестирования?
4. Что такое pact-файл?

### 🟡 Средний уровень
5. Объясните подход Consumer-Driven Contracts.
6. Из каких шагов состоит процесс контрактного тестирования с Pact?
7. Чем контрактное тестирование отличается от интеграционного?
8. Что такое provider state и зачем он нужен?
9. Что такое Pact Broker?
10. Когда контрактное тестирование не имеет смысла?

### 🔴 Продвинутый уровень
11. Чем Pact отличается от Spring Cloud Contract? Когда что выбирать?
12. Как организовать контрактное тестирование в CI/CD pipeline?
13. Что такое can-i-deploy и как это работает?
14. Как обрабатывать breaking changes при наличии контрактов?
15. Как тестировать асинхронное взаимодействие (messaging) через контракты?

---

## Практические задания

### Задание 1: Consumer-тест с Pact
Напишите consumer-тест для сервиса заказов (OrderService), который обращается к сервису пользователей (UserService) для получения данных пользователя по ID. Опишите контракт для успешного и неуспешного сценариев.

### Задание 2: Provider-верификация
На стороне UserService напишите тест верификации, который загружает pact-файл и проверяет реальный API на соответствие контракту. Реализуйте provider states.

### Задание 3: Сравнительный анализ
Составьте таблицу сравнения: какие баги находит контрактное тестирование, какие — интеграционное, какие — unit-тестирование. Приведите конкретные примеры.

### Задание 4: Matching Rules
Напишите контракт с различными matching rules: типы, regex, массивы, вложенные объекты. Убедитесь, что контракт не ломается при изменении конкретных значений.

---

## Дополнительные ресурсы

- [Pact Official Documentation](https://docs.pact.io)
- [Pact JVM (Java)](https://github.com/pact-foundation/pact-jvm)
- [Pact Broker](https://docs.pact.io/pact_broker)
- [Spring Cloud Contract Documentation](https://spring.io/projects/spring-cloud-contract)
- [Contract Testing vs Integration Testing (Martin Fowler)](https://martinfowler.com/bliki/ContractTest.html)
- [Consumer-Driven Contracts: A Service Evolution Pattern](https://martinfowler.com/articles/consumerDrivenContracts.html)
