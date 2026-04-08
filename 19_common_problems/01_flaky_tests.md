# Flaky-тесты: нестабильные тесты и борьба с ними

## Обзор

Flaky-тест (нестабильный тест) — это тест, который при одном и том же коде может то проходить, то падать
без каких-либо изменений в тестируемом приложении или тестовом коде. Это одна из самых болезненных
проблем автоматизации тестирования: flaky-тесты подрывают доверие к test suite, замедляют CI/CD pipeline
и отнимают время команды на расследование ложных падений.

По данным Google, до 16% их тестов демонстрируют flaky-поведение. В реальных проектах, особенно
с UI-автоматизацией, этот показатель может быть ещё выше. Умение диагностировать и устранять
нестабильность тестов — критически важный навык для QA-инженера любого уровня.

---

## Что делает тест flaky

Flaky-тест — это тест, который **недетерминирован**: при одинаковых входных данных он даёт
разные результаты. Формально:

```
Если test(code_v1) = PASS, а затем test(code_v1) = FAIL (без изменения code_v1)
→ тест является flaky
```

Ключевое отличие от обычного падения: причина нестабильности лежит **не в продуктовом коде**
и **не в логике теста**, а в побочных факторах — окружении, таймингах, данных, порядке выполнения.

### Flakiness Rate — показатель нестабильности

```
Flakiness Rate = (количество нестабильных прогонов / общее количество прогонов) × 100%
```

Типичные пороговые значения:

| Flakiness Rate | Оценка | Действие |
|----------------|--------|----------|
| 0–1% | Отлично | Мониторинг |
| 1–5% | Допустимо | Планомерное исправление |
| 5–15% | Проблема | Приоритетное исправление |
| > 15% | Критично | Остановить и чинить |

---

## Основные причины flaky-тестов

### 1. Timing Issues (проблемы синхронизации)

Самая распространённая причина нестабильности, особенно в UI-тестах. Тест не дожидается
готовности элемента или завершения асинхронной операции.

```java
// ПЛОХО — жёсткое ожидание, не гарантирует результат
@Test
void testAddToCart() {
    driver.findElement(By.id("add-btn")).click();
    Thread.sleep(3000); // Иногда 3 секунд хватает, иногда нет
    String count = driver.findElement(By.id("cart-count")).getText();
    assertEquals("1", count);
}

// ХОРОШО — явное ожидание конкретного условия
@Test
void testAddToCart() {
    driver.findElement(By.id("add-btn")).click();
    WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));
    // Ждём, пока счётчик корзины изменится
    wait.until(ExpectedConditions.textToBePresentInElementLocated(
        By.id("cart-count"), "1"
    ));
    String count = driver.findElement(By.id("cart-count")).getText();
    assertEquals("1", count);
}
```

### 2. Shared State (разделяемое состояние)

Тесты используют общие данные или ресурсы, и результат одного теста влияет на другой.

```java
// ПЛОХО — тесты зависят от общего пользователя
class OrderTests {

    // Этот тест создаёт заказ для user_1
    @Test
    void testCreateOrder() {
        api.createOrder("user_1", "product_A");
        List<Order> orders = api.getOrders("user_1");
        assertFalse(orders.isEmpty());
    }

    // Этот тест проверяет пустую корзину для того же user_1
    // Если запустить после testCreateOrder — может упасть
    @Test
    void testEmptyCart() {
        Cart cart = api.getCart("user_1");
        assertTrue(cart.isEmpty()); // Может быть непусто!
    }
}

// ХОРОШО — каждый тест создаёт собственные данные
class OrderTests {

    @Test
    void testCreateOrder() {
        String userId = "user_" + UUID.randomUUID();
        api.createUser(userId);
        api.createOrder(userId, "product_A");
        List<Order> orders = api.getOrders(userId);
        assertFalse(orders.isEmpty());
    }

    @Test
    void testEmptyCart() {
        String userId = "user_" + UUID.randomUUID();
        api.createUser(userId);
        Cart cart = api.getCart(userId);
        assertTrue(cart.isEmpty());
    }
}
```

### 3. Environment Instability (нестабильность окружения)

Тест зависит от доступности внешних сервисов, сетевых условий или ресурсов машины.

Типичные проявления:
- Тест проходит локально, но падает в CI из-за нехватки памяти
- Тест зависит от внешнего API, который периодически недоступен
- База данных перегружена, и запросы выполняются дольше обычного
- DNS-резолвинг занимает непредсказуемое время

