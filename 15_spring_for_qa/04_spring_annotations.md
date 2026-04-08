# Основные аннотации Spring

## Обзор

Spring-приложения активно используют аннотации для конфигурации, маршрутизации, внедрения зависимостей
и управления транзакциями. QA-инженеру не нужно знать каждую аннотацию наизусть, но понимание ключевых
аннотаций позволяет читать код приложения, понимать его поведение, находить потенциальные баги и писать
более осмысленные тесты. Этот раздел — справочник по аннотациям, которые QA встречает в коде чаще всего,
с объяснением, на что обращать внимание при тестировании.

---

## Аннотации для определения компонентов

Эти аннотации говорят Spring: «создай bean из этого класса и управляй им».

### @Component — базовая аннотация

```java
// Универсальная аннотация — Spring создаёт bean из этого класса
@Component
public class EmailValidator {

    public boolean isValid(String email) {
        return email != null && email.matches("^[\\w.-]+@[\\w.-]+\\.[a-zA-Z]{2,}$");
    }
}
```

> **Для QA**: если класс помечен `@Component` — он управляется Spring и может быть
> внедрён в другие bean через DI. Его можно заменить моком в тестах через `@MockBean`.

### @Service — сервисный слой

```java
// Семантически: этот класс содержит бизнес-логику
// Технически: то же самое, что @Component, но с ясным намерением
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final PaymentService paymentService;

    public OrderService(OrderRepository orderRepository, PaymentService paymentService) {
        this.orderRepository = orderRepository;
        this.paymentService = paymentService;
    }

    public Order placeOrder(OrderRequest request) {
        // Бизнес-логика: валидация, расчёт, сохранение
        validateStock(request);
        BigDecimal total = calculateTotal(request);
        paymentService.charge(total);
        return orderRepository.save(new Order(request, total));
    }

    private void validateStock(OrderRequest request) {
        // Проверка наличия товара на складе
    }

    private BigDecimal calculateTotal(OrderRequest request) {
        // Расчёт итоговой стоимости
        return BigDecimal.ZERO;
    }
}
```

> **Для QA**: `@Service` — здесь живут бизнес-правила. Именно эти классы содержат логику,
> которую нужно тестировать в первую очередь юнит-тестами.

### @Repository — слой доступа к данным

```java
// Семантически: доступ к данным
// Дополнительно: Spring автоматически транслирует SQL-исключения в Spring DataAccessException
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    List<Product> findByCategory(String category);

    @Query("SELECT p FROM Product p WHERE p.price < :maxPrice ORDER BY p.price ASC")
    List<Product> findCheaperThan(@Param("maxPrice") BigDecimal maxPrice);
}
```

> **Для QA**: `@Repository` оборачивает SQL-исключения. Вместо `SQLException` вы увидите
> `DataIntegrityViolationException`, `EmptyResultDataAccessException` и т.д.

### @Controller и @RestController — веб-слой

```java
// @Controller — возвращает имя view-шаблона (Thymeleaf, JSP)
@Controller
public class PageController {

    @GetMapping("/home")
    public String homePage(Model model) {
        model.addAttribute("title", "Главная страница");
        return "home"; // Ищет шаблон home.html
    }
}

// @RestController — возвращает данные напрямую (JSON/XML)
// Эквивалент: @Controller + @ResponseBody на каждом методе
@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductService productService;

    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    @GetMapping
    public List<ProductResponse> getAll() {
        return productService.getAllProducts(); // Автоматически → JSON
    }
}
```

> **Для QA**: в тестах API вы обычно тестируете `@RestController`. Для тестирования
> используйте `@WebMvcTest(ProductController.class)` — поднимает только веб-слой.

### Иерархия аннотаций компонентов

```
        @Component
       /    |     \
@Service  @Repository  @Controller
                            |
                     @RestController
```

Все они являются разновидностями `@Component`. Разница — в семантике и дополнительном
поведении (трансляция исключений у `@Repository`, `@ResponseBody` у `@RestController`).

---

## Аннотации внедрения зависимостей

### @Autowired — автоматическое внедрение

```java
@Service
public class NotificationService {

    // Field injection — Spring внедряет через рефлексию
    @Autowired
    private EmailSender emailSender;

    // Setter injection — Spring вызывает setter
    private SmsSender smsSender;

    @Autowired
    public void setSmsSender(SmsSender smsSender) {
        this.smsSender = smsSender;
    }

    // Constructor injection — @Autowired не обязателен, если конструктор один
    private final PushSender pushSender;

    public NotificationService(PushSender pushSender) {
        this.pushSender = pushSender;
    }
}
```

