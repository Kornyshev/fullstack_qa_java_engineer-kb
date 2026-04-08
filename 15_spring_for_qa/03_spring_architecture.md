# Архитектура Spring приложения

## Обзор

Подавляющее большинство Spring-приложений построено по слоистой архитектуре: Controller → Service →
Repository. Для QA-инженера понимание этой архитектуры критически важно: оно определяет, какие тесты
писать для каждого слоя, где искать причины багов, как устроен поток запроса от HTTP-вызова до базы
данных и обратно. Этот раздел объясняет назначение каждого слоя, как данные преобразуются между слоями,
и что тестировать на каждом уровне.

---

## Трёхслойная архитектура

### Общая схема

```
       HTTP-запрос (клиент, браузер, Postman)
              │
              ▼
┌─────────────────────────────┐
│     Controller (Web Layer)  │  ← Принимает запросы, валидирует вход,
│     @RestController         │    возвращает HTTP-ответ
└─────────────┬───────────────┘
              │ DTO → Entity (или наоборот)
              ▼
┌─────────────────────────────┐
│     Service (Business Logic)│  ← Бизнес-логика, транзакции,
│     @Service                │    оркестрация вызовов
└─────────────┬───────────────┘
              │ Entity
              ▼
┌─────────────────────────────┐
│   Repository (Data Layer)   │  ← Доступ к данным, SQL-запросы,
│   @Repository               │    CRUD-операции
└─────────────┬───────────────┘
              │ JDBC / JPA
              ▼
        ┌──────────┐
        │    БД    │
        └──────────┘
```

### Принцип разделения ответственности

| Слой | Ответственность | НЕ должен делать |
|------|-----------------|------------------|
| Controller | Приём HTTP-запросов, валидация входных данных, формирование HTTP-ответа | Бизнес-логику, работу с БД |
| Service | Бизнес-логика, оркестрация, транзакции | Знать о HTTP, формировать ответы |
| Repository | CRUD-операции, SQL-запросы, доступ к данным | Бизнес-логику, HTTP |

---

## Controller Layer — веб-слой

Controller принимает HTTP-запросы и преобразует их в вызовы сервисов.

### Типичный REST Controller

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    // Constructor injection — зависимость от сервисного слоя
    public UserController(UserService userService) {
        this.userService = userService;
    }

    // GET /api/users — получить всех пользователей
    @GetMapping
    public ResponseEntity<List<UserResponse>> getAllUsers() {
        List<UserResponse> users = userService.getAllUsers();
        return ResponseEntity.ok(users);
    }

    // GET /api/users/{id} — получить пользователя по ID
    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUserById(@PathVariable Long id) {
        UserResponse user = userService.getUserById(id);
        return ResponseEntity.ok(user);
    }

    // POST /api/users — создать пользователя
    @PostMapping
    public ResponseEntity<UserResponse> createUser(
            @Valid @RequestBody CreateUserRequest request) {
        UserResponse created = userService.createUser(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }

    // PUT /api/users/{id} — обновить пользователя
    @PutMapping("/{id}")
    public ResponseEntity<UserResponse> updateUser(
            @PathVariable Long id,
            @Valid @RequestBody UpdateUserRequest request) {
        UserResponse updated = userService.updateUser(id, request);
        return ResponseEntity.ok(updated);
    }

    // DELETE /api/users/{id} — удалить пользователя
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.deleteUser(id);
        return ResponseEntity.noContent().build();
    }
}
```

### Что тестировать в Controller

| Аспект | Пример проверки |
|--------|----------------|
| HTTP-статусы | 200 OK, 201 Created, 400 Bad Request, 404 Not Found |
| Валидация входных данных | Пустое имя → 400, невалидный email → 400 |
| Формат ответа | JSON-структура, наличие обязательных полей |
| Маршрутизация | GET /api/users/{id} попадает в нужный метод |
| Content-Type | Ответ имеет правильный Content-Type |
| Обработка ошибок | Корректный формат ошибок, нет стектрейсов в ответе |

---

## Service Layer — бизнес-логика

Service содержит бизнес-правила и оркестрирует взаимодействие между компонентами.

### Типичный Service

```java
@Service
@Transactional(readOnly = true) // По умолчанию — только чтение
public class UserService {

    private final UserRepository userRepository;
    private final EmailService emailService;
    private final UserMapper userMapper;