### 4. Test Data Pollution (загрязнение тестовых данных)

Тесты не убирают за собой данные, и со временем состояние базы данных или системы
«засоряется» артефактами предыдущих прогонов.

```java
// ПЛОХО — тест создаёт данные, но не удаляет их
@Test
void testUserRegistration() {
    api.register("test@email.com", "password123");
    // Тест проходит, но пользователь остаётся в базе
    // При повторном запуске — ошибка "User already exists"
}

// ХОРОШО — очистка данных после теста
@Test
void testUserRegistration() {
    String email = "test_" + System.currentTimeMillis() + "@email.com";
    try {
        api.register(email, "password123");
        User user = api.getUser(email);
        assertNotNull(user);
    } finally {
        // Очистка данных, даже если тест упал
        api.deleteUser(email);
    }
}
```

### 5. Async Operations (асинхронные операции)

Тест не учитывает, что операция выполняется асинхронно (очередь сообщений, фоновый процесс,
email-отправка).

```java
// ПЛОХО — проверка сразу после запуска асинхронной операции
@Test
void testEmailNotification() {
    api.placeOrder(orderId);
    // Email отправляется асинхронно через очередь сообщений
    Email email = mailService.getLastEmail("user@test.com");
    assertNotNull(email); // Может быть null — email ещё не отправлен!
}

// ХОРОШО — polling с таймаутом
@Test
void testEmailNotification() {
    api.placeOrder(orderId);

    // Ожидание с повторными попытками
    Email email = Awaitility.await()
        .atMost(Duration.ofSeconds(30))
        .pollInterval(Duration.ofSeconds(2))
        .until(
            () -> mailService.getLastEmail("user@test.com"),
            Objects::nonNull
        );

    assertEquals("Order Confirmation", email.getSubject());
}
```

### 6. UI Rendering Delays (задержки отрисовки интерфейса)

Элемент присутствует в DOM, но ещё не отрисован, не кликабелен или перекрыт другим элементом
(spinner, overlay, toast notification).

```java
// ПЛОХО — элемент есть в DOM, но перекрыт спиннером
@Test
void testSubmitForm() {
    driver.findElement(By.id("submit")).click(); // ElementClickInterceptedException!
}

// ХОРОШО — ждём исчезновения спиннера и кликабельности элемента
@Test
void testSubmitForm() {
    WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));
    // Ждём, пока спиннер исчезнет
    wait.until(ExpectedConditions.invisibilityOfElementLocated(By.className("spinner")));
    // Ждём, пока кнопка станет кликабельной
    WebElement submitBtn = wait.until(
        ExpectedConditions.elementToBeClickable(By.id("submit"))
    );
    submitBtn.click();
}
```

### 7. Порядок выполнения тестов

JUnit 5 не гарантирует порядок выполнения тестов. Если тесты зависят от порядка — они нестабильны.

### 8. Проблемы с датой и временем

Тесты, привязанные к текущей дате, могут падать в начале/конце месяца, года, при смене
часовых поясов или переходе на летнее время.

---

## Диагностика flaky-тестов

### Allure History — история прогонов

Allure хранит историю результатов тестов. Если тест мигает между PASS и FAIL — это чёткий
сигнал flaky-поведения.

```
allure-results/
└── history/
    ├── history.json          # История статусов каждого теста
    ├── history-trend.json    # Тренды по прогонам
    └── retry-trend.json      # Тренды по retry
```

В Allure-отчёте перейдите в раздел **Retries** — там видны тесты, которые проходили
только с повторной попытки.

### Retry Analysis — анализ повторных запусков

Специальный прогон для выявления flaky-тестов:

```xml
<!-- Maven Surefire Plugin — запуск каждого теста N раз -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <!-- Повторить упавшие тесты до 3 раз -->
        <rerunFailingTestsCount>3</rerunFailingTestsCount>
    </configuration>
</plugin>
```

### Сравнение Local vs CI

Чек-лист для диагностики расхождений:

| Фактор | Локально | CI |
|--------|----------|-----|
| Ресурсы (CPU, RAM) | Мощная машина | Ограниченный контейнер |
| Разрешение экрана | 1920x1080 | 1024x768 (headless) |
| Сетевые задержки | Минимальные | Зависят от инфраструктуры |
| Параллельность | Один поток | Несколько потоков |
| База данных | Локальная, быстрая | Удалённая, нагруженная |
| Браузер | С GUI | Headless-режим |

