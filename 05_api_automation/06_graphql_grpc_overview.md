# GraphQL и gRPC — обзор для QA

## Обзор

Помимо REST API, современные приложения используют и другие протоколы/подходы для обмена данными между сервисами. Два наиболее значимых — **GraphQL** и **gRPC**. Для QA-инженера важно понимать их основы, отличия от REST и знать инструменты для тестирования, так как всё больше компаний внедряют эти технологии.

GraphQL создан Facebook (2015) для гибкого получения данных — клиент сам определяет, какие поля ему нужны. gRPC создан Google для высокопроизводительного межсервисного взаимодействия на основе Protocol Buffers.

---

## GraphQL — основы

### Что такое GraphQL

GraphQL — это язык запросов для API и среда выполнения (runtime) для обработки этих запросов. В отличие от REST, где каждый ресурс имеет свой endpoint, в GraphQL есть **один endpoint** (`/graphql`), а клиент описывает, какие данные ему нужны.

### Ключевые концепции

| Концепция | Описание |
|-----------|----------|
| **Query** | Чтение данных (аналог GET в REST) |
| **Mutation** | Изменение данных (аналог POST/PUT/DELETE) |
| **Subscription** | Подписка на обновления в реальном времени (WebSocket) |
| **Schema** | Описание всех типов, запросов и мутаций API |
| **Resolver** | Функция, которая возвращает данные для конкретного поля |

### Schema и Types

```graphql
# Определение типов
type User {
  id: ID!              # ! означает обязательное поле (non-null)
  name: String!
  email: String!
  age: Int
  posts: [Post!]       # Массив постов (может быть пустым, но элементы non-null)
  address: Address
}

type Post {
  id: ID!
  title: String!
  body: String!
  author: User!
  comments: [Comment!]
  createdAt: String!
}

type Comment {
  id: ID!
  text: String!
  author: User!
}

type Address {
  city: String!
  zipCode: String
  country: String!
}

# Определение доступных запросов
type Query {
  user(id: ID!): User
  users(page: Int, size: Int): [User!]!
  post(id: ID!): Post
  posts(authorId: ID): [Post!]!
}

# Определение мутаций (изменение данных)
type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User!
  deleteUser(id: ID!): Boolean!
  createPost(input: CreatePostInput!): Post!
}

# Input-типы для мутаций
input CreateUserInput {
  name: String!
  email: String!
  age: Int
}

input UpdateUserInput {
  name: String
  email: String
  age: Int
}

input CreatePostInput {
  title: String!
  body: String!
  authorId: ID!
}

# Подписки
type Subscription {
  postCreated: Post!
  commentAdded(postId: ID!): Comment!
}
```

### Queries (чтение данных)

```graphql
# Простой запрос — получить пользователя с конкретными полями
query {
  user(id: "1") {
    name
    email
  }
}

# Ответ — только запрошенные поля
{
  "data": {
    "user": {
      "name": "Иван",
      "email": "ivan@test.com"
    }
  }
}

# Вложенные запросы — пользователь с постами и комментариями
query {
  user(id: "1") {
    name
    posts {
      title
      comments {
        text
        author {
          name
        }
      }
    }
  }
}

# С переменными (параметризация)
query GetUser($userId: ID!) {
  user(id: $userId) {
    name
    email
    age
  }
}

# Переменные передаются отдельно:
# { "userId": "1" }
```

### Mutations (изменение данных)

```graphql
# Создание пользователя
mutation {
  createUser(input: {
    name: "Новый пользователь"
    email: "new@test.com"
    age: 25
  }) {
    id
    name
    email
  }
}

# Обновление с переменными
mutation UpdateUser($id: ID!, $input: UpdateUserInput!) {
  updateUser(id: $id, input: $input) {
    id
    name
    email
  }
}

# Переменные:
# { "id": "1", "input": { "name": "Обновлённое имя" } }

# Удаление
mutation {
  deleteUser(id: "1")
}
```

### Subscriptions (подписки)

```graphql
# Подписка на создание новых постов (через WebSocket)
subscription {
  postCreated {
    id
    title
    author {
      name
    }
  }
}
```

---

## GraphQL vs REST — сравнение

