# Другие системы репортинга

## Обзор

Помимо Allure и ReportPortal, в мире автоматизации тестирования существует ряд других
инструментов для генерации отчётов. Каждый из них имеет свои сильные стороны и области
применения. QA-инженеру важно знать о существовании этих инструментов, понимать их
особенности и уметь выбрать подходящий для конкретного проекта.

В этом разделе рассматриваются:
- **ExtentReports** — популярная библиотека для красивых HTML-отчётов в Java
- **Cucumber Reports** — отчёты для BDD-проектов на базе Cucumber
- **pytest-html** — отчёты для Python-тестов (для общего кругозора)
- **Кастомные HTML-отчёты** — когда стандартных инструментов недостаточно
- **Уведомления** — интеграция с Slack и Telegram для оповещения о результатах

---

## ExtentReports

### Что такое ExtentReports

ExtentReports — это Java-библиотека для генерации красивых HTML-отчётов о тестировании.
Разработана Anshoo Arora, широко используется в Java-проектах, особенно в связке
с TestNG и Selenium. ExtentReports 5 — текущая мажорная версия.

### Основные возможности

- Красивые HTML-отчёты с современным дизайном
- Поддержка вложений (скриншоты, видео)
- Категории и теги для группировки тестов
- Системная информация (окружение, ОС, браузер)
- Параллельная генерация отчётов (thread-safe)
- Кастомизация через CSS/JS

### Зависимости Maven

```xml
<dependency>
    <groupId>com.aventstack</groupId>
    <artifactId>extentreports</artifactId>
    <version>5.1.1</version>
    <scope>test</scope>
</dependency>
```

### Базовая настройка

```java
import com.aventstack.extentreports.ExtentReports;
import com.aventstack.extentreports.ExtentTest;
import com.aventstack.extentreports.Status;
import com.aventstack.extentreports.reporter.ExtentSparkReporter;
import com.aventstack.extentreports.reporter.configuration.Theme;

public class ExtentReportManager {

    private static ExtentReports extent;
    // ThreadLocal для поддержки параллельного запуска
    private static final ThreadLocal<ExtentTest> testThread = new ThreadLocal<>();

    // Инициализация отчёта (вызывается один раз перед всеми тестами)
    public static ExtentReports getInstance() {
        if (extent == null) {
            ExtentSparkReporter spark = new ExtentSparkReporter(
                    "target/extent-report.html");

            // Настройка внешнего вида
            spark.config().setTheme(Theme.DARK);
            spark.config().setDocumentTitle("Отчёт о тестировании");
            spark.config().setReportName("Regression Tests");
            spark.config().setTimeStampFormat("dd.MM.yyyy HH:mm:ss");
            spark.config().setEncoding("UTF-8");

            extent = new ExtentReports();
            extent.attachReporter(spark);

            // Системная информация
            extent.setSystemInfo("OS", System.getProperty("os.name"));
            extent.setSystemInfo("Java Version",
                    System.getProperty("java.version"));
            extent.setSystemInfo("Browser", "Chrome 120");
            extent.setSystemInfo("Environment", "Staging");
        }
        return extent;
    }

    // Создание теста
    public static ExtentTest createTest(String testName, String description) {
        ExtentTest test = getInstance().createTest(testName, description);
        testThread.set(test);
        return test;
    }

    // Получение текущего теста
    public static ExtentTest getTest() {
        return testThread.get();
    }

    // Финализация отчёта (вызывается один раз после всех тестов)
    public static void flush() {
        if (extent != null) {
            extent.flush();
        }
    }
}
```

### Использование с JUnit 5

