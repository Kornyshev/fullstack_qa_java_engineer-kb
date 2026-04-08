# Практическое использование AI в QA

## Обзор

Этот раздел -- практическое руководство по использованию AI (в первую очередь LLM) в
повседневной работе QA-инженера. Здесь собраны конкретные примеры промптов, ожидаемые
результаты и рекомендации по каждому сценарию использования.

В отличие от предыдущего раздела, фокус здесь на практике: "как именно написать промпт",
"что ожидать в ответе", "как верифицировать результат". Каждый раздел содержит реальный
пример промпта и разбор полученного результата.

Важно помнить: AI -- это инструмент, требующий контроля. Каждый сгенерированный артефакт
нуждается в ревью, проверке и адаптации к конкретному проекту.

---

## Массовая генерация тест-кейсов

### Сценарий

У вас есть User Story или требования, и нужно быстро создать набор тест-кейсов
с покрытием позитивных, негативных и граничных сценариев.

### Пример промпта

```
Роль: Ты -- Senior QA Engineer с 10-летним опытом.

Задача: Сгенерируй полный набор тест-кейсов для следующей User Story.

User Story:
"Как зарегистрированный пользователь, я хочу изменить свой пароль,
чтобы повысить безопасность аккаунта."

Acceptance Criteria:
1. Пользователь должен ввести текущий пароль
2. Новый пароль: мин. 8 символов, макс. 64 символа, 1 заглавная, 1 цифра, 1 спецсимвол
3. Подтверждение нового пароля должно совпадать
4. Новый пароль не может совпадать с текущим
5. После смены пароля все активные сессии кроме текущей завершаются
6. На email отправляется уведомление о смене пароля
7. После 5 неудачных попыток ввода текущего пароля -- блокировка на 15 минут

Формат: таблица Markdown с колонками:
| ID | Категория | Название | Предусловия | Шаги | Тестовые данные | Ожидаемый результат | Приоритет |

Категории: Positive, Negative, Boundary, Security, UX.
Сгенерируй минимум 25 тест-кейсов.
```

### Ожидаемый результат (фрагмент)

| ID | Категория | Название | Предусловия | Шаги | Тестовые данные | Ожидаемый результат | Приоритет |
|----|-----------|----------|-------------|------|-----------------|---------------------|-----------|
| TC-01 | Positive | Успешная смена пароля | Пользователь авторизован | 1. Перейти в настройки 2. Ввести текущий пароль 3. Ввести новый пароль 4. Подтвердить 5. Нажать "Сохранить" | Текущий: OldPass1!, Новый: NewPass2@ | Пароль изменён, уведомление на email | High |
| TC-02 | Negative | Неверный текущий пароль | Пользователь авторизован | 1. Ввести неверный текущий пароль 2. Ввести новый пароль 3. Нажать "Сохранить" | Текущий: WrongPass1! | Ошибка "Неверный текущий пароль" | High |
| TC-03 | Boundary | Минимальная длина пароля (8 символов) | Пользователь авторизован | 1. Ввести новый пароль длиной 8 символов | Новый: Abcdef1! | Пароль принят | Medium |

### Верификация результата

После генерации обязательно проверьте:
- [ ] Покрыты ли все Acceptance Criteria
- [ ] Есть ли тесты на граничные значения (7, 8, 64, 65 символов)
- [ ] Есть ли security-тесты (блокировка, брутфорс, инъекции в поле пароля)
- [ ] Корректны ли ожидаемые результаты
- [ ] Нет ли дублирующихся тест-кейсов
- [ ] Учтена ли специфика вашего проекта (которую AI не знает)

---

## Генерация тестовых данных

### Сценарий

Нужны реалистичные тестовые данные для параметризованных тестов, CSV-файлов
или наполнения тестовой базы данных.

### Пример промпта

```
Сгенерируй 15 тестовых записей заказов интернет-магазина в формате JSON Array.

Структура каждого заказа:
{
  "orderId": "string (UUID)",
  "customerName": "string",
  "email": "string",
  "items": [{"productName": "string", "quantity": int, "price": float}],
  "totalAmount": float,
  "status": "string",
  "createdAt": "ISO 8601 datetime",
  "shippingAddress": {"city": "string", "street": "string", "zip": "string"}
}

Требования:
- Реалистичные названия товаров (электроника, одежда, книги)
- Статусы: PENDING (3), CONFIRMED (5), SHIPPED (4), DELIVERED (2), CANCELLED (1)
- totalAmount = сумма (quantity * price) для всех items
- Даты: последние 30 дней
- 2 заказа с невалидным email (для негативных тестов)
- 1 заказ с пустым массивом items (edge case)
- 1 заказ с отрицательным quantity (edge case для валидации)
```

