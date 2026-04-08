# Анализ падений тестов: систематический подход к диагностике

## Обзор

Падение автотеста — это событие, требующее расследования. Но далеко не каждое падение означает
баг в продукте: причиной может быть ошибка в тестовом коде, проблема окружения или flaky-поведение.
Систематический подход к анализу падений позволяет быстро определить корневую причину и принять
правильное решение — завести баг, починить тест или перезапустить pipeline.

QA-инженер, который умеет быстро и точно классифицировать падения, экономит команде десятки часов
в неделю. Этот навык проверяется на собеседованиях через ситуационные вопросы:
«Вот скриншот упавшего теста — расскажите, как будете разбираться».

---

## Систематический алгоритм анализа

### Пошаговый процесс

```
┌──────────────────────────────────────────────────────────────────┐
│                АЛГОРИТМ АНАЛИЗА ПАДЕНИЯ ТЕСТА                    │
│                                                                  │
│  Шаг 1: Прочитать error message и stack trace                    │
│    ↓                                                             │
│  Шаг 2: Посмотреть screenshot / video (UI-тесты)                │
│    ↓                                                             │
│  Шаг 3: Проверить логи приложения                                │
│    ↓                                                             │
│  Шаг 4: Проанализировать Allure steps                            │
│    ↓                                                             │
│  Шаг 5: Проверить состояние окружения                            │
│    ↓                                                             │
│  Шаг 6: Проверить тестовые данные                                │
│    ↓                                                             │
│  Шаг 7: Воспроизвести локально                                   │
│    ↓                                                             │
│  Шаг 8: Классифицировать и задокументировать                     │
└──────────────────────────────────────────────────────────────────┘
```

### Шаг 1: Error Message и Stack Trace

Первое и самое информативное, что нужно прочитать. Сообщение об ошибке часто прямо указывает
на причину.

**Типичные ошибки и их расшифровка:**

| Error | Вероятная причина | Категория |
|-------|-------------------|-----------|
| `AssertionError: expected <200> but was <500>` | Баг в приложении (сервер вернул ошибку) | Product bug |
| `NoSuchElementException` | Локатор устарел или элемент не загрузился | Test bug / Flaky |
| `StaleElementReferenceException` | DOM перерисован, ссылка на элемент устарела | Test bug |
| `TimeoutException` | Элемент/условие не появилось за указанное время | Flaky / Env issue |
| `ConnectionRefusedException` | Сервис недоступен | Environment issue |
| `ElementClickInterceptedException` | Элемент перекрыт другим (spinner, popup) | Test bug |
| `JsonParseException` | Формат ответа API изменился | Product bug / Test bug |
| `NullPointerException` в тестовом коде | Баг в тесте | Test bug |

```java
/**
 * Утилита для автоматической первичной классификации ошибок.
 * Анализирует exception type и сообщение для определения вероятной причины.
 */
public class FailureClassifier {

    // Правила классификации: паттерн ошибки → категория
    private static final Map<String, FailureCategory> RULES = Map.of(
        "Connection refused", FailureCategory.ENVIRONMENT,
        "Connection reset", FailureCategory.ENVIRONMENT,
        "500 Internal Server Error", FailureCategory.PRODUCT_BUG,
        "502 Bad Gateway", FailureCategory.ENVIRONMENT,
        "503 Service Unavailable", FailureCategory.ENVIRONMENT,
        "NoSuchElementException", FailureCategory.TEST_BUG,
        "StaleElementReferenceException", FailureCategory.TEST_BUG,
        "ElementClickInterceptedException", FailureCategory.TEST_BUG
    );

    public FailureCategory classify(Throwable exception) {
        String message = exception.getClass().getSimpleName()
            + ": " + exception.getMessage();

        return RULES.entrySet().stream()
            .filter(entry -> message.contains(entry.getKey()))
            .map(Map.Entry::getValue)
            .findFirst()
            .orElse(FailureCategory.UNKNOWN);
    }
}

// Категории падений
public enum FailureCategory {
    PRODUCT_BUG,    // Баг в продукте — нужен bug report
    TEST_BUG,       // Ошибка в тесте — нужен fix теста
    ENVIRONMENT,    // Проблема окружения — нужна диагностика инфраструктуры
    FLAKY,          // Нестабильный тест — нужна стабилизация
    UNKNOWN         // Не удалось определить — требуется ручной анализ
}
```

