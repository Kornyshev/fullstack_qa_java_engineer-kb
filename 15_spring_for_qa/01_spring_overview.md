# Spring Framework — обзор для QA

## Обзор

Spring Framework — это основа большинства современных Java-приложений. Как QA-инженер, вы почти наверняка
тестируете приложения, построенные на Spring. Понимание ключевых концепций — IoC (Inversion of Control),
DI (Dependency Injection), ApplicationContext, жизненный цикл bean — позволяет эффективнее тестировать,
быстрее находить причины багов и лучше взаимодействовать с разработчиками. Этот раздел не учит строить
Spring-приложения, а объясняет, как устроено приложение, которое вы тестируете.

---

## Что такое Spring Framework

Spring — это фреймворк для разработки enterprise-приложений на Java. Его главная идея — **управление
зависимостями** между компонентами приложения. Вместо того чтобы каждый класс сам создавал свои зависимости,
Spring берёт эту задачу на себя.

### Почему QA должен знать Spring

| Причина | Как помогает |
|---------|-------------|
| Понимание структуры приложения | Быстрее находите нужный код и причины багов |
| Чтение логов | Логи Spring содержат информацию о bean, context, profiles |
| Конфигурирование тестов | Spring Test предоставляет мощные инструменты |
| Общение с разработчиками | Говорите на одном языке с командой |
| Понимание окружений | Профили Spring определяют поведение на разных стендах |

---

## Inversion of Control (IoC)

**IoC (Inversion of Control)** — принцип, при котором управление созданием объектов и их зависимостями
передаётся фреймворку, а не остаётся в коде приложения.

### Без IoC (традиционный подход)

```java
// Каждый класс сам создаёт свои зависимости — жёсткая связанность
public class OrderService {

    // Сервис сам решает, какую реализацию использовать
    private final OrderRepository repository = new OrderRepositoryImpl();
    private final NotificationService notifier = new EmailNotificationService();

    public Order createOrder(OrderRequest request) {
        Order order = new Order(request);
        repository.save(order);
        notifier.sendNotification(order);
        return order;
    }
}
```

**Проблема для тестирования**: невозможно подменить `OrderRepositoryImpl` на мок. Тест всегда будет
обращаться к реальной базе данных.

### С IoC (Spring-подход)

```java
// Spring управляет созданием и внедрением зависимостей
@Service
public class OrderService {

    // Зависимости внедряются извне — Spring решает, что подставить
    private final OrderRepository repository;
    private final NotificationService notifier;

    // Constructor injection — Spring передаёт нужные реализации
    public OrderService(OrderRepository repository, NotificationService notifier) {
        this.repository = repository;
        this.notifier = notifier;
    }

    public Order createOrder(OrderRequest request) {
        Order order = new Order(request);
        repository.save(order);
        notifier.sendNotification(order);
        return order;
    }
}
```

**Выигрыш для тестирования**: в тесте можно подставить мок вместо реальной реализации.

```java
@Test
void shouldCreateOrderAndNotify() {
    // Создаём моки — подменяем реальные зависимости
    OrderRepository mockRepo = Mockito.mock(OrderRepository.class);
    NotificationService mockNotifier = Mockito.mock(NotificationService.class);

    // Внедряем моки через конструктор — именно так работает IoC
    OrderService service = new OrderService(mockRepo, mockNotifier);

    OrderRequest request = new OrderRequest("item-1", 2);
    service.createOrder(request);

    // Проверяем, что зависимости были вызваны
    verify(mockRepo).save(any(Order.class));
    verify(mockNotifier).sendNotification(any(Order.class));
}
```

---

## Dependency Injection (DI)

**DI (Dependency Injection)** — конкретная реализация принципа IoC. Spring «внедряет» (inject)
зависимости в объект тремя способами:

### 1. Constructor Injection (рекомендуемый)