### Верификация данных

После генерации проверьте:
- [ ] Правильно ли рассчитан totalAmount (AI часто ошибается в арифметике)
- [ ] Корректен ли формат дат
- [ ] Уникальны ли orderId
- [ ] Соответствует ли распределение статусов запросу
- [ ] Включены ли заказанные edge cases

> **Важно:** AI часто допускает ошибки в математических расчётах.
> Всегда пересчитывайте totalAmount вручную или скриптом.

---

## Ревью тестового кода с AI

### Сценарий

Вы написали автотест и хотите получить обратную связь по качеству, надёжности
и соответствию best practices.

### Пример промпта

```
Проведи детальный code review следующего Selenium-теста на Java.
Оцени по критериям:
1. Надёжность (flaky-факторы, waits)
2. Читаемость и поддерживаемость
3. Соответствие паттерну Page Object
4. Качество assertions
5. Обработка ошибок
6. Потенциальные проблемы

Для каждого замечания укажи:
- Строку кода
- Категорию (Critical / Major / Minor / Suggestion)
- Проблему
- Рекомендацию с примером исправленного кода

Код:

@Test
public void testLogin() {
    driver.get("http://localhost:8080/login");
    driver.findElement(By.id("username")).sendKeys("admin");
    driver.findElement(By.id("password")).sendKeys("admin123");
    driver.findElement(By.id("loginBtn")).click();
    Thread.sleep(3000);
    String title = driver.getTitle();
    Assert.assertEquals(title, "Dashboard");
    driver.findElement(By.xpath("//div[@class='user-menu']//span")).click();
    Thread.sleep(1000);
    driver.findElement(By.linkText("Logout")).click();
    Thread.sleep(2000);
    Assert.assertTrue(driver.getCurrentUrl().contains("login"));
}
```

### Ожидаемые замечания от AI

| # | Категория | Проблема | Рекомендация |
|---|-----------|----------|-------------|
| 1 | Critical | `Thread.sleep(3000)` -- хардкод ожидания, flaky | Использовать `WebDriverWait` с `ExpectedConditions` |
| 2 | Major | Отсутствие Page Object -- локаторы и логика в тесте | Вынести в `LoginPage` и `DashboardPage` |
| 3 | Major | Хардкод credentials в тесте | Вынести в конфигурацию или properties-файл |
| 4 | Minor | Нет `@DisplayName` -- неинформативное имя теста | Добавить описание: `shouldLoginAndLogoutSuccessfully` |
| 5 | Minor | XPath `//div[@class='user-menu']//span` -- хрупкий локатор | Добавить `data-testid` атрибут |

---

## Генерация скелетов API-тестов

### Сценарий

У вас есть Swagger/OpenAPI-спецификация, и нужно быстро сгенерировать структуру
автотестов для всех эндпоинтов.

### Пример промпта

```
На основе следующей OpenAPI-спецификации сгенерируй автотесты на Java
с использованием RestAssured + JUnit 5 + AssertJ.

Спецификация (фрагмент):
POST /api/v1/users -- создание пользователя
  Request: {"name": "string", "email": "string", "role": "USER|ADMIN"}
  Response 201: {"id": "long", "name": "string", "email": "string", "role": "string"}
  Response 400: {"error": "string", "details": ["string"]}
  Response 409: {"error": "Пользователь с таким email уже существует"}

GET /api/v1/users/{id} -- получение пользователя
  Response 200: {"id": "long", "name": "string", "email": "string", "role": "string"}
  Response 404: {"error": "Пользователь не найден"}

Требования:
- Отдельный класс для каждого ресурса (UserApiTest)
- Вспомогательные методы для создания/удаления тестовых данных
- @BeforeEach / @AfterEach для setup/cleanup
- Позитивные тесты для каждого эндпоинта
- Негативные тесты: невалидные данные (400), дубликат (409), не найден (404)
- @Tag для категоризации (smoke, regression)
- Комментарии на русском языке
```

### Ожидаемый результат (фрагмент)