### Шаг 2: Screenshot и Video

Для UI-тестов скриншот в момент падения — бесценный артефакт. Видеозапись позволяет увидеть
всю последовательность действий.

```java
/**
 * Автоматическое снятие скриншота при падении теста.
 * JUnit 5 Extension для интеграции с Allure.
 */
public class ScreenshotOnFailureExtension implements TestWatcher {

    @Override
    public void testFailed(ExtensionContext context, Throwable cause) {
        // Получаем WebDriver из контекста
        WebDriver driver = getDriverFromContext(context);
        if (driver != null) {
            attachScreenshot(driver, "Скриншот при падении");
            attachPageSource(driver, "HTML страницы при падении");
            attachBrowserLogs(driver, "Логи браузера");
        }
    }

    @Attachment(value = "{name}", type = "image/png")
    private byte[] attachScreenshot(WebDriver driver, String name) {
        return ((TakesScreenshot) driver).getScreenshotAs(OutputType.BYTES);
    }

    @Attachment(value = "{name}", type = "text/html")
    private String attachPageSource(WebDriver driver, String name) {
        return driver.getPageSource();
    }

    @Attachment(value = "{name}", type = "text/plain")
    private String attachBrowserLogs(WebDriver driver, String name) {
        try {
            return driver.manage().logs()
                .get(LogType.BROWSER)
                .getAll()
                .stream()
                .map(entry -> entry.getLevel() + ": " + entry.getMessage())
                .collect(Collectors.joining("\n"));
        } catch (Exception e) {
            return "Не удалось получить логи браузера: " + e.getMessage();
        }
    }

    private WebDriver getDriverFromContext(ExtensionContext context) {
        return context.getStore(ExtensionContext.Namespace.GLOBAL)
            .get("webdriver", WebDriver.class);
    }
}
```

**На что обращать внимание в скриншоте:**
- Правильная ли это страница? (может быть редирект на логин)
- Есть ли popup/overlay/spinner, перекрывающий элемент?
- Загружены ли данные? (пустая таблица, спиннер загрузки)
- Есть ли сообщение об ошибке на странице?
- Корректна ли верстка? (может быть проблема с разрешением)

### Шаг 3: Логи приложения

Логи серверного приложения часто содержат корневую причину ошибки, которую невозможно
увидеть со стороны клиента.

```java
/**
 * Утилита для сбора логов приложения после падения теста.
 * Подключается к серверу и извлекает логи за время выполнения теста.
 */
public class LogCollector {

    private final String logEndpoint; // Actuator endpoint или Kibana API

    public LogCollector(String logEndpoint) {
        this.logEndpoint = logEndpoint;
    }

    // Получение логов за указанный период
    @Step("Сбор логов приложения за период {from} — {to}")
    public String collectLogs(Instant from, Instant to) {
        try {
            HttpClient client = HttpClient.newHttpClient();
            String url = String.format(
                "%s?from=%s&to=%s&level=ERROR,WARN",
                logEndpoint, from, to
            );
            HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(url))
                .GET()
                .build();
            HttpResponse<String> response = client.send(
                request, HttpResponse.BodyHandlers.ofString()
            );
            return response.body();
        } catch (Exception e) {
            return "Не удалось получить логи: " + e.getMessage();
        }
    }

    // Прикрепление логов к Allure-отчёту
    @Attachment(value = "Логи приложения", type = "text/plain")
    public String attachLogs(Instant from, Instant to) {
        return collectLogs(from, to);
    }
}
```