    public UserService(UserRepository userRepository,
                       EmailService emailService,
                       UserMapper userMapper) {
        this.userRepository = userRepository;
        this.emailService = emailService;
        this.userMapper = userMapper;
    }

    // Бизнес-логика: получить всех пользователей
    public List<UserResponse> getAllUsers() {
        return userRepository.findAll().stream()
                .map(userMapper::toResponse)
                .toList();
    }

    // Бизнес-логика: получить пользователя с проверкой существования
    public UserResponse getUserById(Long id) {
        User user = userRepository.findById(id)
                .orElseThrow(() -> new UserNotFoundException("Пользователь с ID " + id + " не найден"));
        return userMapper.toResponse(user);
    }

    // Бизнес-логика: создание пользователя с проверками
    @Transactional // Операция записи — нужна транзакция
    public UserResponse createUser(CreateUserRequest request) {
        // Бизнес-правило: email должен быть уникальным
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new DuplicateEmailException("Email уже используется: " + request.getEmail());
        }

        User user = userMapper.toEntity(request);
        user.setCreatedAt(LocalDateTime.now());
        user.setStatus(UserStatus.ACTIVE);

        User saved = userRepository.save(user);

        // Побочный эффект: отправка приветственного письма
        emailService.sendWelcomeEmail(saved.getEmail());

        return userMapper.toResponse(saved);
    }

    // Бизнес-логика: обновление с проверкой существования
    @Transactional
    public UserResponse updateUser(Long id, UpdateUserRequest request) {
        User user = userRepository.findById(id)
                .orElseThrow(() -> new UserNotFoundException("Пользователь с ID " + id + " не найден"));

        userMapper.updateEntity(user, request);
        User updated = userRepository.save(user);

        return userMapper.toResponse(updated);
    }

    @Transactional
    public void deleteUser(Long id) {
        if (!userRepository.existsById(id)) {
            throw new UserNotFoundException("Пользователь с ID " + id + " не найден");
        }
        userRepository.deleteById(id);
    }
}
```

### Что тестировать в Service

| Аспект | Пример проверки |
|--------|----------------|
| Бизнес-правила | Дупликат email → исключение |
| Корректность вызовов | Repository вызывается с правильными параметрами |
| Обработка исключений | Несуществующий пользователь → UserNotFoundException |
| Побочные эффекты | Email отправляется после создания пользователя |
| Граничные условия | Пустой список пользователей, null-значения |
| Транзакционность | Откат при ошибке (email не отправлен → пользователь не создан?) |

---

## Repository Layer — доступ к данным

Repository отвечает за взаимодействие с базой данных. Spring Data JPA генерирует
реализации автоматически.

### Типичный Repository

```java
// Spring Data JPA автоматически создаёт реализацию CRUD-операций
@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    // Spring генерирует SQL: SELECT * FROM users WHERE email = ?
    Optional<User> findByEmail(String email);

    // Spring генерирует SQL: SELECT COUNT(*) > 0 FROM users WHERE email = ?
    boolean existsByEmail(String email);

    // Spring генерирует SQL: SELECT * FROM users WHERE status = ?
    List<User> findByStatus(UserStatus status);

    // Spring генерирует SQL: SELECT * FROM users WHERE name LIKE ?
    List<User> findByNameContainingIgnoreCase(String namePart);

    // Кастомный JPQL-запрос
    @Query("SELECT u FROM User u WHERE u.createdAt >= :date AND u.status = :status")
    List<User> findRecentActiveUsers(
            @Param("date") LocalDateTime date,
            @Param("status") UserStatus status);

    // Нативный SQL-запрос
    @Query(value = "SELECT * FROM users WHERE created_at >= NOW() - INTERVAL '30 days'",
            nativeQuery = true)
    List<User> findUsersCreatedLastMonth();
}
```

### Entity — объект, связанный с таблицей БД

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false, unique = true)
    private String email;

    @Enumerated(EnumType.STRING)
    private UserStatus status;

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    // Getters, setters, конструкторы...
}
```

### Что тестировать в Repository

| Аспект | Пример проверки |
|--------|----------------|
| CRUD-операции | Сохранение, чтение, обновление, удаление |
| Кастомные запросы | `findByEmail` возвращает правильного пользователя |
| Пустые результаты | `findByEmail` для несуществующего email → `Optional.empty()` |
| Constraints | Дупликат unique-поля → `DataIntegrityViolationException` |
| Пагинация | Правильное количество элементов на странице |
| Сортировка | Результат отсортирован по указанному полю |