### Когда @Autowired не находит bean

```java
@Service
public class ReportService {

    @Autowired
    private PdfGenerator pdfGenerator; // Если PdfGenerator не является bean → ошибка!
}
// Ошибка при старте:
// NoSuchBeanDefinitionException: No qualifying bean of type 'PdfGenerator'
```

> **Для QA**: если приложение не запускается с `NoSuchBeanDefinitionException` — это значит,
> что Spring не нашёл реализацию для какого-то интерфейса или класса. Проверьте, есть ли
> `@Component`/`@Service` на нужном классе.

### @Qualifier — выбор конкретной реализации

```java
// Два bean реализуют один интерфейс — Spring не знает, какой выбрать
@Service("emailNotifier")
public class EmailNotificationService implements NotificationService { }

@Service("smsNotifier")
public class SmsNotificationService implements NotificationService { }

@Service
public class OrderService {

    // @Qualifier указывает, какой именно bean внедрить
    @Autowired
    @Qualifier("emailNotifier")
    private NotificationService notificationService;
}
```

> **Для QA**: если видите `NoUniqueBeanDefinitionException` — Spring нашёл несколько
> реализаций одного интерфейса и не знает, какую использовать. Нужен `@Qualifier` или `@Primary`.

### @Value — внедрение значений из конфигурации

```java
@Service
public class PaymentService {

    // Значение из application.yml, значение по умолчанию — 3
    @Value("${payment.retry.max-attempts:3}")
    private int maxRetryAttempts;

    // Секрет из переменной окружения
    @Value("${payment.api.key}")
    private String apiKey;

    // Статическое значение (редко, но бывает)
    @Value("#{T(java.time.ZoneId).of('UTC')}")
    private ZoneId timeZone;

    public PaymentResult processPayment(BigDecimal amount) {
        for (int attempt = 1; attempt <= maxRetryAttempts; attempt++) {
            try {
                return callPaymentApi(amount, apiKey);
            } catch (PaymentException e) {
                if (attempt == maxRetryAttempts) throw e;
            }
        }
        throw new PaymentException("Все попытки исчерпаны");
    }

    private PaymentResult callPaymentApi(BigDecimal amount, String key) {
        // Вызов внешнего API
        return new PaymentResult(true);
    }
}
```

> **Для QA**: `@Value` определяет, какие настройки влияют на поведение сервиса. Если
> `payment.retry.max-attempts=1` на тестовом стенде, а на production — `5`, поведение
> при сбоях будет разным. Всегда проверяйте конфигурацию тестового окружения.

---

## Аннотации веб-запросов

### @RequestMapping — базовая маршрутизация

```java
@RestController
@RequestMapping("/api/v1/orders") // Общий префикс для всех методов
public class OrderController {

    // Полная форма — указываем метод, путь, формат
    @RequestMapping(method = RequestMethod.GET, produces = "application/json")
    public List<OrderResponse> getAll() {
        return orderService.getAllOrders();
    }
}
```

