# Mockito

## Обзор

Mockito — самый популярный фреймворк для создания mock-объектов (заглушек) в Java-тестах. Он позволяет изолировать тестируемый класс от его зависимостей, создавая контролируемые подстановки вместо реальных объектов. Для QA-инженера Mockito критически важен при написании unit-тестов сервисного слоя, тестировании кода, работающего с внешними API, базами данных и другими зависимостями.

**Зависимости (Maven):**

```xml
<!-- Mockito Core -->
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <version>5.11.0</version>
    <scope>test</scope>
</dependency>

<!-- Интеграция с JUnit 5 -->
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-junit-jupiter</artifactId>
    <version>5.11.0</version>
    <scope>test</scope>
</dependency>
```

---

## Зачем нужно мокирование

### Проблема без моков

```java
// Сервис зависит от репозитория (база данных) и email-клиента (внешний сервис)
public class UserService {

    private final UserRepository userRepository;
    private final EmailClient emailClient;

    public UserService(UserRepository userRepository, EmailClient emailClient) {
        this.userRepository = userRepository;
        this.emailClient = emailClient;
    }

    public User register(String name, String email) {
        if (userRepository.existsByEmail(email)) {
            throw new DuplicateEmailException("Email уже занят: " + email);
        }
        User user = new User(name, email);
        User saved = userRepository.save(user);
        emailClient.sendWelcomeEmail(email);
        return saved;
    }
}
```

Без моков для тестирования `register()` потребуется:
- Реальная база данных
- Реальный email-сервер
- Очистка данных после теста

С моками мы **изолируем** `UserService` и тестируем **только его логику**.

---

## @Mock и @InjectMocks

### Настройка с MockitoExtension

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.mockito.Mockito.*;
import static org.assertj.core.api.Assertions.*;

@ExtendWith(MockitoExtension.class)  // Инициализирует моки автоматически
class UserServiceTest {

    @Mock
    private UserRepository userRepository;  // Мок репозитория

    @Mock
    private EmailClient emailClient;  // Мок email-клиента

    @InjectMocks
    private UserService userService;  // Тестируемый объект с внедрёнными моками

    @Test
    void shouldRegisterNewUser() {
        // Arrange — настройка поведения моков
        when(userRepository.existsByEmail("ivan@mail.com")).thenReturn(false);
        when(userRepository.save(any(User.class))).thenAnswer(invocation -> {
            User user = invocation.getArgument(0);
            user.setId(1L);  // Имитируем присвоение ID базой данных
            return user;
        });

        // Act — вызов тестируемого метода
        User result = userService.register("Иван", "ivan@mail.com");

        // Assert — проверка результата
        assertThat(result.getId()).isEqualTo(1L);
        assertThat(result.getName()).isEqualTo("Иван");
        assertThat(result.getEmail()).isEqualTo("ivan@mail.com");
    }
}
```

### Ручная инициализация (без Extension)

```java
class UserServiceManualTest {

    private UserRepository userRepository;
    private EmailClient emailClient;
    private UserService userService;

    @BeforeEach
    void setUp() {
        userRepository = mock(UserRepository.class);
        emailClient = mock(EmailClient.class);
        userService = new UserService(userRepository, emailClient);
    }
}
```

---

## Mock vs Spy

| Аспект | Mock (`@Mock`) | Spy (`@Spy`) |
|---|---|---|
| Поведение по умолчанию | Методы возвращают `null`/`0`/`false` | Вызываются **реальные** методы |
| Кастомизация | `when(...).thenReturn(...)` | `doReturn(...).when(spy).method()` |
| Создание | `mock(Class.class)` | `spy(realObject)` |
| Применение | Полная замена зависимости | Частичная подмена поведения |

### Пример Spy

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Spy
    private PriceCalculator priceCalculator = new PriceCalculator();
    // Spy — реальный объект с возможностью переопределения методов

    @Mock
    private OrderRepository orderRepository;

    @InjectMocks
    private OrderService orderService;

    @Test
    void shouldApplySpecialDiscount() {
        // Переопределяем только метод получения скидки,
        // остальные методы PriceCalculator работают реально
        doReturn(0.25).when(priceCalculator).getDiscountRate("VIP");

        Order order = orderService.createOrder("VIP", List.of(
            new Item("Laptop", 1000.0)
        ));

        // Реальный метод calculateTotal был вызван,
        // но getDiscountRate вернул подменённое значение 0.25
        assertThat(order.getTotal()).isCloseTo(750.0, within(0.01));
    }
}
```