```java
import com.aventstack.extentreports.ExtentTest;
import com.aventstack.extentreports.Status;
import com.aventstack.extentreports.MediaEntityBuilder;
import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.*;

// Расширение JUnit 5 для ExtentReports
public class ExtentReportExtension implements
        BeforeEachCallback, AfterEachCallback, AfterAllCallback, TestWatcher {

    @Override
    public void beforeEach(ExtensionContext context) {
        String testName = context.getDisplayName();
        String className = context.getRequiredTestClass().getSimpleName();
        ExtentReportManager.createTest(className + " :: " + testName);
    }

    @Override
    public void testSuccessful(ExtensionContext context) {
        ExtentReportManager.getTest().pass("Тест прошёл успешно");
    }

    @Override
    public void testFailed(ExtensionContext context, Throwable cause) {
        ExtentTest test = ExtentReportManager.getTest();
        test.fail("Тест упал: " + cause.getMessage());
        test.fail(cause);

        // Прикрепляем скриншот при падении
        Object testInstance = context.getRequiredTestInstance();
        if (testInstance instanceof BaseTest baseTest) {
            try {
                String screenshotPath = baseTest.takeScreenshot();
                test.fail(MediaEntityBuilder
                        .createScreenCaptureFromPath(screenshotPath)
                        .build());
            } catch (Exception e) {
                test.warning("Не удалось прикрепить скриншот: " + e.getMessage());
            }
        }
    }

    @Override
    public void testDisabled(ExtensionContext context, java.util.Optional<String> reason) {
        ExtentReportManager.getTest().skip(
                "Тест пропущен: " + reason.orElse("причина не указана"));
    }

    @Override
    public void afterEach(ExtensionContext context) {
        // Дополнительная логика после каждого теста, если необходимо
    }

    @Override
    public void afterAll(ExtensionContext context) {
        ExtentReportManager.flush();
    }
}
```

### Логирование шагов

```java
import com.aventstack.extentreports.Status;

@ExtendWith(ExtentReportExtension.class)
public class LoginTests extends BaseTest {

    @Test
    @DisplayName("Успешная авторизация")
    void testSuccessfulLogin() {
        ExtentTest test = ExtentReportManager.getTest();

        test.log(Status.INFO, "Открываем страницу авторизации");
        driver.get("https://example.com/login");

        test.log(Status.INFO, "Вводим логин: user@example.com");
        driver.findElement(By.id("username")).sendKeys("user@example.com");

        test.log(Status.INFO, "Вводим пароль");
        driver.findElement(By.id("password")).sendKeys("password123");

        test.log(Status.INFO, "Нажимаем кнопку 'Войти'");
        driver.findElement(By.id("login-btn")).click();

        test.log(Status.PASS, "Переход на главную страницу выполнен");
    }
}
```

### ExtentReports vs Allure: сравнение

| Характеристика | ExtentReports | Allure Report |
|---|---|---|
| **Подход** | Программный API | Аннотации + аспекты |
| **Шаги** | Вручную через `test.log()` | Автоматически через `@Step` |
| **Интеграция** | Требует ручной настройки | Готовые адаптеры для фреймворков |
| **BDD-группировка** | Категории и теги | `@Epic`, `@Feature`, `@Story` |
| **Тренды** | Нет | Да (через `history/`) |
| **CI/CD интеграция** | Базовая | Allure CLI, Jenkins plugin |
| **Кастомизация** | CSS/JS | Плагины |
| **Сообщество** | Среднее | Большое |
| **Популярность в Java** | Средняя | Высокая (де-факто стандарт) |

---

## Cucumber Reports

### Обзор

Cucumber — BDD-фреймворк, который генерирует отчёты в нескольких форматах.
Для проектов, использующих Cucumber, отчёты являются важной частью BDD-процесса,
так как они должны быть читаемы не только для инженеров, но и для бизнес-аналитиков.

### Форматы отчётов Cucumber

```java
// Настройка плагинов отчётности в Cucumber Runner (JUnit 5)
@Suite
@IncludeEngines("cucumber")
@SelectPackages("com.example.features")
@ConfigurationParameter(key = PLUGIN_PROPERTY_NAME,
        value = "pretty,"                                          // Консольный вывод
             + "html:target/cucumber-reports/cucumber.html,"       // Встроенный HTML
             + "json:target/cucumber-reports/cucumber.json,"       // JSON (для других инструментов)
             + "junit:target/cucumber-reports/cucumber.xml,"       // JUnit XML
             + "io.qameta.allure.cucumber7jvm.AllureCucumber7Jvm") // Allure-интеграция
public class RunCucumberTests {
}
```

### Cucumber HTML Report (встроенный)

Cucumber из коробки генерирует простой HTML-отчёт. Он минималистичен, но содержит
все необходимые данные: сценарии, шаги, статусы, длительность.

### Masterthought Cucumber Reports

Для более красивых отчётов часто используется библиотека `cucumber-reporting`
от Masterthought:

```xml
<dependency>
    <groupId>net.masterthought</groupId>
    <artifactId>cucumber-reporting</artifactId>
    <version>5.7.8</version>
    <scope>test</scope>
</dependency>
```