### Скрипт массового перезапуска для выявления flaky

```bash
#!/bin/bash
# Запуск тестов N раз для выявления нестабильных
RUNS=10
FAILURES=0

for i in $(seq 1 $RUNS); do
    echo "=== Запуск $i из $RUNS ==="
    mvn test -pl module-name -Dtest=SuspiciousTest
    if [ $? -ne 0 ]; then
        FAILURES=$((FAILURES + 1))
    fi
done

echo "Результат: $FAILURES падений из $RUNS запусков"
echo "Flakiness Rate: $(echo "scale=1; $FAILURES * 100 / $RUNS" | bc)%"
```

---

## Стратегии исправления

### 1. Правильные ожидания (Proper Waits)

```java
public class WaitHelper {

    private final WebDriverWait wait;

    public WaitHelper(WebDriver driver, Duration timeout) {
        this.wait = new WebDriverWait(driver, timeout);
    }

    // Ожидание элемента с кастомным условием
    public WebElement waitForElement(By locator) {
        return wait.until(ExpectedConditions.elementToBeClickable(locator));
    }

    // Ожидание исчезновения элемента (спиннеры, лоадеры)
    public void waitForElementToDisappear(By locator) {
        wait.until(ExpectedConditions.invisibilityOfElementLocated(locator));
    }

    // Ожидание завершения AJAX-запросов (jQuery)
    public void waitForAjax() {
        wait.until(driver ->
            ((JavascriptExecutor) driver)
                .executeScript("return jQuery.active == 0")
        );
    }

    // Ожидание полной загрузки страницы
    public void waitForPageLoad() {
        wait.until(driver ->
            ((JavascriptExecutor) driver)
                .executeScript("return document.readyState")
                .equals("complete")
        );
    }
}
```

### 2. Изоляция тестов (Test Isolation)

```java
@TestInstance(TestInstance.Lifecycle.PER_METHOD)
class IsolatedApiTest {

    private String testUserId;

    @BeforeEach
    void setUp() {
        // Создаём уникальные данные для каждого теста
        testUserId = "user_" + UUID.randomUUID();
        apiClient.createUser(testUserId, "Test User");
    }

    @AfterEach
    void tearDown() {
        // Гарантированная очистка данных
        try {
            apiClient.deleteUser(testUserId);
        } catch (Exception e) {
            log.warn("Не удалось удалить тестового пользователя: {}", testUserId, e);
        }
    }

    @Test
    void testUpdateProfile() {
        apiClient.updateProfile(testUserId, "New Name");
        User user = apiClient.getUser(testUserId);
        assertEquals("New Name", user.getName());
    }
}
```

### 3. Retry Mechanism (механизм повторных попыток)

Retry — это **не решение**, а **временная мера**, позволяющая снизить влияние flaky-тестов
на pipeline, пока идёт работа над устранением причин.

```java
// JUnit 5 — кастомное расширение для retry
public class RetryExtension implements TestExecutionExceptionHandler {

    private static final int MAX_RETRIES = 2;

    @Override
    public void handleTestExecutionException(
            ExtensionContext context, Throwable throwable) throws Throwable {

        int retryCount = getRetryCount(context);

        if (retryCount < MAX_RETRIES) {
            incrementRetryCount(context);
            log.warn("Тест {} упал, повторная попытка {}/{}",
                context.getDisplayName(), retryCount + 1, MAX_RETRIES);
            // Повторный запуск — пересоздаём состояние
            context.getTestMethod().ifPresent(method -> {
                // Логика перезапуска
            });
        } else {
            throw throwable; // Исчерпаны все попытки
        }
    }

    private int getRetryCount(ExtensionContext context) {
        return context.getStore(ExtensionContext.Namespace.GLOBAL)
            .getOrDefault("retryCount_" + context.getUniqueId(), Integer.class, 0);
    }

    private void incrementRetryCount(ExtensionContext context) {
        int current = getRetryCount(context);
        context.getStore(ExtensionContext.Namespace.GLOBAL)
            .put("retryCount_" + context.getUniqueId(), current + 1);
    }
}

// Использование
@ExtendWith(RetryExtension.class)
class FlakyTests {
    @Test
    void testWithRetry() {
        // Если упадёт — будет перезапущен до 2 раз
    }
}
```

### 4. Quarantine Approach (карантин)