---

## DTO — объекты передачи данных

DTO (Data Transfer Object) используются для передачи данных между слоями и во внешний мир.
Они отделяют внутреннее представление (Entity) от внешнего (API).

### Зачем DTO

```
Клиент ←→ DTO (UserResponse) ←→ Controller
                                     ↕
                              Service (маппинг)
                                     ↕
                              Entity (User) ←→ Repository ←→ БД
```

### Примеры DTO

```java
// Запрос на создание пользователя — только нужные поля
public record CreateUserRequest(
        @NotBlank(message = "Имя обязательно")
        String name,

        @NotBlank(message = "Email обязателен")
        @Email(message = "Невалидный формат email")
        String email,

        @Size(min = 8, message = "Пароль минимум 8 символов")
        String password
) {}

// Ответ клиенту — без пароля и внутренних полей
public record UserResponse(
        Long id,
        String name,
        String email,
        String status,
        LocalDateTime createdAt
) {}

// Запрос на обновление — все поля опциональные
public record UpdateUserRequest(
        String name,

        @Email(message = "Невалидный формат email")
        String email
) {}
```

> **Для QA**: DTO определяют контракт API. Поля DTO с аннотациями валидации (`@NotBlank`,
> `@Email`, `@Size`) — это готовые тест-кейсы для негативного тестирования.

---

## Поток запроса от начала до конца

### Пример: POST /api/users

```
1. HTTP-запрос: POST /api/users
   Body: {"name": "Иван", "email": "ivan@mail.ru", "password": "12345678"}
              │
              ▼
2. DispatcherServlet (Spring внутренний механизм)
   — Находит подходящий Controller по URL и методу
              │
              ▼
3. UserController.createUser(@Valid @RequestBody CreateUserRequest request)
   — Spring десериализует JSON → CreateUserRequest (Jackson)
   — @Valid запускает валидацию (Bean Validation)
   — Если валидация не прошла → 400 Bad Request (не доходит до Service)
              │
              ▼
4. UserService.createUser(request)
   — Проверяет бизнес-правила (уникальность email)
   — Создаёт Entity из DTO
   — Сохраняет в БД через Repository
   — Отправляет email
   — Возвращает UserResponse
              │
              ▼
5. UserRepository.save(user)
   — Hibernate генерирует INSERT SQL
   — БД сохраняет запись
   — Возвращает Entity с заполненным ID
              │
              ▼
6. Ответ: 201 Created
   Body: {"id": 42, "name": "Иван", "email": "ivan@mail.ru",
          "status": "ACTIVE", "createdAt": "2026-03-23T10:30:00"}
```

### Поток ошибки

```
POST /api/users
Body: {"name": "", "email": "не-email", "password": "123"}
              │
              ▼
Controller: @Valid → MethodArgumentNotValidException
              │
              ▼
ExceptionHandler преобразует в ответ:
{
  "status": 400,
  "errors": [
    {"field": "name", "message": "Имя обязательно"},
    {"field": "email", "message": "Невалидный формат email"},
    {"field": "password", "message": "Пароль минимум 8 символов"}
  ]
}
```

---

## Exception Handling — обработка ошибок

### Глобальный обработчик исключений

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // Обработка ошибок валидации (400 Bad Request)
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(
            MethodArgumentNotValidException ex) {

        List<FieldError> errors = ex.getBindingResult().getFieldErrors().stream()
                .map(err -> new FieldError(err.getField(), err.getDefaultMessage()))
                .toList();

        return ResponseEntity.badRequest()
                .body(new ErrorResponse("VALIDATION_ERROR", "Ошибка валидации", errors));
    }

    // Обработка "не найдено" (404 Not Found)
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(UserNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(new ErrorResponse("NOT_FOUND", ex.getMessage(), List.of()));
    }

    // Обработка дупликата (409 Conflict)
    @ExceptionHandler(DuplicateEmailException.class)
    public ResponseEntity<ErrorResponse> handleDuplicate(DuplicateEmailException ex) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
                .body(new ErrorResponse("DUPLICATE", ex.getMessage(), List.of()));
    }

    // Обработка всех остальных ошибок (500 Internal Server Error)
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(Exception ex) {
        // Логируем стектрейс, но не показываем клиенту
        log.error("Необработанная ошибка", ex);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(new ErrorResponse("INTERNAL_ERROR", "Внутренняя ошибка сервера", List.of()));
    }
}