### Специализированные аннотации маршрутизации

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    // GET /api/users — получить список
    @GetMapping
    public List<UserResponse> getAll() {
        return userService.getAllUsers();
    }

    // GET /api/users/42 — получить одного
    @GetMapping("/{id}")
    public UserResponse getById(@PathVariable Long id) {
        return userService.getUserById(id);
    }

    // POST /api/users — создать
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED) // Явно указываем статус 201
    public UserResponse create(@Valid @RequestBody CreateUserRequest request) {
        return userService.createUser(request);
    }

    // PUT /api/users/42 — полное обновление
    @PutMapping("/{id}")
    public UserResponse update(
            @PathVariable Long id,
            @Valid @RequestBody UpdateUserRequest request) {
        return userService.updateUser(id, request);
    }

    // PATCH /api/users/42 — частичное обновление
    @PatchMapping("/{id}")
    public UserResponse partialUpdate(
            @PathVariable Long id,
            @RequestBody Map<String, Object> updates) {
        return userService.partialUpdate(id, updates);
    }

    // DELETE /api/users/42 — удалить
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT) // 204 — без тела
    public void delete(@PathVariable Long id) {
        userService.deleteUser(id);
    }
}
```

> **Для QA**: маршрутизация определяет контракт API. Из аннотаций можно извлечь:
> HTTP-метод, URL, ожидаемый статус ответа. Это основа для тест-кейсов API.

---

## Аннотации параметров запроса

### @PathVariable — параметр из URL

```java
// URL: /api/users/42/orders/7
@GetMapping("/api/users/{userId}/orders/{orderId}")
public OrderResponse getOrder(
        @PathVariable Long userId,    // userId = 42
        @PathVariable Long orderId) { // orderId = 7
    return orderService.getOrder(userId, orderId);
}
```

**Тест-кейсы для @PathVariable:**
- Валидный ID: `/api/users/1` → 200
- Несуществующий ID: `/api/users/999999` → 404
- Невалидный тип: `/api/users/abc` → 400
- Отрицательный ID: `/api/users/-1` → 400 или 404
- Ноль: `/api/users/0` → зависит от бизнес-логики

### @RequestParam — параметр из query string

```java
// URL: /api/products?category=electronics&minPrice=100&page=0&size=20
@GetMapping("/api/products")
public Page<ProductResponse> searchProducts(
        @RequestParam String category,                       // Обязательный
        @RequestParam(required = false) BigDecimal minPrice, // Необязательный
        @RequestParam(defaultValue = "0") int page,          // Со значением по умолчанию
        @RequestParam(defaultValue = "20") int size) {       // Со значением по умолчанию
    return productService.search(category, minPrice, page, size);
}
```

**Тест-кейсы для @RequestParam:**
- Все параметры: `/api/products?category=electronics&minPrice=100` → 200
- Без обязательного: `/api/products?minPrice=100` → 400 (category обязателен)
- Без опционального: `/api/products?category=electronics` → 200 (minPrice = null)
- Пустое значение: `/api/products?category=` → зависит от валидации
- Невалидный тип: `/api/products?category=electronics&minPrice=abc` → 400

### @RequestBody — тело запроса

```java
// POST /api/users
// Body: {"name": "Иван", "email": "ivan@mail.ru"}
@PostMapping("/api/users")
public UserResponse createUser(
        @Valid                    // Запускает валидацию
        @RequestBody              // Десериализует JSON → объект
        CreateUserRequest request) {
    return userService.createUser(request);
}
```

**Тест-кейсы для @RequestBody:**
- Валидный JSON → 201
- Пустое тело → 400
- Невалидный JSON (синтаксическая ошибка) → 400
- Лишние поля → обычно игнорируются (Jackson по умолчанию)
- Отсутствующие обязательные поля → 400 (если есть @Valid)
- Неправильные типы полей → 400

### @RequestHeader — заголовок запроса

```java
@GetMapping("/api/data")
public DataResponse getData(
        @RequestHeader("Authorization") String authToken,
        @RequestHeader(value = "X-Request-Id", required = false) String requestId) {
    return dataService.getData(authToken, requestId);
}
```

---

## Аннотации валидации

Spring интегрирован с Bean Validation (Jakarta Validation). Аннотации на полях DTO
автоматически проверяются при `@Valid`.

### Основные аннотации валидации

```java
public record CreateProductRequest(

        @NotBlank(message = "Название обязательно")
        @Size(min = 2, max = 100, message = "Название: от 2 до 100 символов")
        String name,

        @NotNull(message = "Цена обязательна")
        @Positive(message = "Цена должна быть положительной")
        @DecimalMax(value = "999999.99", message = "Цена не может превышать 999999.99")
        BigDecimal price,

        @NotBlank(message = "Категория обязательна")
        String category,

        @Size(max = 500, message = "Описание: максимум 500 символов")
        String description,

        @Min(value = 0, message = "Количество не может быть отрицательным")
        int quantity,

        @Email(message = "Невалидный email поставщика")
        String supplierEmail,

        @Pattern(regexp = "^[A-Z]{2}-\\d{4}$", message = "Код в формате XX-0000")
        String productCode
) {}
```

> **Для QA**: аннотации валидации — это **готовые тест-кейсы**. Каждая аннотация подсказывает,
> какой невалидный ввод нужно проверить. `@NotBlank` → пустая строка, `@Positive` → ноль
> и отрицательные числа, `@Email` → строка без `@`, `@Pattern` → строка не по формату.

### Таблица: аннотации валидации → тест-кейсы

| Аннотация | Что проверяет | Невалидные значения для тестирования |
|-----------|--------------|--------------------------------------|
| `@NotNull` | Не null | `null` |
| `@NotBlank` | Не null, не пустая, не только пробелы | `null`, `""`, `"   "` |
| `@NotEmpty` | Не null и не пустая (для коллекций тоже) | `null`, `""`, `[]` |
| `@Size(min, max)` | Длина строки или размер коллекции | Строка < min, строка > max |
| `@Min(value)` | Число >= value | Число < value |
| `@Max(value)` | Число <= value | Число > value |
| `@Positive` | Число > 0 | `0`, `-1` |
| `@PositiveOrZero` | Число >= 0 | `-1` |
| `@Email` | Формат email | `"abc"`, `"@mail"`, `"a@"` |
| `@Pattern(regexp)` | Соответствие регулярному выражению | Строка, не соответствующая паттерну |
| `@Past` | Дата в прошлом | Текущая дата, будущая дата |
| `@Future` | Дата в будущем | Текущая дата, прошлая дата |

---

## @Transactional — управление транзакциями

### Как работает @Transactional

```java
@Service
public class TransferService {