### Важное различие при настройке Spy

```java
// НЕПРАВИЛЬНО для spy — вызовет реальный метод при настройке!
when(spy.getDiscountRate("VIP")).thenReturn(0.25);

// ПРАВИЛЬНО для spy — не вызывает реальный метод
doReturn(0.25).when(spy).getDiscountRate("VIP");
```

---

## when/thenReturn, when/thenThrow

### Базовая настройка поведения

```java
@Test
void shouldReturnUserById() {
    User expectedUser = new User(1L, "Иван", "ivan@mail.com");

    // Настройка: при вызове findById(1L) вернуть expectedUser
    when(userRepository.findById(1L)).thenReturn(Optional.of(expectedUser));

    // Настройка: при вызове findById(999L) вернуть пустой Optional
    when(userRepository.findById(999L)).thenReturn(Optional.empty());

    // Act & Assert
    Optional<User> found = userService.findById(1L);
    assertThat(found).isPresent().get().extracting(User::getName).isEqualTo("Иван");

    Optional<User> notFound = userService.findById(999L);
    assertThat(notFound).isEmpty();
}

@Test
void shouldThrowWhenDatabaseIsDown() {
    // Настройка: имитируем сбой базы данных
    when(userRepository.findById(anyLong()))
        .thenThrow(new RuntimeException("Connection refused"));

    assertThatThrownBy(() -> userService.findById(1L))
        .isInstanceOf(ServiceException.class)
        .hasMessageContaining("Ошибка при обращении к базе данных");
}
```

### Последовательные вызовы

```java
@Test
void shouldReturnDifferentResultsOnConsecutiveCalls() {
    // Первый вызов → "token-1", второй → "token-2", третий → исключение
    when(authService.generateToken())
        .thenReturn("token-1")
        .thenReturn("token-2")
        .thenThrow(new TokenLimitExceededException());

    assertThat(authService.generateToken()).isEqualTo("token-1");
    assertThat(authService.generateToken()).isEqualTo("token-2");
    assertThatThrownBy(() -> authService.generateToken())
        .isInstanceOf(TokenLimitExceededException.class);
}
```

### thenAnswer — динамический ответ

```java
@Test
void shouldReturnDynamicResponse() {
    when(userRepository.save(any(User.class))).thenAnswer(invocation -> {
        // Получаем аргумент, переданный в save()
        User user = invocation.getArgument(0);
        user.setId(ThreadLocalRandom.current().nextLong(1, 1000));
        user.setCreatedAt(LocalDateTime.now());
        return user;
    });

    User saved = userService.register("Тест", "test@mail.com");
    assertThat(saved.getId()).isPositive();
    assertThat(saved.getCreatedAt()).isNotNull();
}
```

---

## doReturn / doThrow

Альтернативный синтаксис, **обязательный** для spy-объектов и void-методов.

```java
// Для void-методов — нельзя использовать when().thenReturn()
doNothing().when(emailClient).sendWelcomeEmail(anyString());

doThrow(new MailException("SMTP error"))
    .when(emailClient).sendWelcomeEmail("bad@mail.com");

// Для spy — безопасная настройка без вызова реального метода
doReturn(Optional.of(user)).when(spyRepository).findById(1L);
```

---

## verify — проверка вызовов

Verify проверяет, что методы мока были вызваны с ожидаемыми аргументами и нужное количество раз.

```java
@Test
void shouldSendWelcomeEmailAfterRegistration() {
    when(userRepository.existsByEmail(anyString())).thenReturn(false);
    when(userRepository.save(any(User.class))).thenReturn(new User(1L, "Иван", "ivan@mail.com"));

    userService.register("Иван", "ivan@mail.com");

    // Проверяем, что email был отправлен ровно 1 раз с правильным адресом
    verify(emailClient).sendWelcomeEmail("ivan@mail.com");
    // Эквивалентно:
    verify(emailClient, times(1)).sendWelcomeEmail("ivan@mail.com");
}

@Test
void shouldNotSendEmailWhenRegistrationFails() {
    when(userRepository.existsByEmail("dup@mail.com")).thenReturn(true);

    assertThatThrownBy(() -> userService.register("Дубль", "dup@mail.com"))
        .isInstanceOf(DuplicateEmailException.class);

    // Проверяем, что email НЕ был отправлен
    verify(emailClient, never()).sendWelcomeEmail(anyString());

    // Проверяем, что save() тоже не вызывался
    verify(userRepository, never()).save(any());
}

@Test
void verifyCallCounts() {
    // ... тестовая логика ...

    verify(userRepository, times(2)).findById(anyLong());   // Ровно 2 раза
    verify(emailClient, atLeastOnce()).sendEmail(any());     // Минимум 1 раз
    verify(emailClient, atMost(3)).sendEmail(any());         // Максимум 3 раза
    verify(emailClient, atLeast(1)).sendEmail(any());        // Минимум 1 раз

    // Проверяем, что больше никаких взаимодействий не было
    verifyNoMoreInteractions(emailClient);
}
```