// Структура ответа об ошибке
public record ErrorResponse(
        String code,
        String message,
        List<FieldError> errors
) {}

public record FieldError(
        String field,
        String message
) {}
```

> **Для QA**: всегда проверяйте, что API возвращает корректные коды ошибок и понятные
> сообщения. Стектрейсы не должны утекать в ответ клиенту — это уязвимость.

---

## Тестирование каждого слоя

### Пирамида тестирования для Spring-приложения

```
          ┌──────────┐
          │   E2E    │  ← Полные сценарии через HTTP (мало)
          │  тесты   │
         ┌┴──────────┴┐
         │ Интеграцион-│  ← Несколько слоёв вместе (среднее)
         │ ные тесты   │
        ┌┴────────────┴┐
        │  Юнит-тесты  │  ← Один класс в изоляции (много)
        └──────────────┘
```

### Юнит-тесты Service (без Spring)

```java
// Быстрые, изолированные, без Spring-контекста
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @Mock
    private EmailService emailService;

    @Mock
    private UserMapper userMapper;

    @InjectMocks
    private UserService userService;

    @Test
    void shouldCreateUserWhenEmailIsUnique() {
        // Arrange — подготовка данных
        CreateUserRequest request = new CreateUserRequest("Иван", "ivan@mail.ru", "password");
        User entity = new User(null, "Иван", "ivan@mail.ru", UserStatus.ACTIVE, null);
        User saved = new User(1L, "Иван", "ivan@mail.ru", UserStatus.ACTIVE, LocalDateTime.now());
        UserResponse response = new UserResponse(1L, "Иван", "ivan@mail.ru", "ACTIVE", saved.getCreatedAt());

        when(userRepository.existsByEmail("ivan@mail.ru")).thenReturn(false);
        when(userMapper.toEntity(request)).thenReturn(entity);
        when(userRepository.save(any(User.class))).thenReturn(saved);
        when(userMapper.toResponse(saved)).thenReturn(response);

        // Act — выполнение
        UserResponse result = userService.createUser(request);

        // Assert — проверка
        assertEquals("Иван", result.name());
        verify(emailService).sendWelcomeEmail("ivan@mail.ru");
        verify(userRepository).save(any(User.class));
    }

    @Test
    void shouldThrowWhenEmailAlreadyExists() {
        CreateUserRequest request = new CreateUserRequest("Иван", "ivan@mail.ru", "password");

        when(userRepository.existsByEmail("ivan@mail.ru")).thenReturn(true);

        assertThrows(DuplicateEmailException.class,
                () -> userService.createUser(request));

        // Email НЕ должен отправляться при ошибке
        verify(emailService, never()).sendWelcomeEmail(anyString());
    }
}
```

### Slice-тесты Controller (@WebMvcTest)

```java
// Поднимает только web-слой, без Service и Repository
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean // Мок Service-слоя
    private UserService userService;

    @Test
    void shouldReturnUserById() throws Exception {
        UserResponse user = new UserResponse(1L, "Иван", "ivan@mail.ru", "ACTIVE", LocalDateTime.now());
        when(userService.getUserById(1L)).thenReturn(user);

        mockMvc.perform(get("/api/users/1"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.name").value("Иван"))
                .andExpect(jsonPath("$.email").value("ivan@mail.ru"));
    }

    @Test
    void shouldReturn400ForInvalidInput() throws Exception {
        // Пустое имя и невалидный email — должна сработать валидация
        String invalidJson = """
                {
                    "name": "",
                    "email": "not-an-email",
                    "password": "123"
                }
                """;

        mockMvc.perform(post("/api/users")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(invalidJson))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.errors").isArray());
    }
}
```

### Slice-тесты Repository (@DataJpaTest)

```java
// Поднимает только JPA-слой с in-memory БД
@DataJpaTest
class UserRepositoryTest {

    @Autowired
    private UserRepository userRepository;

    @Test
    void shouldFindUserByEmail() {
        // Arrange
        User user = new User(null, "Иван", "ivan@mail.ru", UserStatus.ACTIVE, LocalDateTime.now());
        userRepository.save(user);

        // Act
        Optional<User> found = userRepository.findByEmail("ivan@mail.ru");

        // Assert
        assertTrue(found.isPresent());
        assertEquals("Иван", found.get().getName());
    }

    @Test
    void shouldReturnEmptyForNonExistentEmail() {
        Optional<User> found = userRepository.findByEmail("nobody@mail.ru");
        assertTrue(found.isEmpty());
    }
}
```

---

## Связь с тестированием

### Какой тип теста для какого слоя

| Слой | Тип теста | Аннотация | Скорость | Что моки |
|------|-----------|-----------|----------|----------|
| Controller | Slice-тест | `@WebMvcTest` | Быстро | Service |
| Service | Юнит-тест | `@ExtendWith(MockitoExtension.class)` | Очень быстро | Repository, внешние сервисы |
| Repository | Slice-тест | `@DataJpaTest` | Средне | Ничего (H2 in-memory) |
| Все слои | Интеграционный | `@SpringBootTest` | Медленно | Внешние сервисы (опционально) |

### На что обращать внимание при тестировании

1. **Controller**: валидация, HTTP-статусы, формат ответов, обработка ошибок
2. **Service**: бизнес-правила, граничные условия, взаимодействие с зависимостями
3. **Repository**: SQL-запросы, constraints, пагинация, сортировка
4. **Между слоями**: правильность маппинга DTO ↔ Entity

---

## Типичные ошибки

### 1. Бизнес-логика в Controller

```java
// ОШИБКА: Controller содержит бизнес-логику
@PostMapping
public ResponseEntity<UserResponse> createUser(@RequestBody CreateUserRequest request) {
    // Это должно быть в Service!
    if (userRepository.existsByEmail(request.getEmail())) {
        throw new DuplicateEmailException("Email уже используется");
    }
    User user = new User();
    user.setName(request.getName());
    user.setEmail(request.getEmail());
    userRepository.save(user);
    emailService.sendWelcomeEmail(user.getEmail());
    // ...
}
```

> **Для QA**: если бизнес-логика в Controller — её сложнее тестировать изолированно,
> и она может дублироваться в разных контроллерах.

### 2. Прямое использование Entity в API

```java
// ОШИБКА: Entity возвращается клиенту напрямую
@GetMapping("/{id}")
public User getUser(@PathVariable Long id) {
    return userRepository.findById(id).orElseThrow();
    // Проблема: пароль, внутренние поля утекают в API
    // Проблема: LazyInitializationException при lazy-полях
}
```

### 3. Отсутствие обработки ошибок

```java
// ОШИБКА: нет обработки — клиент получит стектрейс при ошибке
@GetMapping("/{id}")
public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
    User user = userRepository.findById(id).get(); // NoSuchElementException!
    return ResponseEntity.ok(mapper.toResponse(user));
}
```

### 4. N+1 проблема в Repository

```java
// Entity с lazy-загрузкой
@Entity
public class Order {
    @OneToMany(fetch = FetchType.LAZY)
    private List<OrderItem> items; // Загружается отдельным запросом
}