```java
import net.masterthought.cucumber.Configuration;
import net.masterthought.cucumber.ReportBuilder;

import java.io.File;
import java.util.List;

public class CucumberReportGenerator {

    // Генерация красивого HTML-отчёта из JSON-результатов Cucumber
    public static void generateReport() {
        File reportOutputDirectory = new File("target/cucumber-advanced-report");
        List<String> jsonFiles = List.of("target/cucumber-reports/cucumber.json");

        Configuration configuration = new Configuration(
                reportOutputDirectory, "My Project");
        configuration.setBuildNumber("Build #42");

        // Добавляем классификацию (информация об окружении)
        configuration.addClassifications("Platform", "Linux");
        configuration.addClassifications("Browser", "Chrome");
        configuration.addClassifications("Environment", "Staging");
        configuration.addClassifications("Branch", "develop");

        ReportBuilder reportBuilder = new ReportBuilder(jsonFiles, configuration);
        reportBuilder.generateReports();
    }
}
```

### Allure + Cucumber

Наиболее мощная комбинация — использование Allure как отчётного движка для Cucumber:

```xml
<dependency>
    <groupId>io.qameta.allure</groupId>
    <artifactId>allure-cucumber7-jvm</artifactId>
    <version>2.25.0</version>
    <scope>test</scope>
</dependency>
```

Преимущества этой связки:
- Сценарии Cucumber автоматически становятся тестами в Allure
- Шаги `Given/When/Then` отображаются как `@Step` в отчёте
- Можно использовать все аннотации Allure (`@Epic`, `@Feature`, `@Story`)
- BDD-вкладка Behaviors заполняется данными из feature-файлов

---

## pytest-html (для контекста)

### Обзор

`pytest-html` — плагин для Python-фреймворка pytest, генерирующий HTML-отчёты.
QA-инженеру полезно знать о нём, так как в смешанных командах Python-тесты
встречаются часто.

```bash
# Установка
pip install pytest-html

# Генерация отчёта
pytest --html=report.html --self-contained-html
```

### Особенности

- Простой и лёгкий — минимум настроек
- Self-contained HTML — один файл со всеми стилями и данными
- Скриншоты — через conftest.py хуки
- Метаданные — информация об окружении в заголовке отчёта

Для Python-проектов также широко используется Allure (`allure-pytest`),
который обеспечивает аналогичную функциональность как и `allure-junit5` для Java.

---

## Кастомные HTML-отчёты

### Когда нужен кастомный отчёт

Иногда стандартных инструментов недостаточно:
- Нестандартный формат данных (например, результаты нагрузочного тестирования)
- Специфические требования бизнеса (корпоративный стиль, определённые метрики)
- Интеграция с внутренними системами
- Агрегация результатов из разных источников (JUnit + API-мониторинг + нагрузка)

### Пример генерации кастомного HTML-отчёта

```java
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.List;

/**
 * Генератор простого кастомного HTML-отчёта.
 * Полезен, когда нужен минимальный отчёт без тяжёлых зависимостей.
 */
public class CustomHtmlReportGenerator {

    // Контейнер для результатов теста
    public record TestResult(String name, String status,
                             long durationMs, String errorMessage) {}

    // Генерация HTML-отчёта из списка результатов
    public static void generateReport(List<TestResult> results, Path outputPath)
            throws IOException {

        long total = results.size();
        long passed = results.stream()
                .filter(r -> "PASSED".equals(r.status())).count();
        long failed = results.stream()
                .filter(r -> "FAILED".equals(r.status())).count();
        long skipped = total - passed - failed;
        double passRate = total > 0 ? (double) passed / total * 100 : 0;

        // Генерируем строки таблицы
        StringBuilder tableRows = new StringBuilder();
        for (TestResult result : results) {
            String rowClass = switch (result.status()) {
                case "PASSED" -> "passed";
                case "FAILED" -> "failed";
                default -> "skipped";
            };
            tableRows.append(String.format("""
                <tr class="%s">
                    <td>%s</td>
                    <td>%s</td>
                    <td>%d ms</td>
                    <td>%s</td>
                </tr>
                """, rowClass, result.name(), result.status(),
                    result.durationMs(),
                    result.errorMessage() != null ? result.errorMessage() : "—"));
        }

        // Формируем HTML
        String html = String.format("""
            <!DOCTYPE html>
            <html lang="ru">
            <head>
                <meta charset="UTF-8">
                <title>Отчёт о тестировании</title>
                <style>
                    body { font-family: Arial, sans-serif; margin: 20px; }
                    .summary { display: flex; gap: 20px; margin: 20px 0; }
                    .summary .card {
                        padding: 15px; border-radius: 8px;
                        color: white; min-width: 120px; text-align: center;
                    }
                    .card.total { background: #2196F3; }
                    .card.passed { background: #4CAF50; }
                    .card.failed { background: #f44336; }
                    .card.skipped { background: #9E9E9E; }
                    table { border-collapse: collapse; width: 100%%; }
                    th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
                    th { background: #333; color: white; }
                    tr.passed { background: #E8F5E9; }
                    tr.failed { background: #FFEBEE; }
                    tr.skipped { background: #F5F5F5; }
                </style>
            </head>
            <body>
                <h1>Отчёт о тестировании</h1>
                <p>Дата: %s</p>
                <div class="summary">
                    <div class="card total">Всего: %d</div>
                    <div class="card passed">Прошло: %d</div>
                    <div class="card failed">Упало: %d</div>
                    <div class="card skipped">Пропущено: %d</div>
                </div>
                <p>Pass Rate: %.1f%%</p>
                <table>
                    <tr><th>Тест</th><th>Статус</th><th>Время</th><th>Ошибка</th></tr>
                    %s
                </table>
            </body>
            </html>
            """,
            LocalDateTime.now().format(
                    DateTimeFormatter.ofPattern("dd.MM.yyyy HH:mm:ss")),
            total, passed, failed, skipped, passRate, tableRows);

        Files.writeString(outputPath, html);
    }
}
```