### Проверка порядка вызовов

```java
@Test
void shouldCallMethodsInCorrectOrder() {
    userService.register("Иван", "ivan@mail.com");

    // InOrder гарантирует порядок вызовов
    InOrder inOrder = inOrder(userRepository, emailClient);
    inOrder.verify(userRepository).existsByEmail("ivan@mail.com");
    inOrder.verify(userRepository).save(any(User.class));
    inOrder.verify(emailClient).sendWelcomeEmail("ivan@mail.com");
}
```

---

## ArgumentCaptor

Перехватывает аргументы, переданные в мок, для последующей проверки.

```java
@ExtendWith(MockitoExtension.class)
class NotificationServiceTest {

    @Mock
    private EmailClient emailClient;

    @Captor  // Альтернатива: ArgumentCaptor.forClass(Email.class)
    private ArgumentCaptor<Email> emailCaptor;

    @InjectMocks
    private NotificationService notificationService;

    @Test
    void shouldSendCorrectWelcomeEmail() {
        notificationService.sendWelcome("ivan@mail.com", "Иван");

        // Перехватываем аргумент, с которым был вызван sendEmail()
        verify(emailClient).sendEmail(emailCaptor.capture());

        // Проверяем перехваченный объект
        Email capturedEmail = emailCaptor.getValue();
        assertThat(capturedEmail.getTo()).isEqualTo("ivan@mail.com");
        assertThat(capturedEmail.getSubject()).contains("Добро пожаловать");
        assertThat(capturedEmail.getBody()).contains("Иван");
    }

    @Test
    void shouldSendMultipleNotifications() {
        notificationService.notifyAll(List.of("a@mail.com", "b@mail.com"));

        // Перехватываем все вызовы
        verify(emailClient, times(2)).sendEmail(emailCaptor.capture());

        // getAllValues() возвращает список всех перехваченных значений
        List<Email> capturedEmails = emailCaptor.getAllValues();
        assertThat(capturedEmails).hasSize(2);
        assertThat(capturedEmails)
            .extracting(Email::getTo)
            .containsExactly("a@mail.com", "b@mail.com");
    }
}
```

---

## ArgumentMatchers

Гибкие матчеры для настройки поведения и верификации вызовов.

```java
@Test
void argumentMatchersExample() {
    // any() — любой аргумент
    when(userRepository.findById(anyLong())).thenReturn(Optional.of(testUser));

    // eq() — точное совпадение (нужен при смешивании с any())
    when(userRepository.findByNameAndAge(eq("Иван"), anyInt()))
        .thenReturn(List.of(testUser));

    // any(Class) — любой объект указанного типа
    when(userRepository.save(any(User.class))).thenReturn(testUser);

    // Строковые матчеры
    when(userRepository.findByEmail(contains("@mail.com")))
        .thenReturn(Optional.of(testUser));

    // argThat — кастомное условие
    when(userRepository.save(argThat(user ->
            user.getName() != null && user.getAge() > 0)))
        .thenReturn(testUser);
}
```

### Важное правило: все или ничего

```java
// НЕПРАВИЛЬНО — нельзя смешивать литералы и матчеры
when(service.process("literal", anyInt())).thenReturn(result);

// ПРАВИЛЬНО — если один аргумент матчер, все должны быть матчерами
when(service.process(eq("literal"), anyInt())).thenReturn(result);
```

### argThat — кастомная проверка аргумента

```java
@Test
void shouldSaveUserWithCorrectData() {
    userService.register("Иван", "ivan@mail.com");

    verify(userRepository).save(argThat(user ->
        user.getName().equals("Иван") &&
        user.getEmail().equals("ivan@mail.com") &&
        user.getCreatedAt() != null
    ));
}
```

---

## BDD-стиль (given/willReturn)