```java
@Service
public class UserService {

    private final UserRepository userRepository;

    // Spring автоматически передаёт UserRepository через конструктор
    // Если конструктор один — @Autowired не обязателен (Spring 4.3+)
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

### 2. Field Injection (через аннотацию @Autowired)

```java
@Service
public class UserService {

    // Spring напрямую устанавливает значение поля через рефлексию
    @Autowired
    private UserRepository userRepository;
}
```

> **Для QA**: field injection затрудняет юнит-тестирование — нельзя передать мок через конструктор.
> Если видите `@Autowired` на полях, для теста потребуется `ReflectionTestUtils` или Spring-контекст.

### 3. Setter Injection

```java
@Service
public class UserService {

    private UserRepository userRepository;

    @Autowired
    public void setUserRepository(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

### Сравнение подходов с точки зрения тестирования

| Тип DI | Тестируемость | Обязательность зависимости | Рекомендация |
|--------|---------------|---------------------------|-------------|
| Constructor | Отличная — мок передаётся через конструктор | Да (final поля) | Предпочтительный |
| Field | Плохая — нужен Spring-контекст или рефлексия | Нет (можно забыть) | Не рекомендуется |
| Setter | Средняя — можно вызвать setter | Нет (nullable) | Для опциональных зависимостей |

---

## ApplicationContext

**ApplicationContext** — это IoC-контейнер Spring, главный объект, который управляет всеми bean
(компонентами) приложения.

### Что делает ApplicationContext

```
┌─────────────────────────────────────────────────┐
│              ApplicationContext                  │
│                                                 │
│  ┌─────────────┐  ┌─────────────────────────┐   │
│  │ UserService  │  │ OrderService            │   │
│  │  (bean)      │──│  (bean)                 │   │
│  └─────────────┘  └─────────────────────────┘   │
│         │                    │                   │
│  ┌─────────────┐  ┌─────────────────────────┐   │
│  │UserRepository│  │ OrderRepository         │   │
│  │  (bean)      │  │  (bean)                 │   │
│  └─────────────┘  └─────────────────────────┘   │
│                                                 │
│  + Конфигурация (properties, profiles)          │
│  + Event System                                 │
│  + AOP Proxies                                  │
└─────────────────────────────────────────────────┘
```

### Как QA сталкивается с ApplicationContext

```java
// В тестах Spring контекст поднимается автоматически
@SpringBootTest
class OrderServiceIntegrationTest {

    // Spring внедряет bean из контекста в тестовый класс
    @Autowired
    private OrderService orderService;

    @Test
    void contextLoads() {
        // Если контекст не поднялся — тест упадёт здесь
        assertNotNull(orderService);
    }
}
```

> **Важно для QA**: если при запуске тестов вы видите ошибку
> `Failed to load ApplicationContext` — это значит, что Spring не смог создать все необходимые bean.
> Чаще всего причина — отсутствующая конфигурация, циклическая зависимость или проблема с БД.

---

## Bean и его жизненный цикл

**Bean** — это объект, который создаётся и управляется Spring-контейнером. По сути, это обычный
Java-объект, но его жизненным циклом управляет Spring.

### Жизненный цикл bean

```
  Создание экземпляра (Instantiation)
              │
              ▼
  Внедрение зависимостей (Dependency Injection)
              │
              ▼
  @PostConstruct метод (Инициализация)
              │
              ▼
  Bean готов к использованию
              │
              ▼
  @PreDestroy метод (Очистка ресурсов)
              │
              ▼
  Уничтожение bean
```

### Пример жизненного цикла

```java
@Service
public class CacheService {

    private final Map<String, Object> cache = new ConcurrentHashMap<>();

    // Вызывается после внедрения всех зависимостей
    @PostConstruct
    public void init() {
        System.out.println("Кэш инициализирован, загружаем начальные данные...");
        loadInitialData();
    }

    // Вызывается перед уничтожением bean
    @PreDestroy
    public void cleanup() {
        System.out.println("Очищаем кэш перед завершением...");
        cache.clear();
    }

    private void loadInitialData() {
        // Загрузка данных при старте приложения
    }
}
```

> **Для QA**: если баг воспроизводится только при старте приложения или только при остановке — проверьте
> методы `@PostConstruct` и `@PreDestroy`.

### Scope (область видимости) bean

| Scope | Описание | Когда создаётся | Типичный пример |
|-------|----------|-----------------|-----------------|
| `singleton` (по умолчанию) | Один экземпляр на весь контекст | При старте приложения | Service, Repository |
| `prototype` | Новый экземпляр при каждом запросе | При каждом вызове `getBean()` | Объекты с состоянием |
| `request` | Один экземпляр на HTTP-запрос | При каждом HTTP-запросе | Данные текущего запроса |
| `session` | Один экземпляр на HTTP-сессию | При создании сессии | Данные пользовательской сессии |

```java
// Singleton — один экземпляр на всё приложение (по умолчанию)
@Service
public class UserService {
    // Все потоки используют один и тот же экземпляр!
    // Ошибка: хранить состояние в полях singleton — источник race condition
}

// Prototype — новый экземпляр каждый раз
@Component
@Scope("prototype")
public class ReportGenerator {
    private final List<String> lines = new ArrayList<>();
    // Безопасно хранить состояние — каждый получает свой экземпляр
}
```

> **Для QA**: баг, связанный с singleton, может проявляться только при параллельном выполнении
> запросов. Если состояние «утекает» между запросами — проверьте, не хранит ли singleton
> данные в полях.

---

## Конфигурация Spring

Spring поддерживает три способа конфигурации. Современные приложения обычно используют аннотации
и Java-конфигурацию.

### 1. Аннотации (основной способ)

```java
// Spring автоматически находит этот класс и создаёт bean
@Service
public class PaymentService {
    // ...
}
```

### 2. Java-конфигурация (@Configuration + @Bean)

```java
@Configuration
public class AppConfig {

    // Явно создаём bean — используется, когда нужна настройка
    @Bean
    public RestTemplate restTemplate() {
        RestTemplate template = new RestTemplate();
        template.setRequestFactory(new HttpComponentsClientHttpRequestFactory());
        return template;
    }

    // Bean с зависимостью на другой bean
    @Bean
    public PaymentClient paymentClient(RestTemplate restTemplate) {
        return new PaymentClient(restTemplate);
    }
}
```

### 3. Тестовая конфигурация

```java
// Конфигурация, которая используется только в тестах
@TestConfiguration
public class TestConfig {

    // Подменяем реальный клиент на заглушку для тестов
    @Bean
    public PaymentClient paymentClient() {
        return new FakePaymentClient();
    }
}

@SpringBootTest
@Import(TestConfig.class) // Подключаем тестовую конфигурацию
class PaymentServiceTest {

    @Autowired
    private PaymentService paymentService;

    @Test
    void shouldProcessPayment() {
        // paymentService использует FakePaymentClient вместо реального
        PaymentResult result = paymentService.processPayment(new PaymentRequest(100));
        assertEquals(PaymentStatus.SUCCESS, result.getStatus());
    }
}
```

---

## Связь с тестированием

### Что QA должен понимать об IoC/DI для тестирования

1. **Подмена зависимостей (mocking)** — главное преимущество DI для тестирования. Благодаря IoC
   можно заменить реальные сервисы на моки, не трогая production-код.

2. **Тестовый контекст** — Spring создаёт отдельный ApplicationContext для тестов. Он может
   кэшироваться между тестовыми классами, если конфигурация одинаковая.

3. **Профили** — через `@ActiveProfiles("test")` можно активировать тестовую конфигурацию.

```java
@SpringBootTest
@ActiveProfiles("test") // Активирует application-test.yml
class UserServiceTest {

    @MockBean // Подменяет реальный bean на Mockito-мок в контексте
    private EmailService emailService;

    @Autowired
    private UserService userService;

    @Test
    void shouldRegisterUserWithoutSendingRealEmail() {
        // emailService — мок, реальный email не отправится
        when(emailService.sendWelcomeEmail(anyString()))
                .thenReturn(true);

        userService.registerUser("test@example.com", "password");

        verify(emailService).sendWelcomeEmail("test@example.com");
    }
}
```

### Типичные ошибки контекста в тестах

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `NoSuchBeanDefinitionException` | Bean не найден в контексте | Проверьте `@ComponentScan`, добавьте `@Import` |
| `UnsatisfiedDependencyException` | Зависимость не может быть внедрена | Добавьте `@MockBean` или тестовую конфигурацию |
| `BeanCurrentlyInCreationException` | Циклическая зависимость | Рефакторинг — вынести общую логику |
| `ApplicationContext failure` | Ошибка при инициализации | Проверьте properties, подключение к БД |

---

## Типичные ошибки

### 1. Непонимание singleton-scope

```java
// ОШИБКА: хранение состояния в singleton
@Service
public class RequestCounter {
    private int count = 0; // Общий для всех потоков!

    public void increment() {
        count++; // Race condition при параллельных запросах
    }
}
```

### 2. Циклическая зависимость

```java
// ServiceA зависит от ServiceB, а ServiceB зависит от ServiceA
@Service
public class ServiceA {
    public ServiceA(ServiceB serviceB) { } // Нужен ServiceB
}

@Service
public class ServiceB {
    public ServiceB(ServiceA serviceA) { } // Нужен ServiceA — цикл!
}
// Результат: BeanCurrentlyInCreationException при старте
```

### 3. Использование new вместо DI

```java
@Service
public class ReportService {
    public Report generate() {
        // ОШИБКА: создание через new — Spring не управляет этим объектом
        DataService dataService = new DataService();
        // @Transactional, @Cacheable и другие аннотации НЕ работают
        return dataService.fetchData();
    }
}
```

### 4. Тяжёлый тестовый контекст

```java
// ОШИБКА: полный контекст для юнит-теста — медленно и избыточно
@SpringBootTest // Поднимает ВСЕ bean — ненужная трата времени
class CalculatorServiceTest {

    @Autowired
    private CalculatorService calculatorService;

    @Test
    void shouldAdd() {
        assertEquals(4, calculatorService.add(2, 2));
    }
}

// ПРАВИЛЬНО: простой юнит-тест без Spring
class CalculatorServiceTest {

    private final CalculatorService calculatorService = new CalculatorService();

    @Test
    void shouldAdd() {
        assertEquals(4, calculatorService.add(2, 2));
    }
}
```

---

## Вопросы на интервью

### 🟢 Базовый уровень

1. **Что такое Spring Framework и зачем он нужен?**
   Фреймворк для разработки Java-приложений. Предоставляет IoC-контейнер, который управляет
   зависимостями и жизненным циклом объектов.

2. **Что такое IoC и DI? В чём разница?**
   IoC — принцип: управление передаётся фреймворку. DI — конкретный механизм реализации IoC,
   при котором зависимости «внедряются» извне.

3. **Что такое bean в Spring?**
   Объект, созданный и управляемый Spring-контейнером. По умолчанию — singleton.

4. **Какие способы внедрения зависимостей существуют?**
   Constructor injection (рекомендуемый), field injection (`@Autowired`), setter injection.

### 🟡 Средний уровень

5. **Что такое ApplicationContext и BeanFactory?**
   `BeanFactory` — базовый контейнер, создающий bean. `ApplicationContext` — расширенный контейнер
   с поддержкой событий, интернационализации, загрузки ресурсов. В тестах обычно используется
   `ApplicationContext`.

6. **В чём разница между singleton и prototype scope?**
   `singleton` — один экземпляр на контейнер (по умолчанию). `prototype` — новый экземпляр
   при каждом запросе. Singleton может вызвать проблемы при хранении состояния.

7. **Как работает @MockBean в тестах?**
   Заменяет реальный bean в ApplicationContext на Mockito-мок. Контекст пересоздаётся, если
   набор `@MockBean` отличается от предыдущего теста.

8. **Почему constructor injection предпочтительнее field injection?**
   Позволяет создавать объект без Spring-контекста, делает зависимости явными, поддерживает
   `final` поля, упрощает тестирование.

### 🔴 Продвинутый уровень

9. **Как Spring кэширует тестовый контекст?**
   Spring кэширует ApplicationContext по ключу, включающему конфигурацию, профили, properties
   и `@MockBean`. Если ключ совпадает — контекст переиспользуется. Разные `@MockBean`
   создают разные контексты.

10. **Что происходит при циклической зависимости и как её решить?**
    Spring не может создать bean с циклической зависимостью через constructor injection.
    Решения: рефакторинг, вынесение общей логики в отдельный сервис, `@Lazy` для отложенной
    инициализации.

11. **Как работают AOP-проксирования и почему self-invocation не работает?**
    Spring создаёт прокси-объект вокруг bean. Аннотации `@Transactional`, `@Cacheable` работают
    через прокси. Вызов метода внутри того же класса (`this.method()`) обходит прокси — аннотация
    не сработает.

---

## Практические задания

### Задание 1: Определение типа DI

Посмотрите на код и определите, какой тип DI используется. Объясните, как написать юнит-тест.

```java
@Service
public class NotificationService {

    @Autowired
    private EmailSender emailSender;

    @Autowired
    private SmsSender smsSender;

    public void notifyUser(User user, String message) {
        if (user.prefersEmail()) {
            emailSender.send(user.getEmail(), message);
        } else {
            smsSender.send(user.getPhone(), message);
        }
    }
}
```

### Задание 2: Найдите баг

```java
@Service
public class ShoppingCart {

    private final List<Item> items = new ArrayList<>(); // Состояние в singleton!

    public void addItem(Item item) {
        items.add(item);
    }

    public List<Item> getItems() {
        return items;
    }

    public double getTotal() {
        return items.stream()
                .mapToDouble(Item::getPrice)
                .sum();
    }
}
```

Вопрос: что произойдёт, если два пользователя одновременно добавят товары в корзину?

### Задание 3: Напишите тест

Напишите юнит-тест для `OrderService`, подменив зависимости моками.

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final PaymentService paymentService;

    public OrderService(OrderRepository orderRepository, PaymentService paymentService) {
        this.orderRepository = orderRepository;
        this.paymentService = paymentService;
    }

    public Order placeOrder(String productId, int quantity) {
        Order order = new Order(productId, quantity);
        boolean paid = paymentService.charge(order.getTotalPrice());
        if (!paid) {
            throw new PaymentException("Оплата не прошла");
        }
        return orderRepository.save(order);
    }
}
```

### Задание 4: Исправьте тестовый код

```java
// Почему этот тест медленный и как его оптимизировать?
@SpringBootTest
class StringUtilsTest {

    @Test
    void shouldCapitalizeFirstLetter() {
        String result = StringUtils.capitalize("hello");
        assertEquals("Hello", result);
    }

    @Test
    void shouldReturnEmptyForNull() {
        String result = StringUtils.capitalize(null);
        assertEquals("", result);
    }
}
```

---

## Дополнительные ресурсы

- [Spring Framework Documentation](https://docs.spring.io/spring-framework/reference/) — официальная документация
- [Spring IoC Container](https://docs.spring.io/spring-framework/reference/core/beans.html) — подробно о контейнере
- [Baeldung: Spring Tutorial](https://www.baeldung.com/spring-tutorial) — практические руководства
- [Spring Testing Documentation](https://docs.spring.io/spring-framework/reference/testing.html) — тестирование в Spring
- [Martin Fowler: Inversion of Control](https://martinfowler.com/articles/injection.html) — оригинальная статья об IoC и DI