---

## Уведомления: Slack и Telegram

### Зачем нужны уведомления

Отчёт бесполезен, если никто не знает о его существовании. Уведомления в мессенджеры
решают эту проблему:
- Команда мгновенно узнаёт о падениях тестов
- Ссылка на отчёт приходит прямо в рабочий чат
- Краткая статистика видна без открытия отчёта

### Slack-уведомления

#### Через Slack Incoming Webhook

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;

public class SlackNotifier {

    private final String webhookUrl;
    private final HttpClient httpClient;

    public SlackNotifier(String webhookUrl) {
        this.webhookUrl = webhookUrl;
        this.httpClient = HttpClient.newHttpClient();
    }

    // Отправка уведомления о результатах тестирования
    public void sendTestResults(int total, int passed, int failed,
                                String reportUrl) {
        // Выбираем эмодзи и цвет в зависимости от результата
        String color = failed == 0 ? "#36a64f" : "#ff0000";
        String emoji = failed == 0 ? ":white_check_mark:" : ":x:";

        String payload = String.format("""
            {
                "attachments": [{
                    "color": "%s",
                    "blocks": [
                        {
                            "type": "header",
                            "text": {
                                "type": "plain_text",
                                "text": "%s Результаты тестирования"
                            }
                        },
                        {
                            "type": "section",
                            "fields": [
                                {"type": "mrkdwn", "text": "*Всего:* %d"},
                                {"type": "mrkdwn", "text": "*Прошло:* %d"},
                                {"type": "mrkdwn", "text": "*Упало:* %d"},
                                {"type": "mrkdwn", "text": "*Pass Rate:* %.1f%%"}
                            ]
                        },
                        {
                            "type": "actions",
                            "elements": [{
                                "type": "button",
                                "text": {"type": "plain_text", "text": "Открыть отчёт"},
                                "url": "%s"
                            }]
                        }
                    ]
                }]
            }
            """, color, emoji, total, passed, failed,
                (double) passed / total * 100, reportUrl);

        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(webhookUrl))
                .header("Content-Type", "application/json")
                .POST(HttpRequest.BodyPublishers.ofString(payload))
                .build();