```java
@Tag("regression")
class UserApiTest extends BaseApiTest {

    private static final String USERS_PATH = "/api/v1/users";
    private Long createdUserId;

    @BeforeEach
    void setUp() {
        // Подготовка: создаём тестового пользователя, если нужен
    }

    @AfterEach
    void tearDown() {
        // Очистка: удаляем созданного пользователя
        if (createdUserId != null) {
            given()
                .pathParam("id", createdUserId)
            .when()
                .delete(USERS_PATH + "/{id}")
            .then()
                .statusCode(anyOf(is(200), is(204), is(404)));
        }
    }

    @Test
    @Tag("smoke")
    @DisplayName("Создание пользователя с валидными данными")
    void shouldCreateUserWithValidData() {
        // Подготовка тестовых данных
        var requestBody = Map.of(
            "name", "Иван Петров",
            "email", "ivan.petrov@example.com",
            "role", "USER"
        );

        // Выполнение запроса и проверка
        createdUserId = given()
            .body(requestBody)
        .when()
            .post(USERS_PATH)
        .then()
            .statusCode(201)
            .body("name", equalTo("Иван Петров"))
            .body("email", equalTo("ivan.petrov@example.com"))
            .body("role", equalTo("USER"))
            .body("id", notNullValue())
            .extract().jsonPath().getLong("id");
    }

    @Test
    @DisplayName("Создание пользователя с невалидным email -- ошибка 400")
    void shouldReturn400WhenEmailIsInvalid() {
        var requestBody = Map.of(
            "name", "Тест",
            "email", "invalid-email",
            "role", "USER"
        );

        given()
            .body(requestBody)
        .when()
            .post(USERS_PATH)
        .then()
            .statusCode(400)
            .body("error", notNullValue());
    }

    @Test
    @DisplayName("Получение несуществующего пользователя -- ошибка 404")
    void shouldReturn404WhenUserNotFound() {
        given()
            .pathParam("id", 999999)
        .when()
            .get(USERS_PATH + "/{id}")
        .then()
            .statusCode(404)
            .body("error", containsString("не найден"));
    }
}
```

### Верификация

- [ ] Компилируется ли код (правильные импорты, методы)
- [ ] Соответствует ли структура реальному API
- [ ] Корректны ли HTTP-коды в assertions
- [ ] Есть ли cleanup тестовых данных
- [ ] Не хардкодит ли AI Base URL (должен быть в BaseApiTest)

---

## Конвертация ручных тестов в автоматизированные

### Сценарий

У вас есть набор ручных тест-кейсов в текстовом формате, и нужно преобразовать
их в автотесты.

### Пример промпта

```
Преобразуй следующие ручные тест-кейсы в автоматизированные тесты
на Java + Selenium WebDriver + JUnit 5.

Используй паттерн Page Object. Создай необходимые Page Objects.

Ручные тест-кейсы:

TC-001: Успешный логин
1. Открыть страницу логина (http://app.example.com/login)
2. Ввести email: user@example.com
3. Ввести пароль: Password123!
4. Нажать кнопку "Войти"
5. Проверить: отображается Dashboard с приветствием "Добро пожаловать, User"

TC-002: Логин с неверным паролем
1. Открыть страницу логина
2. Ввести email: user@example.com
3. Ввести пароль: WrongPassword
4. Нажать кнопку "Войти"
5. Проверить: отображается ошибка "Неверный email или пароль"
6. Проверить: пользователь остаётся на странице логина

TC-003: Логин с пустыми полями
1. Открыть страницу логина
2. Оставить поля пустыми
3. Нажать кнопку "Войти"
4. Проверить: отображается валидация "Email обязателен" и "Пароль обязателен"

Требования:
- Page Objects: LoginPage, DashboardPage
- WebDriverWait вместо Thread.sleep
- Локаторы через By.id или By.cssSelector (не XPath)
- Комментарии на русском
- Credentials из конфигурации, не хардкод
```

### Ожидаемый результат (фрагмент Page Object)