    private final AccountRepository accountRepository;

    public TransferService(AccountRepository accountRepository) {
        this.accountRepository = accountRepository;
    }

    // Spring оборачивает метод в транзакцию:
    // BEGIN TRANSACTION → выполнение метода → COMMIT (или ROLLBACK при исключении)
    @Transactional
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        Account from = accountRepository.findById(fromId)
                .orElseThrow(() -> new AccountNotFoundException("Счёт не найден: " + fromId));
        Account to = accountRepository.findById(toId)
                .orElseThrow(() -> new AccountNotFoundException("Счёт не найден: " + toId));

        if (from.getBalance().compareTo(amount) < 0) {
            throw new InsufficientFundsException("Недостаточно средств");
        }

        from.setBalance(from.getBalance().subtract(amount)); // Списание
        to.setBalance(to.getBalance().add(amount));           // Зачисление

        accountRepository.save(from);
        accountRepository.save(to);
        // Если здесь произойдёт исключение — ОБЕ операции откатятся
    }

    // readOnly = true — оптимизация для запросов на чтение
    @Transactional(readOnly = true)
    public AccountResponse getAccount(Long id) {
        return accountRepository.findById(id)
                .map(this::toResponse)
                .orElseThrow();
    }

    private AccountResponse toResponse(Account account) {
        return new AccountResponse(account.getId(), account.getBalance());
    }
}
```

> **Для QA**: `@Transactional` — критически важная аннотация. Если при переводе денег
> произойдёт ошибка после списания, но до зачисления — деньги не должны «исчезнуть».
> Тестируйте сценарии с ошибками посередине транзакции.

### Типичные ловушки @Transactional

```java
// ЛОВУШКА 1: @Transactional на private-методе — НЕ РАБОТАЕТ!
@Service
public class OrderService {

    @Transactional // Игнорируется! Прокси не перехватывает private-методы
    private void processOrder(Order order) {
        // Транзакция НЕ создаётся
    }
}

// ЛОВУШКА 2: Вызов транзакционного метода из того же класса — НЕ РАБОТАЕТ!
@Service
public class PaymentService {

    public void processPayment(Payment payment) {
        // Вызов внутри того же класса — обходит прокси
        this.savePayment(payment); // @Transactional НЕ сработает!
    }

    @Transactional
    public void savePayment(Payment payment) {
        // Транзакция не создаётся при вызове через this
    }
}

// ЛОВУШКА 3: Checked exception не вызывает откат по умолчанию
@Transactional // По умолчанию откат только при RuntimeException
public void riskyOperation() throws BusinessException {
    // ...
    throw new BusinessException("Ошибка"); // НЕ вызовет откат!
}

// Правильно: явно указываем rollback для checked exception
@Transactional(rollbackFor = BusinessException.class)
public void riskyOperation() throws BusinessException {
    // Теперь откат произойдёт при BusinessException
}
```

---

## @Configuration и @Bean

### Явное создание bean

```java
// @Configuration отмечает класс как источник определений bean
@Configuration
public class HttpClientConfig {

    // @Bean — метод создаёт и возвращает bean
    @Bean
    public RestTemplate restTemplate() {
        RestTemplate template = new RestTemplate();
        // Настраиваем таймауты
        HttpComponentsClientHttpRequestFactory factory =
                new HttpComponentsClientHttpRequestFactory();
        factory.setConnectTimeout(Duration.ofSeconds(5));
        factory.setReadTimeout(Duration.ofSeconds(10));
        template.setRequestFactory(factory);
        return template;
    }