**Где искать логи:**
- Actuator endpoint: `GET /actuator/logfile`
- Kibana/Elasticsearch: поиск по correlationId
- Docker: `docker logs <container_name> --since <timestamp>`
- Jenkins: артефакты билда, console output

### Шаг 4: Allure Steps

Allure Steps показывают, на каком шаге упал тест. Это позволяет точно определить,
какое действие вызвало ошибку.

```
✅ Шаг 1: Открыть страницу логина
✅ Шаг 2: Ввести логин "admin"
✅ Шаг 3: Ввести пароль
✅ Шаг 4: Нажать кнопку "Войти"
✅ Шаг 5: Дождаться загрузки дашборда
✅ Шаг 6: Перейти в раздел "Заказы"
❌ Шаг 7: Нажать кнопку "Создать заказ"
    └── ElementClickInterceptedException: element click intercepted...
```

Из этого отчёта понятно: логин прошёл, дашборд загрузился, проблема возникла при создании
заказа на странице заказов.

### Шаг 5: Состояние окружения

Проверяем, было ли окружение исправно в момент падения.

**Чек-лист:**
- Доступен ли тестовый стенд? (`curl -s http://test-env:8080/actuator/health`)
- Доступна ли база данных? (`pg_isready -h db-host -p 5432`)
- Были ли деплои в момент прогона тестов? (проверить CI/CD историю)
- Достаточно ли ресурсов? (CPU, RAM, disk space)
- Есть ли сетевые проблемы? (latency, packet loss)

### Шаг 6: Тестовые данные

Проверяем, существуют ли необходимые тестовые данные, не были ли они повреждены
или удалены.

```java
/**
 * Утилита для валидации тестовых данных при диагностике падений.
 */
public class TestDataDiagnostic {

    private final JdbcTemplate db;

    // Проверка существования тестового пользователя
    public boolean isTestUserExists(String userId) {
        Integer count = db.queryForObject(
            "SELECT COUNT(*) FROM users WHERE id = ?",
            Integer.class, userId
        );
        return count != null && count > 0;
    }

    // Проверка состояния заказа
    public String getOrderStatus(String orderId) {
        return db.queryForObject(
            "SELECT status FROM orders WHERE id = ?",
            String.class, orderId
        );
    }

    // Полный снимок данных для диагностики
    @Step("Снимок тестовых данных для пользователя {userId}")
    public Map<String, Object> getDataSnapshot(String userId) {
        Map<String, Object> snapshot = new LinkedHashMap<>();
        snapshot.put("user", db.queryForMap(
            "SELECT * FROM users WHERE id = ?", userId));
        snapshot.put("orders", db.queryForList(
            "SELECT * FROM orders WHERE user_id = ?", userId));
        snapshot.put("cart", db.queryForList(
            "SELECT * FROM cart_items WHERE user_id = ?", userId));
        return snapshot;
    }
}
```

### Шаг 7: Воспроизведение локально

Если удалённый анализ не дал результата — воспроизводим падение локально.

```bash
# 1. Убедиться, что локальная версия кода совпадает с CI
git checkout <commit-hash-from-ci>

# 2. Запустить конкретный тест
mvn test -Dtest=OrderTest#testCreateOrder -Dtest.env=local

# 3. Запустить с verbose-логированием
mvn test -Dtest=OrderTest#testCreateOrder \
    -Dlogback.configurationFile=logback-debug.xml

# 4. Запустить тест несколько раз (для выявления flaky)
for i in {1..10}; do
    echo "=== Попытка $i ==="
    mvn test -Dtest=OrderTest#testCreateOrder
    echo "Exit code: $?"
done
```

### Шаг 8: Классификация и документирование

По результатам анализа классифицируем падение и принимаем решение.