| Критерий | REST | GraphQL |
|----------|------|---------|
| Endpoints | Много (`/users`, `/posts`, `/comments`) | Один (`/graphql`) |
| Формат данных | Фиксированный (сервер решает) | Гибкий (клиент решает) |
| Over-fetching | Да (сервер возвращает все поля) | Нет (клиент запрашивает нужные) |
| Under-fetching | Да (нужны множественные запросы) | Нет (один запрос для связанных данных) |
| HTTP-методы | GET, POST, PUT, PATCH, DELETE | POST (для всех операций) |
| Статус-коды | Семантические (200, 404, 500) | Почти всегда 200 (ошибки в теле) |
| Кэширование | HTTP-кэш (ETag, Cache-Control) | Сложнее (один URL для всех запросов) |
| Версионирование | URL (`/api/v1/`, `/api/v2/`) | Не нужно (эволюция схемы) |
| Документация | Swagger/OpenAPI | Самодокументирующееся (Schema Introspection) |
| Сложность | Проще для простых API | Проще для сложных связных данных |

### Проблемы over-fetching и under-fetching

```
Over-fetching (REST):
GET /api/users/1 → возвращает ВСЕ поля, хотя нужно только имя
{ "id": 1, "name": "Иван", "email": "...", "phone": "...", "address": {...}, "roles": [...] }

Under-fetching (REST):
Для отображения поста с автором и комментариями нужно 3 запроса:
1. GET /api/posts/1
2. GET /api/users/{authorId}
3. GET /api/posts/1/comments

GraphQL — один запрос:
query {
  post(id: "1") {
    title
    author { name }
    comments { text }
  }
}
```

---

## Тестирование GraphQL

### Инструменты

| Инструмент | Описание |
|-----------|----------|
| **GraphQL Playground / GraphiQL** | Веб-IDE для интерактивного тестирования запросов |
| **Postman** | Поддерживает GraphQL-запросы (вкладка Body → GraphQL) |
| **REST Assured** | GraphQL-запросы через POST с JSON-телом |
| **Insomnia** | REST-клиент с поддержкой GraphQL |
| **Apollo DevTools** | Расширение для Chrome (для Apollo Client) |

### Тестирование с REST Assured

GraphQL-запросы отправляются как **POST** с JSON-телом, содержащим `query` и опционально `variables`.

```java
class GraphqlApiTest {

    private static final String GRAPHQL_ENDPOINT = "https://api.example.com/graphql";

    @Test
    @DisplayName("GraphQL Query — получение пользователя")
    void shouldGetUserByGraphqlQuery() {
        String query = """
            {
                "query": "query { user(id: \\"1\\") { name email age } }"
            }
            """;

        given()
            .contentType(ContentType.JSON)
            .body(query)
        .when()
            .post(GRAPHQL_ENDPOINT)
        .then()
            .statusCode(200)
            .body("data.user.name", notNullValue())
            .body("data.user.email", containsString("@"))
            .body("errors", nullValue());  // Нет ошибок
    }

    @Test
    @DisplayName("GraphQL Query с переменными")
    void shouldGetUserWithVariables() {
        String query = """
            {
                "query": "query GetUser($id: ID!) { user(id: $id) { name email } }",
                "variables": { "id": "1" }
            }
            """;

        given()
            .contentType(ContentType.JSON)
            .body(query)
        .when()
            .post(GRAPHQL_ENDPOINT)
        .then()
            .statusCode(200)
            .body("data.user.name", notNullValue());
    }

    @Test
    @DisplayName("GraphQL Mutation — создание пользователя")
    void shouldCreateUserWithMutation() {
        String mutation = """
            {
                "query": "mutation { createUser(input: { name: \\"Тест\\", email: \\"test@example.com\\" }) { id name email } }"
            }
            """;

        given()
            .contentType(ContentType.JSON)
            .body(mutation)
        .when()
            .post(GRAPHQL_ENDPOINT)
        .then()
            .statusCode(200)
            .body("data.createUser.id", notNullValue())
            .body("data.createUser.name", equalTo("Тест"))
            .body("errors", nullValue());
    }

    @Test
    @DisplayName("GraphQL — обработка ошибок")
    void shouldReturnErrorForNonExistentUser() {
        String query = """
            {
                "query": "{ user(id: \\"99999\\") { name } }"
            }
            """;

        given()
            .contentType(ContentType.JSON)
            .body(query)
        .when()
            .post(GRAPHQL_ENDPOINT)
        .then()
            .statusCode(200)                    // GraphQL всегда возвращает 200
            .body("data.user", nullValue())     // Данных нет
            .body("errors", notNullValue())     // Ошибки в теле ответа
            .body("errors[0].message", notNullValue());
    }

    @Test
    @DisplayName("GraphQL — невалидный запрос (синтаксическая ошибка)")
    void shouldReturnErrorForInvalidQuery() {
        String invalidQuery = """
            {
                "query": "{ user(id: 1 { name } }"
            }
            """;

        given()
            .contentType(ContentType.JSON)
            .body(invalidQuery)
        .when()
            .post(GRAPHQL_ENDPOINT)
        .then()
            .statusCode(200)  // Или 400 — зависит от реализации
            .body("errors", notNullValue());
    }

    @Test
    @DisplayName("GraphQL Introspection — получение схемы")
    void shouldReturnSchemaViaIntrospection() {
        String introspectionQuery = """
            {
                "query": "{ __schema { types { name kind } } }"
            }
            """;

        given()
            .contentType(ContentType.JSON)
            .body(introspectionQuery)
        .when()
            .post(GRAPHQL_ENDPOINT)
        .then()
            .statusCode(200)
            .body("data.__schema.types", notNullValue())
            .body("data.__schema.types.size()", greaterThan(0));
    }
}
```