```java
public class LoginPage {
    private final WebDriver driver;
    private final WebDriverWait wait;

    // Локаторы
    private final By emailField = By.id("email");
    private final By passwordField = By.id("password");
    private final By loginButton = By.id("login-btn");
    private final By errorMessage = By.cssSelector(".error-message");
    private final By emailValidation = By.cssSelector("#email + .validation-error");
    private final By passwordValidation = By.cssSelector("#password + .validation-error");

    public LoginPage(WebDriver driver) {
        this.driver = driver;
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(10));
    }

    // Открытие страницы логина
    public LoginPage open() {
        driver.get(Config.BASE_URL + "/login");
        wait.until(ExpectedConditions.visibilityOfElementLocated(emailField));
        return this;
    }

    // Ввод email
    public LoginPage enterEmail(String email) {
        driver.findElement(emailField).clear();
        driver.findElement(emailField).sendKeys(email);
        return this;
    }

    // Ввод пароля
    public LoginPage enterPassword(String password) {
        driver.findElement(passwordField).clear();
        driver.findElement(passwordField).sendKeys(password);
        return this;
    }

    // Нажатие кнопки "Войти" с ожиданием перехода на Dashboard
    public DashboardPage clickLoginExpectingSuccess() {
        driver.findElement(loginButton).click();
        return new DashboardPage(driver);
    }

    // Нажатие кнопки "Войти" с ожиданием ошибки
    public LoginPage clickLoginExpectingError() {
        driver.findElement(loginButton).click();
        wait.until(ExpectedConditions.visibilityOfElementLocated(errorMessage));
        return this;
    }

    // Получение текста ошибки
    public String getErrorMessage() {
        return wait.until(ExpectedConditions.visibilityOfElementLocated(errorMessage)).getText();
    }
}
```

---

## AI для анализа логов

### Сценарий

У вас есть лог-файл с ошибками, и нужно быстро определить root cause.

### Пример промпта

```
Проанализируй следующий фрагмент лога Java-приложения (Spring Boot)
и определи root cause проблемы.

Лог:
2024-03-15 14:32:01.234 ERROR [http-nio-8080-exec-15] c.e.s.OrderService -
  Failed to create order for user 12345
  org.springframework.dao.DataIntegrityViolationException: could not execute statement;
  SQL [n/a]; constraint [uk_orders_idempotency_key];
  nested exception is org.hibernate.exception.ConstraintViolationException
    at org.springframework.orm.jpa.vendor.HibernateJpaDialect.convertHibernateAccessException
    at org.springframework.orm.jpa.vendor.HibernateJpaDialect.translateExceptionIfPossible
    ...
2024-03-15 14:32:01.235 WARN [http-nio-8080-exec-15] c.e.c.OrderController -
  Returning 500 to client for order creation
2024-03-15 14:32:00.998 INFO [http-nio-8080-exec-14] c.e.s.OrderService -
  Order created successfully: orderId=67890, userId=12345, idempotencyKey=abc-123
2024-03-15 14:32:01.100 INFO [http-nio-8080-exec-15] c.e.s.OrderService -
  Creating order: userId=12345, idempotencyKey=abc-123

Контекст: REST API интернет-магазина, PostgreSQL, высокая нагрузка.

Ответь:
1. Root cause проблемы
2. Почему это произошло
3. Как воспроизвести
4. Как исправить
5. Какой тест-кейс написать для предотвращения
```

### Ожидаемый ответ от AI

```
1. Root cause: Дублирование заказа из-за повторной отправки запроса
   с одинаковым idempotency key (abc-123).

2. Почему: Два запроса (exec-14 и exec-15) с одним idempotency key
   обработались параллельно. Первый успешно создал заказ (14:32:00.998),
   второй попытался создать дубликат (14:32:01.100) и получил
   ConstraintViolationException на unique constraint uk_orders_idempotency_key.

   Это race condition: сервер не обрабатывает idempotency корректно --
   вместо возврата существующего заказа возвращает 500.

3. Как воспроизвести: Отправить два идентичных POST /orders запроса
   с одинаковым Idempotency-Key одновременно (concurrent requests).

4. Как исправить: При ConstraintViolationException на idempotency key
   вернуть существующий заказ (200 OK) вместо 500 Internal Server Error.

5. Тест-кейс: Отправить два одновременных POST-запроса с одинаковым
   idempotency key. Оба должны вернуть 200/201 с одинаковым orderId.
```

---

## Генерация SQL-запросов для тестирования

### Пример промпта