```
┌─────────────────────────────────────────────────────────┐
│              МАТРИЦА ПРИНЯТИЯ РЕШЕНИЙ                    │
│                                                         │
│  Product Bug  → Завести баг в Jira, приложить evidence  │
│  Test Bug     → Исправить тест, закоммитить             │
│  Environment  → Уведомить DevOps, перезапустить pipeline│
│  Flaky        → Добавить в карантин, создать задачу     │
│  Unknown      → Эскалировать, собрать больше данных     │
└─────────────────────────────────────────────────────────┘
```

---

## Allure History и Trends

### Использование истории для анализа

Allure сохраняет историю результатов тестов между прогонами. Это позволяет выявлять паттерны.

**Trend Dashboard показывает:**
- Общий тренд: растёт или снижается количество падений?
- Появились ли новые падения после конкретного коммита?
- Какие тесты стали чаще падать?

**Retry Tab показывает:**
- Какие тесты прошли только с retry?
- Сколько попыток потребовалось?
- Совпадают ли ошибки между попытками?

```java
/**
 * Интеграция Allure History с CI.
 * Копирование истории для формирования трендов между прогонами.
 */
public class AllureHistoryManager {

    // Копирование истории из предыдущего отчёта
    public void copyHistoryFromPreviousReport(
            Path previousReportDir, Path currentResultsDir) {
        Path historySource = previousReportDir.resolve("history");
        Path historyTarget = currentResultsDir.resolve("history");

        if (Files.exists(historySource)) {
            try {
                // Создаём директорию history в allure-results
                Files.createDirectories(historyTarget);
                // Копируем файлы истории
                try (var files = Files.list(historySource)) {
                    files.forEach(file -> {
                        try {
                            Files.copy(file,
                                historyTarget.resolve(file.getFileName()),
                                StandardCopyOption.REPLACE_EXISTING);
                        } catch (IOException e) {
                            System.err.println(
                                "Не удалось скопировать файл истории: " + file);
                        }
                    });
                }
            } catch (IOException e) {
                System.err.println("Не удалось скопировать историю Allure");
            }
        }
    }
}
```

---

## Типичные паттерны падений

### Паттерн 1: Массовое падение

**Симптом:** 80%+ тестов упали одновременно.

**Вероятные причины:**
- Стенд недоступен (все тесты получили `Connection refused`)
- Деплой сломал стенд (все тесты получили `500` или не могут авторизоваться)
- Истёк SSL-сертификат тестового стенда
- Изменился формат авторизации — все тесты получают `401`

**Действие:** Не анализировать каждый тест — найти общую причину.

### Паттерн 2: Один тест стабильно падает

**Симптом:** Конкретный тест падает в каждом прогоне, остальные проходят.

**Вероятные причины:**
- Баг в приложении в конкретной функции
- Тест устарел после изменения требований
- Локатор или API endpoint изменился

**Действие:** Воспроизвести вручную, проверить требования.

### Паттерн 3: Тесты падают хаотично (разные тесты в разных прогонах)

**Симптом:** В каждом прогоне падают 3-5 тестов, но каждый раз — разные.

**Вероятные причины:**
- Flaky-тесты (timing issues, shared state)
- Нестабильное окружение (ресурсы на пределе)
- Проблемы параллельного выполнения

**Действие:** Проанализировать историю, запустить каждый тест 10 раз.

### Паттерн 4: Тесты падают только в CI

**Симптом:** Локально все тесты зелёные, в CI — падения.

**Вероятные причины:**
- Различия в окружении (headless browser, разрешение экрана)
- Нехватка ресурсов в CI-контейнере
- Параллельный запуск (локально — последовательный)
- Разные версии зависимостей

**Действие:** Сравнить конфигурации, проверить ресурсы CI.

### Паттерн 5: Тесты падают после деплоя конкретного сервиса

**Симптом:** Тесты стабильно проходили, после деплоя — 15 падений в модуле X.

**Вероятные причины:**
- Баг, внесённый в этом деплое
- Breaking change в API (изменение контракта)
- Изменение конфигурации (feature flag, настройки)

**Действие:** Проверить changelog деплоя, сравнить API-ответы до и после.