### Что проверять при тестировании GraphQL

1. **Корректность данных** — запрошенные поля возвращают правильные значения
2. **Отсутствие лишних полей** — ответ содержит только запрошенные поля
3. **Обработка ошибок** — ошибки возвращаются в массиве `errors`
4. **Валидация input** — невалидные данные в mutation возвращают понятные ошибки
5. **N+1 проблема** — вложенные запросы не создают чрезмерную нагрузку на базу
6. **Глубина запроса** — защита от слишком глубоких вложенных запросов (DoS)
7. **Introspection** — должен быть отключён в production (безопасность)
8. **Авторизация** — проверка доступа к полям и мутациям

---

## gRPC — основы

### Что такое gRPC

gRPC (Google Remote Procedure Call) — высокопроизводительный фреймворк удалённого вызова процедур. Использует **Protocol Buffers** (protobuf) для сериализации данных и **HTTP/2** для транспорта.

### Ключевые концепции

| Концепция | Описание |
|-----------|----------|
| **Protocol Buffers** | Бинарный формат сериализации данных (`.proto` файлы) |
| **Service Definition** | Описание API в `.proto` файле |
| **Unary RPC** | Один запрос — один ответ (аналог REST) |
| **Server Streaming** | Один запрос — поток ответов |
| **Client Streaming** | Поток запросов — один ответ |
| **Bidirectional Streaming** | Потоки в обе стороны |
| **Stub** | Сгенерированный клиент для вызова gRPC-сервисов |

### Protocol Buffers (.proto файлы)

```protobuf
syntax = "proto3";

package user;

option java_multiple_files = true;
option java_package = "com.example.grpc.user";

// Определение сервиса (аналог контроллера в REST)
service UserService {
  // Unary — один запрос, один ответ
  rpc GetUser (GetUserRequest) returns (UserResponse);
  rpc CreateUser (CreateUserRequest) returns (UserResponse);
  rpc DeleteUser (DeleteUserRequest) returns (DeleteUserResponse);

  // Server Streaming — один запрос, поток ответов
  rpc ListUsers (ListUsersRequest) returns (stream UserResponse);

  // Client Streaming — поток запросов, один ответ
  rpc UploadUsers (stream CreateUserRequest) returns (UploadUsersResponse);

  // Bidirectional Streaming — потоки в обе стороны
  rpc Chat (stream ChatMessage) returns (stream ChatMessage);
}

// Сообщения (аналог DTO/POJO)
message GetUserRequest {
  int32 id = 1;    // Номер поля (не значение!)
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
  int32 age = 3;
  Address address = 4;
}

message UserResponse {
  int32 id = 1;
  string name = 2;
  string email = 3;
  int32 age = 4;
  bool active = 5;
  Address address = 6;
}

message Address {
  string city = 1;
  string zip_code = 2;
  string country = 3;
}

message ListUsersRequest {
  int32 page = 1;
  int32 size = 2;
}

message DeleteUserRequest {
  int32 id = 1;
}

message DeleteUserResponse {
  bool success = 1;
}

message UploadUsersResponse {
  int32 created_count = 1;
}

message ChatMessage {
  string sender = 1;
  string text = 2;
  int64 timestamp = 3;
}
```