```
Сгенерируй SQL-запросы для подготовки тестовых данных и верификации
результатов для тестирования функционала "Список заказов пользователя".

Таблицы:
- users (id, name, email, status)
- orders (id, user_id, total_amount, status, created_at)
- order_items (id, order_id, product_id, quantity, price)
- products (id, name, price, category)

Мне нужны:
1. SQL для создания тестовых данных (5 пользователей, 15 заказов, товары)
2. SQL для проверки корректности фильтрации заказов по статусу
3. SQL для проверки пагинации (правильность offset/limit)
4. SQL для проверки суммы заказа (totalAmount = sum of items)
5. SQL для очистки тестовых данных после теста

Добавь комментарии на русском языке.
```

---

## Генерация документации для тестов

### Пример промпта

```
На основе следующего автотеста сгенерируй описание тест-кейса в формате,
пригодном для тестовой документации (TestRail / Allure).

[вставить код теста]

Формат:
- Название (краткое, информативное)
- Описание (что именно проверяет тест)
- Предусловия
- Шаги (на человеческом языке, не код)
- Ожидаемый результат
- Теги / Категория
```

---

## Этические соображения и требования к верификации

### Правила работы с AI в QA

1. **Всегда верифицируйте** -- AI-вывод это черновик, а не финальный результат.
   Каждый сгенерированный артефакт должен пройти ревью человеком.

2. **Не доверяйте слепо** -- AI может генерировать правдоподобно выглядящий, но
   неверный код. Особенно опасны:
   - Неправильные assertions (тест проходит, но проверяет не то)
   - Несуществующие методы API
   - Некорректная бизнес-логика

3. **Конфиденциальность** -- никогда не отправляйте в публичные LLM:
   - Production-данные пользователей
   - API-ключи, пароли, токены
   - Проприетарный код (если нет разрешения)
   - Персональные данные (PII)

4. **Авторство и ответственность** -- если AI сгенерировал тест, а тест пропустил баг,
   ответственность на QA-инженере, который принял этот тест без должной проверки.

5. **Прозрачность** -- если используете AI для генерации артефактов, сообщайте об этом
   команде. Это помогает в ревью и формирует правильные ожидания.

### Чеклист верификации AI-сгенерированного кода

- [ ] Код компилируется без ошибок
- [ ] Все импорты корректны и существуют
- [ ] Используемые методы API реально существуют (проверить документацию)
- [ ] Assertions проверяют правильные вещи (не просто "статус 200")
- [ ] Тестовые данные реалистичны и покрывают edge cases
- [ ] Нет hardcoded credentials
- [ ] Тесты изолированы (cleanup после выполнения)
- [ ] Нет flaky-факторов (Thread.sleep, отсутствие waits)
- [ ] Логика теста соответствует требованиям проекта
- [ ] Код соответствует стилю и конвенциям команды

### Когда НЕ стоит использовать AI

| Ситуация | Почему |
|----------|--------|
| Тестирование безопасности (penetration testing) | AI не заменяет экспертизу, может пропустить критичные уязвимости |
| Работа с конфиденциальными данными | Риск утечки через API LLM |
| Принятие решений о релизе | AI не понимает бизнес-контекст и риски |
| Exploratory testing (полностью) | AI может предложить идеи, но не заменяет человеческую интуицию |
| Оценка критичности бага | Требует понимания бизнес-импакта, пользовательского опыта |

---

## Связь с тестированием

- Практическое использование AI ускоряет работу QA в 2-5 раз на рутинных задачах
- Генерация тест-кейсов, тестовых данных и шаблонов кода -- основные сценарии
- AI особенно эффективен для "стартовой точки": черновик создаёт AI, финализирует человек
- Ревью AI-сгенерированного кода -- такой же навык, как ревью человеческого кода
- AI помогает Junior-инженерам учиться, показывая примеры хорошего кода и best practices
- Для Senior-инженеров AI -- ускоритель: быстро создать скелет, который затем дорабатывается

---

## Типичные ошибки

1. **Копипаст без ревью** -- самая опасная ошибка. AI сгенерировал красивый код →
   QA скопировал в проект → тесты проходят, но проверяют не то → баг в production.

2. **Слишком общие промпты** -- "Напиши тесты для API" даст generic результат.
   Чем конкретнее промпт (эндпоинты, данные, формат), тем лучше результат.

3. **Отсутствие итераций** -- первый ответ AI редко идеален. Уточняйте, просите
   доработать, указывайте на ошибки. Диалог с AI -- это итеративный процесс.

4. **Игнорирование контекста проекта** -- AI не знает вашу архитектуру, conventions,
   зависимости. Всегда адаптируйте сгенерированный код под проект.