---

## Инструменты для диагностики

### Чек-лист инструментов

| Инструмент | Для чего | Команда / URL |
|------------|----------|---------------|
| Allure Report | Шаги, скриншоты, история | `allure serve allure-results` |
| Kibana / ELK | Логи приложения | `https://kibana.company.com` |
| Grafana | Метрики окружения | `https://grafana.company.com` |
| Docker logs | Логи контейнеров | `docker logs <container>` |
| Jenkins Console | CI-логи прогона | Артефакты билда |
| Database client | Проверка тестовых данных | DBeaver, DataGrip |
| Browser DevTools | Network, Console, Elements | F12 в браузере |

### Автоматический сбор диагностики

```java
/**
 * Автоматический сбор всей диагностической информации при падении теста.
 * Прикрепляет артефакты к Allure-отчёту.
 */
public class DiagnosticsCollector implements TestWatcher {

    private Instant testStartTime;

    @Override
    public void testFailed(ExtensionContext context, Throwable cause) {
        Instant testEndTime = Instant.now();

        // 1. Классификация ошибки
        FailureCategory category = new FailureClassifier().classify(cause);
        Allure.label("failureCategory", category.name());

        // 2. Скриншот (для UI-тестов)
        getDriver(context).ifPresent(driver -> {
            attachScreenshot(driver);
            attachPageSource(driver);
            attachBrowserLogs(driver);
        });

        // 3. Логи приложения
        attachApplicationLogs(testStartTime, testEndTime);

        // 4. Информация об окружении
        attachEnvironmentInfo();

        // 5. Снимок тестовых данных
        attachTestDataSnapshot(context);
    }

    @Attachment(value = "Скриншот", type = "image/png")
    private byte[] attachScreenshot(WebDriver driver) {
        return ((TakesScreenshot) driver).getScreenshotAs(OutputType.BYTES);
    }

    @Attachment(value = "Page Source", type = "text/html")
    private String attachPageSource(WebDriver driver) {
        return driver.getPageSource();
    }

    @Attachment(value = "Browser Logs", type = "text/plain")
    private String attachBrowserLogs(WebDriver driver) {
        return driver.manage().logs().get(LogType.BROWSER)
            .getAll().stream()
            .map(LogEntry::toString)
            .collect(Collectors.joining("\n"));
    }

    @Attachment(value = "Application Logs", type = "text/plain")
    private String attachApplicationLogs(Instant from, Instant to) {
        return new LogCollector(EnvironmentConfig.getLogEndpoint())
            .collectLogs(from, to);
    }

    @Attachment(value = "Environment Info", type = "text/plain")
    private String attachEnvironmentInfo() {
        return String.format(
            "Environment: %s\nBase URL: %s\nBrowser: %s\nJava: %s\nOS: %s",
            EnvironmentConfig.getEnvName(),
            EnvironmentConfig.getBaseUrl(),
            System.getProperty("browser", "chrome"),
            System.getProperty("java.version"),
            System.getProperty("os.name")
        );
    }

    @Attachment(value = "Test Data Snapshot", type = "application/json")
    private String attachTestDataSnapshot(ExtensionContext context) {
        // Сбор данных, специфичных для теста
        return "{}"; // Конкретная реализация зависит от контекста
    }

    private Optional<WebDriver> getDriver(ExtensionContext context) {
        return Optional.ofNullable(
            context.getStore(ExtensionContext.Namespace.GLOBAL)
                .get("webdriver", WebDriver.class)
        );
    }
}
```

---

## Связь с тестированием

Эффективный анализ падений — это основа доверия к автотестам:

- **Скорость обратной связи** — быстрая диагностика ускоряет цикл разработки
- **Точность баг-репортов** — правильная классификация экономит время разработчиков
- **Доверие к test suite** — если каждое падение объяснено, команда доверяет результатам
- **Улучшение процессов** — статистика падений выявляет системные проблемы
- **Развитие навыков** — анализ ошибок углубляет понимание системы