### gRPC vs REST — сравнение

| Критерий | REST | gRPC |
|----------|------|------|
| Протокол | HTTP/1.1 (или 2) | HTTP/2 (обязательно) |
| Формат данных | JSON/XML (текстовый) | Protocol Buffers (бинарный) |
| Скорость | Средняя | Высокая (в 7-10 раз быстрее) |
| Типизация | Слабая (JSON) | Строгая (protobuf schema) |
| Streaming | Нет (нужен WebSocket) | Нативный (4 режима) |
| Генерация кода | Опционально (OpenAPI) | Обязательно (из .proto) |
| Читаемость | Высокая (JSON) | Низкая (бинарный) |
| Браузер | Нативная поддержка | Нужен grpc-web proxy |
| Применение | Публичные API, фронтенд | Микросервисы, внутренние API |

### Типы RPC-вызовов

```
Unary (один к одному):
Клиент ──request──→ Сервер
Клиент ←─response── Сервер

Server Streaming (один ко многим):
Клиент ──request──→ Сервер
Клиент ←─response── Сервер
Клиент ←─response── Сервер
Клиент ←─response── Сервер

Client Streaming (многие к одному):
Клиент ──request──→ Сервер
Клиент ──request──→ Сервер
Клиент ──request──→ Сервер
Клиент ←─response── Сервер

Bidirectional Streaming (многие ко многим):
Клиент ──request──→ Сервер
Клиент ←─response── Сервер
Клиент ──request──→ Сервер
Клиент ←─response── Сервер
```

---

## Тестирование gRPC

### Инструменты

| Инструмент | Описание |
|-----------|----------|
| **grpcurl** | CLI для gRPC (аналог curl) |
| **BloomRPC** | GUI-клиент для gRPC (аналог Postman) |
| **Postman** | Поддержка gRPC (с версии 10) |
| **Evans** | Интерактивный gRPC-клиент (CLI) |
| **gRPC Java Testing** | Библиотеки для unit-тестирования в Java |

### grpcurl — примеры

```bash
# Список сервисов (требуется reflection)
grpcurl -plaintext localhost:9090 list

# Описание сервиса
grpcurl -plaintext localhost:9090 describe user.UserService

# Unary-вызов
grpcurl -plaintext \
  -d '{"id": 1}' \
  localhost:9090 user.UserService/GetUser

# Вызов с .proto файлом (без reflection)
grpcurl -plaintext \
  -import-path ./proto \
  -proto user.proto \
  -d '{"name": "Тест", "email": "test@example.com"}' \
  localhost:9090 user.UserService/CreateUser
```

### Тестирование в Java