    // Bean с профилем — создаётся только для конкретного окружения
    @Bean
    @Profile("prod")
    public RestTemplate secureRestTemplate(SSLContext sslContext) {
        // Настройка SSL для production
        return new RestTemplate();
    }
}
```

> **Для QA**: `@Configuration` классы определяют инфраструктуру приложения. Таймауты,
> подключения, клиенты — всё здесь. Если тест «виснет» при обращении к внешнему сервису —
> проверьте таймауты в `@Configuration`.

---

## Сводная таблица аннотаций

| Аннотация | Слой | Назначение | Важно для QA |
|-----------|------|-----------|-------------|
| `@Component` | Любой | Базовый bean | Компонент управляется Spring |
| `@Service` | Service | Бизнес-логика | Здесь живут правила для тестирования |
| `@Repository` | Data | Доступ к данным | Трансляция SQL-исключений |
| `@Controller` | Web | HTTP + view | Рендеринг страниц |
| `@RestController` | Web | HTTP + JSON | REST API — основной объект тестирования |
| `@Autowired` | Любой | Внедрение зависимости | Можно заменить @MockBean |
| `@Value` | Любой | Значение из properties | Поведение зависит от конфигурации |
| `@Transactional` | Service | Транзакция | Откат при ошибках |
| `@GetMapping` | Web | GET-запрос | URL и метод для тест-кейса |
| `@PostMapping` | Web | POST-запрос | URL, метод, тело для тест-кейса |
| `@PutMapping` | Web | PUT-запрос | URL, метод, тело для тест-кейса |
| `@DeleteMapping` | Web | DELETE-запрос | URL и метод для тест-кейса |
| `@PathVariable` | Web | Параметр из URL | Тесты с разными ID |
| `@RequestParam` | Web | Query-параметр | Тесты обязательных/необязательных |
| `@RequestBody` | Web | Тело запроса (JSON) | Тесты с разным телом |
| `@Valid` | Web | Активирует валидацию | Все аннотации на DTO — тест-кейсы |
| `@Configuration` | Config | Конфигурация bean | Таймауты, подключения |
| `@Bean` | Config | Создание bean | Инфраструктурные настройки |

---

## Связь с тестированием

### Чтение кода через аннотации: пошаговый алгоритм

1. **Найдите `@RestController`** — определите все эндпоинты API
2. **Прочитайте `@GetMapping`, `@PostMapping`** — составьте список URL
3. **Посмотрите на `@Valid` и аннотации на DTO** — определите правила валидации
4. **Найдите `@Service`** — поймите бизнес-логику
5. **Проверьте `@Transactional`** — поймите, что атомарно
6. **Найдите `@Value`** — поймите, какие настройки влияют на поведение

### Пример: извлечение тест-кейсов из аннотаций

```java
// Из этого кода можно извлечь минимум 15 тест-кейсов
@PostMapping("/api/orders")
@ResponseStatus(HttpStatus.CREATED)
public OrderResponse createOrder(
        @Valid @RequestBody CreateOrderRequest request) {
    return orderService.createOrder(request);
}

public record CreateOrderRequest(
        @NotBlank String productId,        // TC: null, "", "   "
        @Min(1) @Max(100) int quantity,    // TC: 0, 1, 100, 101, -1
        @NotNull @Future LocalDate deliveryDate // TC: null, прошлая, сегодня, будущая
) {}
```

---

## Типичные ошибки

### 1. Забытая @Valid

```java
// ОШИБКА: @Valid отсутствует — валидация DTO не выполняется!
@PostMapping("/api/users")
public UserResponse createUser(@RequestBody CreateUserRequest request) {
    // @NotBlank, @Email и другие аннотации на DTO игнорируются
    return userService.createUser(request);
}
```

### 2. @Transactional на private-методе

```java
@Transactional // Не сработает — Spring создаёт прокси только для public-методов
private void saveData(Data data) {
    repository.save(data);
}
```

### 3. Field injection затрудняет тестирование

```java
@Service
public class ReportService {
    @Autowired
    private DataFetcher dataFetcher; // Нельзя передать мок через конструктор

    // Для юнит-теста придётся использовать ReflectionTestUtils
}
```

### 4. @Value без значения по умолчанию

```java
@Value("${external.api.url}") // Если в properties нет — приложение не запустится
private String apiUrl;