---

## Типичные ошибки

1. **Сразу перезапускать упавший тест** без анализа — можно пропустить реальный баг
2. **Анализировать каждый тест при массовом падении** — сначала ищите общую причину
3. **Не собирать артефакты** (скриншоты, логи) — без evidence диагностика невозможна
4. **Не использовать историю Allure** — повторяющиеся паттерны выявляются только через историю
5. **Не классифицировать падения** — все падения «в одну кучу» не дают статистики
6. **Игнорировать environment issues** — «тесты падают, потому что стенд лежит» — тоже нужно фиксировать
7. **Не документировать результаты анализа** — один и тот же анализ повторяется многократно
8. **Не автоматизировать сбор диагностики** — ручной сбор скриншотов и логов занимает слишком много времени

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Тест упал с `NoSuchElementException`. Какие могут быть причины?
2. Чем отличается баг в продукте от бага в тесте? Как их различить?
3. Зачем снимать скриншот при падении теста?
4. Что такое Allure Report и какую информацию он предоставляет?

### 🟡 Средний уровень
5. Опишите ваш алгоритм анализа упавшего теста — пошагово.
6. 80% тестов упали в одном прогоне. Ваши действия?
7. Тест проходит локально, но падает в CI. Как будете расследовать?
8. Как организовать автоматический сбор диагностической информации?
9. Как отличить flaky-тест от реального бага?

### 🔴 Продвинутый уровень
10. Спроектируйте систему автоматической классификации падений тестов.
11. Как организовать post-mortem анализ для тестового pipeline?
12. Как коррелировать падения тестов с деплоями для автоматического определения виновного коммита?
13. Опишите метрики, которые вы бы собирали для анализа эффективности test suite.
14. Как построить dashboard для мониторинга здоровья test suite в реальном времени?

---

## Практические задания

### Задание 1: Failure Classifier
Реализуйте класс `FailureClassifier`, который:
- Принимает `Throwable` и контекст теста
- Анализирует тип исключения, сообщение, stack trace
- Возвращает категорию: PRODUCT_BUG, TEST_BUG, ENVIRONMENT, FLAKY
- Прикрепляет результат классификации к Allure-отчёту

### Задание 2: DiagnosticsCollector Extension
Создайте JUnit 5 Extension, который при падении теста автоматически:
- Снимает скриншот (для UI-тестов)
- Собирает логи приложения за время выполнения теста
- Записывает состояние окружения
- Прикрепляет всё к Allure-отчёту
- Выполняет первичную классификацию ошибки

### Задание 3: Анализ по скриншоту
Вам дан следующий сценарий: тест на оформление заказа падает с `TimeoutException`
на шаге «Нажать кнопку Оформить заказ». На скриншоте видна страница с формой заказа,
но поверх формы — полупрозрачный overlay спиннера загрузки.
Опишите:
- Корневую причину
- Категорию падения
- Способ исправления теста
- Нужно ли заводить баг в продукте?

### Задание 4: Dashboard метрик
Спроектируйте структуру данных и запросы для dashboard, который показывает:
- Распределение падений по категориям за последние 7 дней
- Топ-10 самых часто падающих тестов
- Тренд: количество падений по дням
- Среднее время диагностики (от падения до классификации)

---

## Дополнительные ресурсы

- [Allure Framework: History & Retries](https://docs.qameta.io/allure/) — работа с историей в Allure
- [JUnit 5 Extensions Model](https://junit.org/junit5/docs/current/user-guide/#extensions) — создание расширений
- [Selenium: Logging](https://www.selenium.dev/documentation/webdriver/troubleshooting/logging/) — логирование в Selenium
- [ELK Stack](https://www.elastic.co/elastic-stack) — централизованное логирование
- [Grafana](https://grafana.com/) — визуализация метрик и dashboard
- Google SRE Book: "Monitoring Distributed Systems" — принципы мониторинга, применимые к тестам