        try {
            httpClient.send(request, HttpResponse.BodyHandlers.ofString());
        } catch (Exception e) {
            System.err.println("Не удалось отправить уведомление в Slack: "
                    + e.getMessage());
        }
    }
}
```

### Telegram-уведомления

#### Через Telegram Bot API

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.net.URLEncoder;
import java.nio.charset.StandardCharsets;

public class TelegramNotifier {

    private final String botToken;
    private final String chatId;
    private final HttpClient httpClient;

    public TelegramNotifier(String botToken, String chatId) {
        this.botToken = botToken;
        this.chatId = chatId;
        this.httpClient = HttpClient.newHttpClient();
    }

    // Отправка результатов тестирования в Telegram
    public void sendTestResults(int total, int passed, int failed,
                                String reportUrl) {
        String status = failed == 0 ? "PASSED" : "FAILED";
        String statusIcon = failed == 0 ? "\u2705" : "\u274C";

        String message = String.format("""
            %s *Результаты тестирования*

            *Статус:* %s
            *Всего тестов:* %d
            *Прошло:* %d
            *Упало:* %d
            *Pass Rate:* %.1f%%

            [Открыть отчёт](%s)
            """,
                statusIcon, status, total, passed, failed,
                (double) passed / total * 100, reportUrl);

        String encodedMessage = URLEncoder.encode(
                message, StandardCharsets.UTF_8);
        String url = String.format(
                "https://api.telegram.org/bot%s/sendMessage?chat_id=%s"
                + "&text=%s&parse_mode=Markdown",
                botToken, chatId, encodedMessage);

        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(url))
                .GET()
                .build();

        try {
            httpClient.send(request, HttpResponse.BodyHandlers.ofString());
        } catch (Exception e) {
            System.err.println("Не удалось отправить уведомление в Telegram: "
                    + e.getMessage());
        }
    }
}
```

### Интеграция уведомлений с CI/CD

В CI/CD уведомления обычно отправляются на этапе `post` (after_script):

```yaml
# GitHub Actions
- name: Send Slack notification
  if: always()
  env:
    SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
  run: |
    TOTAL=$(grep -c 'testcase' target/surefire-reports/*.xml || echo 0)
    FAILED=$(grep -c 'failure' target/surefire-reports/*.xml || echo 0)
    PASSED=$((TOTAL - FAILED))
    REPORT_URL="https://company.github.io/project/allure-report"

    curl -X POST "$SLACK_WEBHOOK" \
      -H "Content-Type: application/json" \
      -d "{\"text\": \"Tests: $PASSED/$TOTAL passed. Report: $REPORT_URL\"}"
```

Существуют также готовые GitHub Actions для уведомлений:
- `8398a7/action-slack` — Slack-уведомления
- `appleboy/telegram-action` — Telegram-уведомления

---

## Сводная таблица инструментов репортинга

| Инструмент | Тип | Язык | Лицензия | Real-time | ML | TMS | Сложность |
|---|---|---|---|---|---|---|---|
| **Allure Report** | Генератор HTML | Любой | Open Source | Нет | Нет | Нет | Низкая |
| **Allure TestOps** | TMS + отчёты | Любой | Коммерч. | Да | Базовый | Да | Средняя |
| **ReportPortal** | Платформа | Любой | Open Source | Да | Да | Нет | Высокая |
| **ExtentReports** | Библиотека | Java/.NET | Open Source | Нет | Нет | Нет | Низкая |
| **Cucumber Reports** | Генератор HTML | Любой (BDD) | Open Source | Нет | Нет | Нет | Низкая |
| **pytest-html** | Плагин pytest | Python | Open Source | Нет | Нет | Нет | Низкая |
| **Кастомный HTML** | Самописный | Любой | — | — | — | — | Средняя |

---

## Когда выбирать какой инструмент

### Дерево принятия решений

```
Нужен ли real-time мониторинг?
├── Да → Есть бюджет на коммерческий продукт?
│         ├── Да → Allure TestOps (TMS + real-time + Jira)
│         └── Нет → ReportPortal (open source, но тяжёлый)
└── Нет → Это BDD-проект с Cucumber?
          ├── Да → Allure + allure-cucumber (лучшая комбинация)
          │         или Masterthought Cucumber Reports (проще)
          └── Нет → Стандартный Java-проект
                    ├── Allure Report (рекомендация по умолчанию)
                    └── ExtentReports (если уже используется в проекте)
```

### Рекомендации по размеру команды

| Размер команды | Рекомендация |
|---|---|
| 1-3 QA | Allure Report — просто, бесплатно, достаточно |
| 3-10 QA | Allure Report + Slack/Telegram уведомления |
| 10-30 QA | ReportPortal или Allure TestOps |
| 30+ QA | Allure TestOps (если нужна TMS) или ReportPortal (если нужен ML-анализ) |

---

## Связь с тестированием

Выбор инструмента отчётности влияет на весь QA-процесс:

- **Скорость обратной связи** — real-time отчёты vs статические файлы
- **Эффективность триажа** — ML-анализ ошибок vs ручной разбор
- **Коммуникация** — уведомления в мессенджерах сокращают время реакции
- **Метрики качества** — дашборды и тренды для принятия решений
- **Удобство бизнеса** — BDD-отчёты читаемы для стейкхолдеров
- **Масштабируемость** — с ростом команды меняются требования к инструменту