// Сервис, вызывающий N+1 запросов
public List<OrderResponse> getAllOrders() {
    List<Order> orders = orderRepository.findAll(); // 1 запрос
    return orders.stream()
            .map(order -> {
                order.getItems(); // +1 запрос для КАЖДОГО заказа!
                return mapper.toResponse(order);
            })
            .toList();
}
// Результат: 1 + N запросов вместо 1 (проблема производительности)
```

> **Для QA**: если API-эндпоинт внезапно стал медленным при росте данных — это может быть
> N+1 проблема. Проверьте через `spring.jpa.show-sql=true`.

---

## Вопросы на интервью

### 🟢 Базовый уровень

1. **Какие слои есть в типичном Spring-приложении?**
   Controller (веб-слой), Service (бизнес-логика), Repository (доступ к данным).

2. **Что делает Controller?**
   Принимает HTTP-запросы, валидирует входные данные, вызывает Service,
   формирует HTTP-ответ.

3. **Что такое DTO и зачем нужен?**
   Data Transfer Object — объект для передачи данных между слоями. Отделяет внутреннее
   представление (Entity) от внешнего API. Защищает от утечки внутренних полей.

4. **В чём разница между @Controller и @RestController?**
   `@RestController` = `@Controller` + `@ResponseBody`. Методы автоматически возвращают
   JSON/XML вместо имени view-шаблона.

### 🟡 Средний уровень

5. **Какой тип тестов подходит для каждого слоя?**
   Controller → `@WebMvcTest` (slice), Service → юнит-тесты с Mockito,
   Repository → `@DataJpaTest` (slice), все слои → `@SpringBootTest`.

6. **Как работает @RestControllerAdvice?**
   Глобальный обработчик исключений для всех контроллеров. Перехватывает исключения
   и преобразует их в HTTP-ответы с нужным статусом и телом.

7. **Почему бизнес-логика не должна быть в Controller?**
   Нарушает принцип Single Responsibility, затрудняет тестирование (нужен HTTP-контекст),
   приводит к дублированию логики между контроллерами.

8. **Как проверить, что Spring Data JPA генерирует правильный SQL?**
   Включить `spring.jpa.show-sql=true`, написать `@DataJpaTest` с проверкой результатов,
   использовать `@Query` для сложных запросов.

### 🔴 Продвинутый уровень

9. **Что такое N+1 проблема и как её обнаружить?**
   При lazy-загрузке связанных сущностей генерируется 1 запрос на основную сущность +
   N запросов на каждую связанную. Обнаруживается через анализ SQL-логов или замер
   количества запросов в тестах.

10. **Как тестировать транзакционность?**
    Проверить, что при исключении в середине операции все изменения откатываются.
    `@DataJpaTest` автоматически оборачивает тесты в транзакцию и откатывает после теста.

11. **Как устроен DispatcherServlet и как происходит маршрутизация?**
    `DispatcherServlet` — единая точка входа для всех HTTP-запросов. Он находит подходящий
    handler (Controller-метод) по URL и HTTP-методу, вызывает его, а затем преобразует
    результат в HTTP-ответ через `HttpMessageConverter`.

---

## Практические задания

### Задание 1: Определите слой

Для каждого фрагмента кода определите, к какому слою он относится (Controller, Service, Repository):

```java
// Фрагмент A
@Query("SELECT p FROM Product p WHERE p.price BETWEEN :min AND :max")
List<Product> findByPriceRange(@Param("min") BigDecimal min, @Param("max") BigDecimal max);