Mockito поддерживает стиль BDD (Behavior-Driven Development), который лучше читается в контексте Given-When-Then.

```java
import static org.mockito.BDDMockito.*;

@Test
void shouldRegisterUserBddStyle() {
    // Given — начальные условия
    given(userRepository.existsByEmail("ivan@mail.com")).willReturn(false);
    given(userRepository.save(any(User.class))).willReturn(
        new User(1L, "Иван", "ivan@mail.com"));

    // When — действие
    User result = userService.register("Иван", "ivan@mail.com");

    // Then — проверки
    then(userRepository).should().save(any(User.class));
    then(emailClient).should().sendWelcomeEmail("ivan@mail.com");
    then(userRepository).should(never()).deleteById(anyLong());

    assertThat(result.getName()).isEqualTo("Иван");
}
```

### Сравнение стандартного и BDD стилей

```java
// Стандартный стиль
when(mock.method()).thenReturn(value);
verify(mock).method();

// BDD стиль
given(mock.method()).willReturn(value);
then(mock).should().method();
```

---

## MockitoExtension для JUnit 5

```java
@ExtendWith(MockitoExtension.class)
class ServiceTest {

    @Mock
    private Repository repository;

    @Mock
    private ExternalApi externalApi;

    @Spy
    private Mapper mapper = new MapperImpl();

    @Captor
    private ArgumentCaptor<Request> requestCaptor;

    @InjectMocks
    private Service service;
    // Mockito автоматически подставит repository, externalApi и mapper

    @Test
    void shouldProcessRequest() {
        // Тест с полностью настроенными моками
    }
}
```

### Strict Stubbing

По умолчанию `MockitoExtension` использует **strict stubbing** — если вы настроили мок (`when().thenReturn()`), но не использовали его в тесте, будет ошибка. Это помогает поддерживать чистоту тестов.

```java
// Если этот when() не используется в тесте, MockitoExtension выдаст
// UnnecessaryStubbingException
when(userRepository.findById(1L)).thenReturn(Optional.of(user));

// Для отключения (не рекомендуется):
@ExtendWith(MockitoExtension.class)
@MockitoSettings(strictness = Strictness.LENIENT)
class LenientTest { ... }
```

---

## Когда QA использует моки

### Unit-тестирование сервисного слоя

```java
// Тестируем бизнес-логику OrderService, мокируя все зависимости
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock private OrderRepository orderRepository;
    @Mock private PaymentGateway paymentGateway;
    @Mock private NotificationService notificationService;
    @Mock private InventoryService inventoryService;

    @InjectMocks private OrderService orderService;

    @Test
    void shouldRejectOrderWhenItemOutOfStock() {
        // Имитируем отсутствие товара на складе
        given(inventoryService.isAvailable("ITEM-001", 5)).willReturn(false);

        assertThatThrownBy(() ->
            orderService.placeOrder("ITEM-001", 5, "user-1"))
            .isInstanceOf(OutOfStockException.class);

        // Убеждаемся, что оплата и уведомление НЕ были вызваны
        then(paymentGateway).shouldHaveNoInteractions();
        then(notificationService).shouldHaveNoInteractions();
    }
}
```

### Изоляция от внешних зависимостей

```java
@Test
void shouldHandlePaymentGatewayTimeout() {
    given(inventoryService.isAvailable(anyString(), anyInt())).willReturn(true);

    // Имитируем таймаут платёжного шлюза
    given(paymentGateway.charge(any()))
        .willThrow(new PaymentTimeoutException("Gateway timeout"));

    assertThatThrownBy(() ->
        orderService.placeOrder("ITEM-001", 1, "user-1"))
        .isInstanceOf(OrderProcessingException.class)
        .hasMessageContaining("Ошибка оплаты");

    // Проверяем, что заказ не был сохранён
    then(orderRepository).should(never()).save(any());
}
```

### Тестирование взаимодействий

```java
@Test
void shouldSendNotificationAfterSuccessfulOrder() {
    given(inventoryService.isAvailable("ITEM-001", 1)).willReturn(true);
    given(paymentGateway.charge(any())).willReturn(new PaymentResult(true, "TX-123"));
    given(orderRepository.save(any())).willAnswer(inv -> {
        Order order = inv.getArgument(0);
        order.setId(42L);
        return order;
    });

    orderService.placeOrder("ITEM-001", 1, "user-1");

    // Перехватываем уведомление и проверяем его содержимое
    ArgumentCaptor<Notification> captor = ArgumentCaptor.forClass(Notification.class);
    then(notificationService).should().send(captor.capture());

    Notification notification = captor.getValue();
    assertThat(notification.getUserId()).isEqualTo("user-1");
    assertThat(notification.getMessage()).contains("Заказ #42");
    assertThat(notification.getType()).isEqualTo(NotificationType.ORDER_CONFIRMED);
}
```