Изоляция нестабильных тестов в отдельную группу, чтобы они не блокировали основной pipeline.

```java
// Аннотация для маркировки нестабильных тестов
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Tag("quarantine")
public @interface Quarantine {
    String reason();       // Причина карантина
    String ticket();       // Ссылка на задачу в Jira
    String date();         // Дата помещения в карантин
}

// Использование
class PaymentTests {

    @Test
    @Quarantine(
        reason = "Flaky из-за нестабильного платёжного шлюза на тестовом стенде",
        ticket = "QA-1234",
        date = "2025-12-15"
    )
    void testPaymentProcessing() {
        // ...
    }
}
```

Настройка CI для раздельного запуска:

```xml
<!-- Основной pipeline — без карантинных тестов -->
<configuration>
    <excludedGroups>quarantine</excludedGroups>
</configuration>

<!-- Отдельный pipeline — только карантинные тесты (информационно) -->
<configuration>
    <groups>quarantine</groups>
</configuration>
```

---

## Измерение и мониторинг нестабильности

### Dashboard метрик

```java
/**
 * Утилита для расчёта метрик нестабильности тестов.
 * Анализирует результаты прогонов из Allure history.
 */
public class FlakinessAnalyzer {

    // Расчёт Flakiness Rate для отдельного теста
    public double calculateFlakinessRate(String testId, List<TestRun> runs) {
        if (runs.size() < 2) return 0.0;

        long flips = 0; // Количество переключений PASS ↔ FAIL
        for (int i = 1; i < runs.size(); i++) {
            if (runs.get(i).getStatus() != runs.get(i - 1).getStatus()) {
                flips++;
            }
        }
        // Нормализуем: flips / (runs - 1)
        return (double) flips / (runs.size() - 1);
    }

    // Топ нестабильных тестов
    public List<FlakyTestReport> getTopFlakyTests(
            Map<String, List<TestRun>> allTestRuns, int topN) {
        return allTestRuns.entrySet().stream()
            .map(entry -> new FlakyTestReport(
                entry.getKey(),
                calculateFlakinessRate(entry.getKey(), entry.getValue()),
                entry.getValue().size()
            ))
            .filter(report -> report.getFlakinessRate() > 0.0)
            .sorted(Comparator.comparingDouble(FlakyTestReport::getFlakinessRate).reversed())
            .limit(topN)
            .collect(Collectors.toList());
    }
}
```

### Ключевые метрики для отслеживания

| Метрика | Формула | Целевое значение |
|---------|---------|-----------------|
| Flakiness Rate | Нестабильные прогоны / Всего прогонов | < 1% |
| Mean Time to Fix Flaky | Среднее время от обнаружения до исправления | < 3 дня |
| Quarantine Size | Количество тестов в карантине | < 5% от suite |
| Retry Success Rate | Прошли с retry / Упали с retry | Чем ниже — тем лучше |
| First-Run Pass Rate | Прошли с первого раза / Всего тестов | > 98% |

---

## Реальные примеры из практики

### Пример 1: Тест падает только по понедельникам

**Симптомы:** UI-тест на отображение дашборда стабильно проходит со вторника по пятницу,
но падает в понедельник утром.

**Причина:** Дашборд показывает данные "за последнюю неделю". По понедельникам данные
агрегируются cron-задачей, которая запускается в 06:00. Тесты в CI стартуют в 05:30.

**Решение:** Тест должен сам подготовить данные, а не полагаться на cron-задачу.

### Пример 2: Тест падает при параллельном запуске

**Симптомы:** Поодиночке все тесты проходят. При запуске всего suite — 3-4 теста падают случайно.

**Причина:** Тесты используют общую учётную запись `admin@test.com`. При параллельном запуске
один тест меняет настройки пользователя, другой — проверяет их.

**Решение:** Каждый тест создаёт уникального пользователя через API перед выполнением.

### Пример 3: StaleElementReferenceException в SPAs

**Симптомы:** Тест периодически падает с `StaleElementReferenceException` при работе с таблицей.

**Причина:** React-приложение перерисовывает DOM после каждого обновления состояния.
Ссылка на элемент, полученная до обновления, становится недействительной.

**Решение:** Повторный поиск элемента перед каждым взаимодействием:

```java
// Утилита для работы с динамическим DOM
public WebElement findFreshElement(By locator) {
    return new WebDriverWait(driver, Duration.ofSeconds(5))
        .until(ExpectedConditions.presenceOfElementLocated(locator));
}
```