// Фрагмент B
@PostMapping("/orders")
public ResponseEntity<OrderResponse> placeOrder(@Valid @RequestBody OrderRequest request) {
    return ResponseEntity.status(HttpStatus.CREATED).body(orderService.placeOrder(request));
}

// Фрагмент C
public OrderResponse placeOrder(OrderRequest request) {
    if (request.getItems().isEmpty()) {
        throw new EmptyOrderException("Заказ не может быть пустым");
    }
    BigDecimal total = calculateTotal(request.getItems());
    // ...
}
```

### Задание 2: Напишите тест-кейсы

Для API `POST /api/orders` с телом `{"productId": "...", "quantity": N}` составьте список
тест-кейсов для каждого слоя:
- Controller: валидация, HTTP-статусы
- Service: бизнес-правила (товар на складе, минимальный заказ)
- Repository: сохранение, unique-constraints

### Задание 3: Найдите архитектурные нарушения

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @GetMapping
    public List<Map<String, Object>> getProducts() {
        return jdbcTemplate.queryForList("SELECT * FROM products");
    }
}
```

Перечислите все проблемы с архитектурной точки зрения.

### Задание 4: Проследите поток запроса

Опишите пошагово, что происходит при запросе `DELETE /api/users/999`,
если пользователя с ID 999 не существует. Какой HTTP-статус вернётся?

---

## Дополнительные ресурсы

- [Spring MVC Documentation](https://docs.spring.io/spring-framework/reference/web/webmvc.html) — веб-слой Spring
- [Spring Data JPA Reference](https://docs.spring.io/spring-data/jpa/reference/jpa.html) — доступ к данным
- [Baeldung: Testing in Spring Boot](https://www.baeldung.com/spring-boot-testing) — тестирование слоёв
- [Baeldung: DTO Pattern](https://www.baeldung.com/java-dto-pattern) — паттерн DTO
- [Vlad Mihalcea: N+1 Query Problem](https://vladmihalcea.com/n-plus-1-query-problem/) — N+1 проблема