@Value("${external.api.url:http://localhost:8080}") // С default — безопаснее
private String apiUrlSafe;
```

---

## Вопросы на интервью

### 🟢 Базовый уровень

1. **В чём разница между @Component, @Service, @Repository?**
   Все создают bean. `@Service` — для бизнес-логики. `@Repository` — для доступа к данным
   (+ трансляция исключений). `@Component` — для всего остального.

2. **Что делает @Autowired?**
   Автоматически внедряет зависимость. Spring находит подходящий bean и устанавливает его
   в поле, setter или конструктор.

3. **В чём разница между @PathVariable и @RequestParam?**
   `@PathVariable` — из URL-пути (`/users/42`). `@RequestParam` — из query-строки
   (`/users?name=Ivan`).

4. **Что делает @Valid?**
   Активирует валидацию аннотаций на полях DTO (`@NotBlank`, `@Email`, `@Min` и т.д.).
   Без `@Valid` эти аннотации игнорируются.

### 🟡 Средний уровень

5. **Как @Transactional влияет на тестирование?**
   В тестах `@DataJpaTest` автоматически оборачивает тесты в транзакцию с откатом.
   Для тестирования транзакционности Service нужно проверять откат при исключениях.

6. **Почему @Transactional не работает при self-invocation?**
   Spring создаёт прокси вокруг bean. Вызов метода через `this` обходит прокси,
   и транзакция не создаётся.

7. **Как @Value помогает в тестировании?**
   Позволяет переопределить поведение через тестовый `application-test.yml`.
   Например, отключить отправку email, изменить URL внешнего сервиса.

8. **Какие тест-кейсы можно извлечь из аннотаций валидации?**
   Каждая аннотация определяет граничное условие. `@NotBlank` → null, пустая, пробелы.
   `@Min(1)` → 0, 1, -1. `@Email` → без @, без домена.

### 🔴 Продвинутый уровень

9. **Как работает прокси в Spring и на что это влияет?**
   Spring создаёт прокси (JDK Dynamic Proxy или CGLIB) вокруг bean. Прокси перехватывает
   вызовы и добавляет функциональность (@Transactional, @Cacheable). Private-методы и
   self-invocation не перехватываются.

10. **Что происходит, если два bean реализуют один интерфейс без @Qualifier?**
    `NoUniqueBeanDefinitionException` при запуске. Решение: `@Qualifier`, `@Primary`,
    или `@ConditionalOnProperty`.

11. **Как @Transactional(readOnly = true) влияет на производительность?**
    Hibernate не выполняет dirty-checking, не создаёт snapshot entity, может использовать
    read-only реплику БД. Ускоряет запросы на чтение.

---

## Практические задания

### Задание 1: Извлеките тест-кейсы

Из следующего Controller извлеките все возможные тест-кейсы (позитивные и негативные):

```java
@RestController
@RequestMapping("/api/bookings")
public class BookingController {

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public BookingResponse createBooking(
            @Valid @RequestBody BookingRequest request) {
        return bookingService.create(request);
    }
}

public record BookingRequest(
        @NotNull Long roomId,
        @NotNull @FutureOrPresent LocalDate checkIn,
        @NotNull @Future LocalDate checkOut,
        @Min(1) @Max(4) int guests,
        @Email String contactEmail
) {}
```

### Задание 2: Найдите проблемы

Найдите все проблемы с аннотациями:

```java
@Controller
@RequestMapping("/api/data")
public class DataController {

    @Autowired
    private DataService dataService;

    @GetMapping
    public List<DataResponse> getAll() {
        return dataService.getAll();
    }

    @PostMapping
    public DataResponse create(@RequestBody DataRequest request) {
        return dataService.create(request);
    }
}
```

### Задание 3: Определите поведение

Что произойдёт при вызове `processAndNotify()`:

```java
@Service
public class OrderProcessor {

    @Transactional
    public void processAndNotify(Long orderId) {
        updateOrderStatus(orderId, "PROCESSED");
        this.sendNotification(orderId); // self-invocation
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendNotification(Long orderId) {
        // Должна быть отдельная транзакция, но будет ли?
    }

    private void updateOrderStatus(Long orderId, String status) {
        // Обновление статуса в БД
    }
}
```

---

## Дополнительные ресурсы

- [Spring Annotations Reference](https://docs.spring.io/spring-framework/reference/core/beans/classpath-scanning.html) — аннотации компонентов
- [Spring Web MVC Annotations](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller.html) — аннотации контроллеров
- [Bean Validation Reference](https://jakarta.ee/specifications/bean-validation/3.0/) — спецификация валидации
- [Baeldung: Spring Annotations](https://www.baeldung.com/spring-core-annotations) — обзор основных аннотаций
- [Baeldung: @Transactional](https://www.baeldung.com/transaction-configuration-with-jpa-and-spring) — транзакции в деталях