---

## Связь с тестированием

Mockito — ключевой инструмент QA-инженера для:

- **Unit-тесты сервисов**: изоляция бизнес-логики от БД, API, файловой системы
- **Проверка граничных случаев**: имитация таймаутов, ошибок сети, пустых ответов
- **Тестирование взаимодействий**: verify, что нужные методы вызваны с нужными аргументами
- **Тестирование обработки ошибок**: `thenThrow()` для проверки exception handling
- **Ускорение тестов**: моки работают мгновенно, не нужна БД/сеть
- **CI/CD**: unit-тесты с моками быстро запускаются в пайплайне

---

## Типичные ошибки

1. **Мокирование тестируемого класса** — мокируйте зависимости, а не сам SUT (System Under Test)
2. **`when()` на spy** — используйте `doReturn().when(spy)` вместо `when(spy.method())`
3. **Смешивание литералов и матчеров** — если один аргумент матчер, все должны быть матчерами (`eq()`)
4. **Забыть `@ExtendWith(MockitoExtension.class)`** — моки не инициализируются, `NullPointerException`
5. **Избыточное мокирование** — если mock не нужен в тесте, не настраивайте его (strict stubbing предупредит)
6. **verify без assert** — verify проверяет вызовы, но не результат; нужно и то, и другое
7. **Мокирование final-классов/методов** — требует `mockito-inline` (с Mockito 5 — по умолчанию)
8. **Тестирование реализации вместо поведения** — не проверяйте каждый внутренний вызов, фокусируйтесь на результате

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Что такое mock-объект и зачем он нужен?
2. Чем `@Mock` отличается от `@InjectMocks`?
3. Как настроить возвращаемое значение мока?
4. Как проверить, что метод мока был вызван?
5. Что такое `verify` и какие параметры он принимает?

### 🟡 Средний уровень
6. Чем mock отличается от spy? Когда применять каждый?
7. Почему для spy нужен `doReturn` вместо `when`?
8. Что такое `ArgumentCaptor` и когда он полезен?
9. Как работает strict stubbing в MockitoExtension?
10. Объясните правило «все или ничего» для ArgumentMatchers.

### 🔴 Продвинутый уровень
11. Чем BDD-стиль Mockito отличается от стандартного? Когда его использовать?
12. Как тестировать void-методы с Mockito?
13. Как мокировать static-методы? (MockedStatic)
14. Когда мокирование — антипаттерн? Приведите примеры.
15. Как обеспечить потокобезопасность при тестировании с моками?

---

## Практические задания

### Задание 1: Базовые моки
Напишите тесты для `AuthService.login(username, password)`, который зависит от `UserRepository` и `PasswordEncoder`. Покройте: успешный вход, неверный пароль, пользователь не найден, заблокированный аккаунт.

### Задание 2: ArgumentCaptor
Протестируйте `OrderService.placeOrder()`, перехватив объект `Order`, переданный в `orderRepository.save()`. Проверьте все поля перехваченного объекта.

### Задание 3: Spy
Создайте spy для `PriceCalculator`, переопределив только метод получения налоговой ставки. Убедитесь, что остальные методы работают реально.

### Задание 4: Exception-сценарии
Напишите тесты для `PaymentService`, имитируя различные ошибки платёжного шлюза: таймаут, отклонение карты, недостаточно средств. Проверьте, что сервис корректно обрабатывает каждую ошибку.

### Задание 5: BDD-стиль
Перепишите тесты из Задания 1 в BDD-стиле (given/when/then). Сравните читаемость с обычным стилем.

---

## Дополнительные ресурсы

- [Mockito Official Documentation](https://site.mockito.org/)
- [Mockito GitHub](https://github.com/mockito/mockito)
- [Mockito Javadoc](https://javadoc.io/doc/org.mockito/mockito-core/latest/org/mockito/Mockito.html)
- [Baeldung — Mockito](https://www.baeldung.com/mockito-series)
- [Baeldung — BDD Mockito](https://www.baeldung.com/bdd-mockito)
- [Mockito Wiki — FAQ](https://github.com/mockito/mockito/wiki/FAQ)