5. **Переоценка возможностей** -- AI не заменяет тестирование. Он помогает создавать
   артефакты, но не принимает решения о качестве продукта.

6. **Утечка данных** -- вставка production SQL-дампа в ChatGPT для "анализа" --
   потенциальная утечка персональных данных пользователей.

7. **Отсутствие стандартов** -- в команде нет правил использования AI → каждый использует
   по-своему → несогласованность артефактов, разное качество.

---

## Вопросы на интервью

### 🟢 Базовый уровень (Junior)

1. Как бы вы использовали ChatGPT/Claude для написания тест-кейсов?
2. Что нужно проверять в коде, сгенерированном AI?
3. Какие данные нельзя отправлять в публичные LLM?
4. Приведите пример промпта для генерации тестовых данных.
5. Почему нельзя использовать AI-вывод без проверки?

### 🟡 Средний уровень (Middle)

1. Как вы организуете процесс верификации AI-сгенерированных тестов?
2. Напишите промпт для генерации API-тестов по Swagger-спецификации.
3. Как использовать AI для анализа логов и определения root cause?
4. Какие практики prompt engineering повышают качество результатов?
5. Как конвертировать ручные тест-кейсы в автоматизированные с помощью AI?
6. Как обеспечить соответствие AI-кода code style и conventions команды?

### 🔴 Продвинутый уровень (Senior)

1. Как бы вы внедрили AI-инструменты в QA-процессы команды? Какие правила и guidelines установили бы?
2. Как измерить эффективность использования AI в QA (метрики, KPI)?
3. Как организовать governance: кто ревьюит AI-сгенерированные артефакты?
4. Какие юридические и этические риски связаны с использованием AI в тестировании?
5. Как вы бы обучали команду QA эффективному использованию AI?
6. В каких случаях вы бы отказались от использования AI в пользу ручного подхода?

---

## Практические задания

### Задание 1: Генерация тест-кейсов
Выберите реальную User Story из вашего проекта (или придумайте). Напишите промпт
для генерации тест-кейсов. Итеративно улучшайте промпт (минимум 3 итерации).
Зафиксируйте, что улучшилось в каждой итерации.

### Задание 2: Конвертация тестов
Возьмите 5 ручных тест-кейсов из вашего проекта. С помощью AI конвертируйте
их в автотесты. Проведите ревью сгенерированного кода:
- Сколько тестов скомпилировались без изменений?
- Какие ошибки допустил AI?
- Сколько времени сэкономлено vs написание с нуля?

### Задание 3: Анализ логов
Найдите (или создайте) фрагмент лога с ошибкой. Попросите AI проанализировать его.
Оцените: правильно ли AI определил root cause? Были ли полезны рекомендации?

### Задание 4: Шаблон промптов для команды
Создайте библиотеку из 10 промптов для типичных QA-задач:
1. Генерация тест-кейсов по User Story
2. Генерация тестовых данных (JSON, CSV)
3. Code review автотеста
4. Генерация API-тестов по спецификации
5. Конвертация ручного теста в автоматизированный
6. Анализ stack trace
7. Написание баг-репорта
8. Идеи для exploratory testing
9. Генерация SQL для тестовых данных
10. Описание тест-кейса для документации

Для каждого промпта: шаблон, пример заполнения, чеклист верификации.

### Задание 5: Оценка точности AI
Попросите AI сгенерировать 20 автотестов для REST API. Проверьте каждый:
- Компилируется ли?
- Корректны ли assertions?
- Нет ли несуществующих методов?
- Покрывает ли реальный сценарий?

Рассчитайте: % полностью корректных, % требующих минимальных правок,
% непригодных к использованию.

---

## Дополнительные ресурсы

- [Prompt Engineering Guide](https://www.promptingguide.ai/)
- [OpenAI Best Practices for Prompt Engineering](https://platform.openai.com/docs/guides/prompt-engineering)
- [Anthropic Prompt Engineering Guide](https://docs.anthropic.com/claude/docs/prompt-engineering)
- [GitHub Copilot Documentation](https://docs.github.com/en/copilot)
- [Ministry of Testing -- AI in Testing Community](https://www.ministryoftesting.com/)
- [Test Automation University](https://testautomationu.applitools.com/)
- [AI-Powered Testing -- Thoughtworks Technology Radar](https://www.thoughtworks.com/radar)
- [Книга: "AI-Augmented Testing" -- Tariq King](https://www.packtpub.com/)