---

## Связь с тестированием

Flaky-тесты напрямую влияют на эффективность всего процесса тестирования:

- **Доверие к test suite** — если 10% тестов нестабильны, команда начинает игнорировать падения
- **Скорость CI/CD** — retry увеличивает время pipeline, блокируя релизы
- **Продуктивность QA** — расследование ложных падений отнимает 20-30% рабочего времени
- **Качество продукта** — реальные баги теряются среди flaky-падений (эффект «волк-волк»)

---

## Типичные ошибки

1. **Использовать retry как постоянное решение** — retry маскирует проблему, а не устраняет её
2. **Игнорировать flaky-тесты** — «оно иногда падает» — это не нормально
3. **Использовать `Thread.sleep` вместо proper waits** — самый частый источник нестабильности
4. **Не анализировать историю прогонов** — без данных невозможно приоритизировать исправления
5. **Удалять flaky-тесты вместо исправления** — теряется покрытие критической функциональности
6. **Не инвестировать в инфраструктуру** — нестабильные окружения порождают нестабильные тесты

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Что такое flaky-тест? Приведите пример.
2. Чем flaky-тест отличается от теста с настоящим багом?
3. Почему `Thread.sleep` приводит к нестабильности тестов?
4. Что такое Explicit Wait и как он помогает стабилизировать тесты?

### 🟡 Средний уровень
5. Какие основные причины flaky-тестов вы знаете? Назовите минимум 5.
6. Как вы диагностируете flaky-тесты в CI? Какие инструменты используете?
7. Что такое quarantine-подход? Как его организовать?
8. Как обеспечить изоляцию тестовых данных при параллельном запуске?
9. Как retry-механизм помогает и чем вредит?

### 🔴 Продвинутый уровень
10. Опишите процесс систематического снижения Flakiness Rate в большом проекте.
11. Как измерять стоимость flaky-тестов для команды (в часах, в деньгах)?
12. Вы пришли на проект, где 30% тестов — flaky. Какой ваш план действий на первые 2 недели?
13. Как организовать мониторинг и алертинг по метрикам стабильности test suite?
14. Как flaky-тесты влияют на trunk-based development и feature flags?

---

## Практические задания

### Задание 1: Диагностика
Вам дан Allure-отчёт с 200 тестами, из которых 15 упали. Опишите пошаговый алгоритм
определения: какие из 15 падений — это реальные баги, а какие — flaky.

### Задание 2: Стабилизация UI-теста
Дан нестабильный тест:
```java
@Test
void testSearchResults() {
    driver.get("https://example.com");
    driver.findElement(By.id("search")).sendKeys("laptop");
    driver.findElement(By.id("search-btn")).click();
    Thread.sleep(2000);
    List<WebElement> results = driver.findElements(By.className("result-item"));
    assertTrue(results.size() > 0);
    results.get(0).click();
    Thread.sleep(1000);
    String title = driver.findElement(By.id("product-title")).getText();
    assertFalse(title.isEmpty());
}
```
Перепишите его, устранив все потенциальные источники нестабильности.

### Задание 3: Система метрик
Спроектируйте класс `FlakinessTracker`, который:
- Принимает результаты тестовых прогонов
- Вычисляет Flakiness Rate для каждого теста
- Определяет кандидатов на карантин
- Генерирует отчёт с рекомендациями

### Задание 4: Карантин
Реализуйте JUnit 5 extension, которое:
- Автоматически помечает тест как quarantine после N последовательных flaky-прогонов
- Логирует информацию в Allure-отчёт
- Отправляет уведомление в Slack/Teams (через mock)

---

## Дополнительные ресурсы

- [Google Testing Blog: Flaky Tests](https://testing.googleblog.com/) — статьи от Google о борьбе с flaky-тестами
- [Allure Framework Documentation](https://docs.qameta.io/allure/) — работа с историей и retry в Allure
- [JUnit 5 User Guide: Extensions](https://junit.org/junit5/docs/current/user-guide/#extensions) — создание retry-расширений
- [Awaitility](https://github.com/awaitility/awaitility) — библиотека для ожидания асинхронных условий
- Martin Fowler: "Eradicating Non-Determinism in Tests" — классическая статья о нестабильных тестах
- [Selenium WebDriver Waits](https://www.selenium.dev/documentation/webdriver/waits/) — официальная документация по ожиданиям