```java
import io.grpc.ManagedChannel;
import io.grpc.ManagedChannelBuilder;
import io.grpc.StatusRuntimeException;
import com.example.grpc.user.*;

class GrpcUserServiceTest {

    private ManagedChannel channel;
    private UserServiceGrpc.UserServiceBlockingStub blockingStub;

    @BeforeEach
    void setup() {
        // Создание канала к gRPC-серверу
        channel = ManagedChannelBuilder
            .forAddress("localhost", 9090)
            .usePlaintext()   // Без TLS для тестов
            .build();

        // Создание синхронного stub
        blockingStub = UserServiceGrpc.newBlockingStub(channel);
    }

    @AfterEach
    void teardown() {
        channel.shutdownNow();
    }

    @Test
    @DisplayName("Получение пользователя по ID")
    void shouldGetUser() {
        // Формируем запрос
        GetUserRequest request = GetUserRequest.newBuilder()
            .setId(1)
            .build();

        // Отправляем запрос
        UserResponse response = blockingStub.getUser(request);

        // Проверяем ответ
        assertEquals(1, response.getId());
        assertNotNull(response.getName());
        assertTrue(response.getEmail().contains("@"));
    }

    @Test
    @DisplayName("Создание пользователя")
    void shouldCreateUser() {
        CreateUserRequest request = CreateUserRequest.newBuilder()
            .setName("Тестовый пользователь")
            .setEmail("test@example.com")
            .setAge(30)
            .setAddress(Address.newBuilder()
                .setCity("Москва")
                .setCountry("Россия")
                .build())
            .build();

        UserResponse response = blockingStub.createUser(request);

        assertTrue(response.getId() > 0);
        assertEquals("Тестовый пользователь", response.getName());
        assertEquals("Москва", response.getAddress().getCity());
    }

    @Test
    @DisplayName("Ошибка при запросе несуществующего пользователя")
    void shouldThrowNotFoundForNonExistentUser() {
        GetUserRequest request = GetUserRequest.newBuilder()
            .setId(99999)
            .build();

        StatusRuntimeException exception = assertThrows(
            StatusRuntimeException.class,
            () -> blockingStub.getUser(request)
        );

        assertEquals(Status.NOT_FOUND.getCode(), exception.getStatus().getCode());
    }

    @Test
    @DisplayName("Server Streaming — получение списка пользователей")
    void shouldStreamUsers() {
        ListUsersRequest request = ListUsersRequest.newBuilder()
            .setPage(0)
            .setSize(10)
            .build();

        // Собираем все ответы из потока
        Iterator<UserResponse> responseIterator = blockingStub.listUsers(request);
        List<UserResponse> users = new ArrayList<>();
        responseIterator.forEachRemaining(users::add);

        assertFalse(users.isEmpty());
        assertTrue(users.size() <= 10);
    }
}
```

### Что проверять при тестировании gRPC

1. **Unary-вызовы** — корректность запроса и ответа
2. **Streaming** — все сообщения доставлены, порядок сохранён
3. **Статус-коды gRPC** — `OK`, `NOT_FOUND`, `INVALID_ARGUMENT`, `UNAUTHENTICATED`, `PERMISSION_DENIED`
4. **Обработка ошибок** — корректные статусы при невалидных данных
5. **Метаданные** (metadata) — аналог HTTP-заголовков
6. **Deadlines/Timeouts** — поведение при истечении deadline
7. **Backward compatibility** — добавление новых полей не ломает старых клиентов

### Статус-коды gRPC

| Код | Описание | Аналог HTTP |
|-----|----------|-------------|
| OK | Успех | 200 |
| CANCELLED | Операция отменена | 499 |
| INVALID_ARGUMENT | Невалидные аргументы | 400 |
| NOT_FOUND | Ресурс не найден | 404 |
| ALREADY_EXISTS | Ресурс уже существует | 409 |
| PERMISSION_DENIED | Нет прав | 403 |
| UNAUTHENTICATED | Не аутентифицирован | 401 |
| RESOURCE_EXHAUSTED | Превышен лимит | 429 |
| UNAVAILABLE | Сервис недоступен | 503 |
| INTERNAL | Внутренняя ошибка | 500 |
| DEADLINE_EXCEEDED | Превышен timeout | 504 |
| UNIMPLEMENTED | Метод не реализован | 501 |

---

## Когда QA встречает эти технологии

### GraphQL

- **Фронтенд-приложения** — мобильные и SPA-приложения часто используют GraphQL
- **BFF (Backend for Frontend)** — GraphQL как промежуточный слой между фронтендом и микросервисами
- **Публичные API** — GitHub API v4, Shopify API, Hasura

### gRPC

- **Микросервисная архитектура** — внутреннее взаимодействие между сервисами
- **Высоконагруженные системы** — финтех, стриминг, игровые серверы
- **IoT** — эффективная коммуникация с устройствами
- **Mobile backend** — экономия трафика за счёт бинарного формата

### Что QA-инженер должен уметь

| Технология | Минимальные навыки | Продвинутые навыки |
|------------|-------------------|-------------------|
| GraphQL | Писать queries/mutations, тестировать через Postman | Автоматизация с REST Assured, тестирование подписок |
| gRPC | Вызывать методы через grpcurl/BloomRPC | Автоматизация на Java, тестирование streaming |

---

## Связь с тестированием

### GraphQL — специфика тестирования