---

## Типичные ошибки

1. **Over-engineering** — внедрение ReportPortal для команды из 2 человек и 50 тестов
2. **Использование нескольких инструментов одновременно** — Allure + ExtentReports одновременно создаёт путаницу
3. **Игнорирование уведомлений** — красивый отчёт, который никто не смотрит
4. **Нет уведомлений при падениях** — команда узнаёт о проблемах с опозданием
5. **Кастомный отчёт вместо готового инструмента** — изобретение велосипеда
6. **Не адаптируют инструмент под команду** — внедряют «потому что модно», а не «потому что нужно»
7. **Отсутствие стратегии хранения** — отчёты накапливаются, занимая гигабайты дискового пространства
8. **Отчёт не интегрирован с CI/CD** — генерируется вручную, теряется автоматизация
9. **Нет информации об окружении** — непонятно, на каком окружении и билде запускались тесты
10. **Секреты в webhook URL** — токены Slack/Telegram хранятся в коде, а не в CI/CD secrets

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Какие инструменты репортинга для тестов вы знаете? Перечислите минимум три.
2. Чем Allure Report отличается от ExtentReports?
3. Зачем нужны уведомления о результатах тестов в Slack/Telegram?
4. Что такое Cucumber Reports и когда они используются?
5. Какой инструмент отчётности вы бы рекомендовали для небольшой команды и почему?

### 🟡 Средний уровень
6. Сравните Allure Report и ReportPortal. В каких случаях какой инструмент предпочтительнее?
7. Как реализовать отправку уведомлений о результатах тестов в Slack через Webhook?
8. Как интегрировать Cucumber-отчёты с Allure?
9. Какие метрики должен содержать хороший отчёт о тестировании?
10. Как организовать хранение и ротацию отчётов в CI/CD?

### 🔴 Продвинутый уровень
11. Опишите стратегию выбора инструмента отчётности для нового проекта. Какие факторы учитываете?
12. Как реализовать агрегацию отчётов из нескольких источников (UI-тесты, API-тесты, нагрузочные тесты)?
13. Как интегрировать отчётность с системой мониторинга (Grafana, Prometheus)?
14. Как обеспечить безопасность отчётов (не допустить утечки тестовых данных, токенов)?
15. Опишите архитектуру системы отчётности для крупного проекта с 5000+ тестами и 10 командами.

---

## Практические задания

### Задание 1: ExtentReports
Настройте ExtentReports в JUnit 5 проекте. Реализуйте Extension для автоматической
генерации отчёта с категориями, скриншотами и системной информацией.

### Задание 2: Уведомления
Создайте Slack- и Telegram-бота для отправки уведомлений о результатах тестирования.
Интегрируйте с CI/CD pipeline. Отправляйте краткую статистику и ссылку на отчёт.

### Задание 3: BDD-отчёты
Создайте Cucumber-проект с интеграцией Allure. Сравните встроенный Cucumber HTML-отчёт,
Masterthought Cucumber Reports и Allure Report. Определите, какой формат удобнее.

### Задание 4: Кастомный отчёт
Напишите генератор кастомного HTML-отчёта, который агрегирует результаты из разных
источников: JUnit XML, REST API мониторинга и CSV с результатами ручного тестирования.

### Задание 5: Сравнительный анализ
Настройте три инструмента (Allure, ExtentReports, ReportPortal) для одного проекта.
Запустите одни и те же тесты. Составьте сравнительный отчёт с плюсами и минусами
каждого инструмента для вашего конкретного проекта.

---

## Дополнительные ресурсы

- [ExtentReports Documentation](https://www.extentreports.com/docs/versions/5/java/)
- [ExtentReports GitHub](https://github.com/extent-framework/extentreports-java)
- [Masterthought Cucumber Reports](https://github.com/damianszczepanik/cucumber-reporting)
- [Allure Cucumber Integration](https://docs.qameta.io/allure/#_cucumber_jvm)
- [pytest-html Documentation](https://pytest-html.readthedocs.io/)
- [Slack Incoming Webhooks](https://api.slack.com/messaging/webhooks)
- [Telegram Bot API](https://core.telegram.org/bots/api)
- [GitHub Actions Slack Notification](https://github.com/marketplace/actions/slack-notify)