- Один endpoint → тест-кейсы строятся вокруг **запросов**, а не endpoint-ов
- Ошибки в теле ответа → нельзя полагаться только на HTTP-статус
- Гибкая структура → нужно проверять, что возвращаются именно запрошенные поля
- Introspection → можно автоматически генерировать тесты на основе схемы

### gRPC — специфика тестирования

- Строгая типизация → меньше ошибок формата, но нужно тестировать граничные значения
- Бинарный формат → нельзя просто посмотреть в DevTools
- Streaming → нужны специфические тесты для потоковых режимов
- Backward compatibility → добавление полей безопасно, удаление/переименование — нет

---

## Типичные ошибки

### GraphQL
1. **Проверяют только HTTP-статус** — GraphQL может вернуть 200 с ошибками в `errors`
2. **Не тестируют глубокие вложенные запросы** — N+1 проблема, DoS через глубокие запросы
3. **Не проверяют introspection в production** — должен быть отключён из соображений безопасности
4. **Забывают про валидацию input** — мутации с невалидными данными
5. **Не тестируют авторизацию на уровне полей** — пользователь может не иметь доступа к отдельным полям

### gRPC
1. **Не тестируют deadline/timeout** — клиент может ждать бесконечно
2. **Не проверяют backward compatibility** — удаление поля ломает старых клиентов
3. **Не тестируют streaming** — тестируют только unary-вызовы
4. **Не учитывают metadata** — аналог заголовков, может содержать важную информацию
5. **Не проверяют обработку ошибок** — коды gRPC отличаются от HTTP

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Что такое GraphQL? Чем отличается от REST?
2. Что такое query и mutation в GraphQL?
3. Что такое gRPC? Для чего используется?
4. Что такое Protocol Buffers?
5. Какие инструменты используются для тестирования GraphQL?

### 🟡 Средний уровень
6. Что такое over-fetching и under-fetching? Как GraphQL решает эти проблемы?
7. Как тестировать GraphQL API с помощью REST Assured?
8. Какие типы RPC-вызовов существуют в gRPC?
9. Как обрабатываются ошибки в GraphQL (массив errors)?
10. Чем статус-коды gRPC отличаются от HTTP?
11. Что такое Schema Introspection в GraphQL?

### 🔴 Продвинутый уровень
12. Как тестировать GraphQL Subscriptions (WebSocket)?
13. Как обеспечить backward compatibility в gRPC при изменении .proto?
14. Какие проблемы безопасности специфичны для GraphQL? (DoS через глубокие запросы, introspection)
15. Как тестировать bidirectional streaming в gRPC?
16. Когда стоит выбрать GraphQL, когда gRPC, а когда REST?

---

## Практические задания

### Задание 1: GraphQL-запросы
Используя публичный GraphQL API (например, `https://countries.trevorblades.com/graphql`), напишите queries для получения списка стран, страны по коду, фильтрации по континенту. Используйте Postman или REST Assured.

### Задание 2: GraphQL Mutations
Настройте локальный GraphQL-сервер (например, с помощью json-graphql-server) и напишите тесты для CRUD-операций через mutations.

### Задание 3: gRPC с grpcurl
Установите grpcurl. Найдите публичный gRPC-сервис (или поднимите свой) и выполните: list сервисов, describe, вызов unary-метода, анализ ответа.

### Задание 4: Сравнительный анализ
Реализуйте одно и то же API (пользователи + посты) в REST, GraphQL и gRPC. Сравните: количество запросов для получения связанных данных, объём передаваемых данных, удобство тестирования.

---

## Дополнительные ресурсы

### GraphQL
- [GraphQL Official Documentation](https://graphql.org/learn/)
- [GraphQL Playground](https://www.graphqlbin.com)
- [Apollo GraphQL](https://www.apollographql.com/docs/)
- [Public GraphQL APIs for testing](https://github.com/APIs-guru/graphql-apis)

### gRPC
- [gRPC Official Documentation](https://grpc.io/docs/)
- [Protocol Buffers Language Guide](https://protobuf.dev/programming-guides/proto3/)
- [grpcurl (CLI tool)](https://github.com/fullstorydev/grpcurl)
- [BloomRPC (GUI client)](https://github.com/bloomrpc/bloomrpc)
- [gRPC Java Quick Start](https://grpc.io/docs/languages/java/quickstart/)
