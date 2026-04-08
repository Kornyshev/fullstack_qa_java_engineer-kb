# Allure Framework

## Обзор

Allure Framework — это гибкий и мощный инструмент для генерации наглядных и информативных отчётов
о результатах тестирования. Разработан компанией Qameta Software (ранее — командой Yandex), Allure
стал де-факто стандартом отчётности в Java-экосистеме автоматизации тестирования.

Основные преимущества Allure:
- **Наглядность** — красивые интерактивные HTML-отчёты с графиками, таймлайнами и группировками
- **Универсальность** — поддержка JUnit 5, TestNG, Cucumber, pytest, Mocha и других фреймворков
- **Расширяемость** — система аннотаций и API для кастомизации отчётов
- **Трассируемость** — связь тестов с требованиями, задачами в Jira, тест-кейсами в TMS
- **Шаговая детализация** — разбиение тестов на логические шаги для удобной диагностики

Allure работает по принципу: тестовый фреймворк генерирует результаты в формате JSON/XML,
а Allure CLI собирает их в единый HTML-отчёт.

---

## Архитектура Allure

### Принцип работы

```
┌─────────────────────────────────────────────────────────┐
│                    Тестовый прогон                       │
│                                                         │
│  JUnit 5 / TestNG / Cucumber                            │
│      ↓                                                  │
│  Allure Listener (allure-junit5 / allure-testng)        │
│      ↓                                                  │
│  allure-results/ (JSON + вложения)                      │
│      ↓                                                  │
│  Allure CLI (allure generate / allure serve)             │
│      ↓                                                  │
│  allure-report/ (статический HTML-отчёт)                │
└─────────────────────────────────────────────────────────┘
```

### Структура директории allure-results

После запуска тестов в директории `allure-results/` создаются файлы:

```
allure-results/
├── <uuid>-result.json        # результат каждого теста
├── <uuid>-container.json     # метаданные контейнера (класс, suite)
├── <uuid>-attachment.png     # скриншоты и вложения
├── <uuid>-attachment.txt     # текстовые вложения (логи, page source)
├── categories.json           # правила категоризации ошибок
├── environment.properties    # информация об окружении
└── history/                  # данные для трендов (копируются из предыдущего отчёта)
```

---

## Аннотации Allure

### @Step — шаги теста

Аннотация `@Step` — самая важная в Allure. Она разбивает тест на логические шаги,
делая отчёт читаемым и пригодным для диагностики.

```java
import io.qameta.allure.Step;

public class LoginPage {

    // Шаг с параметрами — значения подставляются в отчёт
    @Step("Ввести логин: {username}")
    public void enterUsername(String username) {
        driver.findElement(By.id("username")).sendKeys(username);
    }

    @Step("Ввести пароль")
    public void enterPassword(String password) {
        // Пароль не выводим в отчёт из соображений безопасности
        driver.findElement(By.id("password")).sendKeys(password);
    }

    @Step("Нажать кнопку 'Войти'")
    public void clickLoginButton() {
        driver.findElement(By.id("login-btn")).click();
    }

    // Составной шаг — вложенные шаги отобразятся иерархически
    @Step("Авторизоваться как {username}")
    public void login(String username, String password) {
        enterUsername(username);
        enterPassword(password);
        clickLoginButton();
    }
}
```

### @Attachment — вложения

```java
import io.qameta.allure.Attachment;

public class AllureAttachments {

    // Скриншот как вложение
    @Attachment(value = "Скриншот страницы", type = "image/png")
    public byte[] takeScreenshot(WebDriver driver) {
        return ((TakesScreenshot) driver).getScreenshotAs(OutputType.BYTES);
    }

    // Текстовое вложение — page source
    @Attachment(value = "Page Source", type = "text/html")
    public String attachPageSource(WebDriver driver) {
        return driver.getPageSource();
    }

    // Вложение через Allure.addAttachment() — альтернативный способ
    public void attachLog(String logContent) {
        Allure.addAttachment("Лог теста", "text/plain", logContent);
    }

    // JSON вложение — полезно для API-тестов
    @Attachment(value = "Response Body", type = "application/json")
    public String attachResponseBody(String responseBody) {
        return responseBody;
    }
}
```

### @Description — описание теста

```java
import io.qameta.allure.Description;

public class LoginTests {

    @Test
    @Description("Проверяем, что пользователь с валидными учётными данными "
               + "может успешно авторизоваться и попасть на главную страницу")
    public void testSuccessfulLogin() {
        // ...
    }
}
```

### BDD-аннотации: @Epic, @Feature, @Story

Эти аннотации позволяют организовать тесты по бизнес-функциональности.
Результаты группируются на вкладке **Behaviors** в отчёте.

```java
import io.qameta.allure.Epic;
import io.qameta.allure.Feature;
import io.qameta.allure.Story;

@Epic("Авторизация и аутентификация")
@Feature("Вход в систему")
public class LoginTests {

    @Test
    @Story("Успешная авторизация")
    public void testLoginWithValidCredentials() {
        // ...
    }

    @Test
    @Story("Неуспешная авторизация")
    public void testLoginWithInvalidPassword() {
        // ...
    }

    @Test
    @Story("Неуспешная авторизация")
    public void testLoginWithEmptyFields() {
        // ...
    }
}
```

Иерархия в отчёте:
```
Epic: Авторизация и аутентификация
  └── Feature: Вход в систему
        ├── Story: Успешная авторизация
        │     └── testLoginWithValidCredentials
        └── Story: Неуспешная авторизация
              ├── testLoginWithInvalidPassword
              └── testLoginWithEmptyFields
```

### @Severity — приоритет теста

```java
import io.qameta.allure.Severity;
import io.qameta.allure.SeverityLevel;

public class PaymentTests {

    @Test
    @Severity(SeverityLevel.BLOCKER)    // Блокирующий — основная функциональность
    public void testPaymentProcessing() { }

    @Test
    @Severity(SeverityLevel.CRITICAL)   // Критический — важная функциональность
    public void testPaymentValidation() { }

    @Test
    @Severity(SeverityLevel.NORMAL)     // Обычный — стандартная функциональность
    public void testPaymentHistory() { }

    @Test
    @Severity(SeverityLevel.MINOR)      // Незначительный — косметика, UI
    public void testPaymentPageLayout() { }

    @Test
    @Severity(SeverityLevel.TRIVIAL)    // Тривиальный — мелочи
    public void testPaymentTooltips() { }
}
```

### @Owner — ответственный за тест

```java
import io.qameta.allure.Owner;

@Owner("Иван Петров")
public class CartTests {

    @Test
    @Owner("Анна Сидорова") // Можно переопределить на уровне метода
    public void testAddToCart() { }
}
```

### Аннотации-ссылки: @Link, @Issue, @TmsLink

```java
import io.qameta.allure.Link;
import io.qameta.allure.Issue;
import io.qameta.allure.TmsLink;

public class UserProfileTests {

    // Произвольная ссылка
    @Test
    @Link(name = "Документация", url = "https://wiki.company.com/user-profile")
    public void testViewProfile() { }

    // Ссылка на баг/задачу в трекере
    @Test
    @Issue("PROJ-1234")
    public void testProfileEditBug() { }

    // Ссылка на тест-кейс в TMS
    @Test
    @TmsLink("TC-5678")
    public void testUpdateAvatar() { }
}
```

Для формирования полных URL из идентификаторов нужно настроить паттерны
в файле `allure.properties`:

```properties
# Базовый URL для @Issue
allure.link.issue.pattern=https://jira.company.com/browse/{}
# Базовый URL для @TmsLink
allure.link.tms.pattern=https://testops.company.com/testcase/{}
# Кастомный тип ссылки
allure.link.docs.pattern=https://wiki.company.com/{}
```

---

## Структура Allure Report

### Overview (Обзор)

Главная страница отчёта содержит:
- **Статистика** — количество passed/failed/broken/skipped/unknown тестов
- **Круговая диаграмма** — визуальное соотношение статусов
- **Trend** — график изменения результатов во времени (требует `history/`)
- **Suites** — виджет с разбивкой по test suites
- **Categories** — виджет с категориями ошибок
- **Environment** — информация об окружении из `environment.properties`

### Categories (Категории)

Ошибки группируются по типам. По умолчанию Allure различает:
- **Product defects** — `AssertionError` (упавшие assertions)
- **Test defects** — все прочие exceptions (ошибки в тестовом коде)

Категории настраиваются через файл `categories.json`.

### Suites (Наборы тестов)

Группировка тестов по пакетам и классам. Показывает иерархию:
`Package → Class → Test Method`

### Graphs (Графики)

- **Status** — круговая диаграмма по статусам
- **Severity** — распределение по severity
- **Duration** — распределение по длительности
- **Duration Trend** — тренд по длительности
- **Retry Trend** — тренд повторных запусков
- **Categories Trend** — тренд по категориям

### Timeline (Таймлайн)

Показывает порядок выполнения тестов во времени. Полезно для:
- Выявления тестов-блокеров (долгие тесты блокируют поток)
- Анализа параллельного выполнения
- Обнаружения тестов с аномально долгим выполнением

### Behaviors (Поведения)

Группировка по BDD-аннотациям: `Epic → Feature → Story`.
Очень удобно для демонстрации покрытия функциональности.

### Packages (Пакеты)

Группировка по Java-пакетам. Отражает техническую структуру проекта.

---

## categories.json — категоризация ошибок

Файл `categories.json` размещается в `src/test/resources/` (или в `allure-results/`).

```json
[
  {
    "name": "Игнорируемые тесты",
    "matchedStatuses": ["skipped"]
  },
  {
    "name": "Дефекты продукта",
    "matchedStatuses": ["failed"],
    "messageRegex": ".*expected.*but was.*"
  },
  {
    "name": "Дефекты тестов",
    "matchedStatuses": ["broken"],
    "messageRegex": ".*"
  },
  {
    "name": "Проблемы с окружением",
    "matchedStatuses": ["broken"],
    "messageRegex": ".*Connection refused.*|.*timeout.*|.*UnreachableBrowserException.*"
  },
  {
    "name": "Известные баги",
    "matchedStatuses": ["failed"],
    "messageRegex": ".*KNOWN_BUG.*",
    "traceRegex": ".*"
  },
  {
    "name": "Flaky-тесты",
    "matchedStatuses": ["failed", "broken"],
    "flaky": true
  }
]
```

Каждая категория содержит:
- `name` — название категории в отчёте
- `matchedStatuses` — статусы тестов (`failed`, `broken`, `skipped`, `passed`, `unknown`)
- `messageRegex` — регулярное выражение для сообщения ошибки
- `traceRegex` — регулярное выражение для stack trace
- `flaky` — отметка нестабильных тестов

---

## environment.properties — информация об окружении

Файл `environment.properties` размещается в `allure-results/` и отображается
на главной странице отчёта:

```properties
Browser=Chrome 120
Browser.Version=120.0.6099.109
OS=Ubuntu 22.04
Java.Version=17.0.9
Environment=staging
Base.URL=https://staging.example.com
Build.Number=build-1234
Test.Framework=JUnit 5.10.1
Allure.Version=2.25.0
```

Программное создание файла (полезно для CI/CD):

```java
import java.io.FileOutputStream;
import java.util.Properties;

public class AllureEnvironmentWriter {

    // Метод для создания environment.properties динамически
    public static void writeEnvironment(Map<String, String> envProperties) {
        Properties props = new Properties();
        envProperties.forEach(props::setProperty);

        try (FileOutputStream fos =
                     new FileOutputStream("allure-results/environment.properties")) {
            props.store(fos, "Allure Environment Properties");
        } catch (Exception e) {
            throw new RuntimeException("Не удалось записать environment.properties", e);
        }
    }
}
```

---

## Тренды (History и Trends)

Для отображения трендов необходимо сохранять историю между запусками:

```bash
# После генерации отчёта скопировать history в allure-results
cp -r allure-report/history allure-results/history

# Теперь следующий allure generate покажет тренды
allure generate allure-results --clean -o allure-report
```

В CI/CD это обычно реализуется так:
1. Перед запуском тестов — скачать `history/` из предыдущего отчёта
2. Запустить тесты
3. Сгенерировать отчёт
4. Сохранить `history/` для следующего запуска

---

## Чтение и интерпретация отчётов

### Статусы тестов в Allure

| Статус | Значение | Типичная причина |
|--------|----------|-----------------|
| **passed** (зелёный) | Тест прошёл | Все assertions выполнены |
| **failed** (красный) | Тест упал | `AssertionError` — дефект продукта или некорректный тест |
| **broken** (жёлтый) | Тест сломан | Exception в коде — проблема в тесте или окружении |
| **skipped** (серый) | Тест пропущен | `@Disabled`, предусловие не выполнено |
| **unknown** (фиолетовый) | Неизвестно | Некорректные результаты |

### Алгоритм анализа отчёта

1. **Смотрим Overview** — общая картина: сколько passed/failed/broken
2. **Проверяем Categories** — какие типы ошибок преобладают
3. **Анализируем Failed тесты** — раскрываем каждый, смотрим шаги и скриншоты
4. **Смотрим Broken тесты** — обычно проблемы инфраструктуры или кода тестов
5. **Проверяем Timeline** — нет ли аномально долгих тестов
6. **Смотрим Trends** — ухудшается ли ситуация по сравнению с прошлыми запусками

---

## Конфигурация Allure

### Файл allure.properties

Размещается в `src/test/resources/allure.properties`:

```properties
# Директория для результатов
allure.results.directory=allure-results

# Паттерны ссылок
allure.link.issue.pattern=https://jira.company.com/browse/{}
allure.link.tms.pattern=https://testops.company.com/testcase/{}

# Включение аспектов для @Step
allure.aspectj.enabled=true
```

### Настройка AspectJ для @Step

Чтобы аннотация `@Step` работала на обычных методах (не только в тестах),
нужен AspectJ Weaver:

```xml
<!-- Maven Surefire Plugin с AspectJ -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.5</version>
    <configuration>
        <argLine>
            -javaagent:"${settings.localRepository}/org/aspectj/aspectjweaver/${aspectj.version}/aspectjweaver-${aspectj.version}.jar"
        </argLine>
    </configuration>
</plugin>
```

---

## Связь с тестированием

Allure Framework напрямую связан с повседневной работой QA-инженера:

- **Диагностика падений** — шаги, скриншоты и вложения позволяют быстро понять причину
- **Коммуникация** — наглядные отчёты понятны менеджерам, разработчикам, заказчикам
- **Трассируемость** — связь тестов с требованиями через `@Epic`, `@Feature`, `@Story`
- **Метрики** — тренды показывают динамику качества продукта
- **CI/CD интеграция** — отчёты автоматически генерируются на каждый билд
- **Категоризация ошибок** — `categories.json` помогает отделить баги продукта от проблем тестов

---

## Типичные ошибки

1. **Отсутствие `@Step` аннотаций** — отчёт превращается в набор имён тестов без деталей
2. **Не настроен AspectJ** — `@Step` на методах Page Object не срабатывает
3. **Забыли скопировать `history/`** — тренды не отображаются
4. **`categories.json` не скопирован в `allure-results/`** — категоризация не работает
5. **Слишком длинные имена шагов** — отчёт становится нечитаемым
6. **Пароли и секреты в параметрах `@Step`** — конфиденциальные данные попадают в отчёт
7. **Не создан `environment.properties`** — нет контекста о том, где запускались тесты
8. **`allure-results/` не очищается перед запуском** — старые результаты смешиваются с новыми
9. **Неправильные regex в `categories.json`** — ошибки не попадают в нужные категории
10. **Нет вложений при падении** — скриншот не делается, диагностика затруднена

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Что такое Allure Framework и зачем он нужен?
2. Какие статусы тестов существуют в Allure? Чем `failed` отличается от `broken`?
3. Для чего нужна аннотация `@Step`?
4. Как добавить скриншот в Allure-отчёт?
5. Что содержит директория `allure-results/`?
6. Чем отличается `allure serve` от `allure generate`?

### 🟡 Средний уровень
7. Как настроить `categories.json` для категоризации ошибок? Приведите пример.
8. Для чего нужны аннотации `@Epic`, `@Feature`, `@Story`? Как они связаны с BDD?
9. Как работает аннотация `@Attachment`? Какие типы вложений поддерживаются?
10. Как настроить ссылки на Jira через `@Issue` и `@TmsLink`?
11. Зачем нужен `environment.properties` и как его формировать автоматически?
12. Как организовать отображение трендов между запусками?

### 🔴 Продвинутый уровень
13. Как работает AspectJ Weaver в связке с Allure? Зачем он нужен?
14. Как реализовать кастомные аннотации для Allure с использованием `@LabelAnnotation`?
15. Опишите архитектуру Allure: от listener до HTML-отчёта.
16. Как интегрировать Allure с CI/CD pipeline с сохранением истории?
17. Как реализовать динамические шаги через `Allure.step()` API?
18. Как настроить Allure для параллельного запуска тестов?

---

## Практические задания

### Задание 1: Базовая настройка Allure
Создайте проект с JUnit 5 и Allure. Напишите 5 тестов с аннотациями `@Step`,
`@Description`, `@Severity`. Сгенерируйте отчёт и изучите каждую вкладку.

### Задание 2: BDD-структура отчёта
Создайте набор тестов для интернет-магазина. Используйте `@Epic`, `@Feature`, `@Story`
для организации тестов по функциональности. Убедитесь, что вкладка Behaviors
отображает логичную иерархию.

### Задание 3: Категоризация ошибок
Напишите `categories.json` с категориями: «Дефекты продукта», «Проблемы окружения»,
«Ошибки тестов», «Известные баги». Создайте тесты, падающие с разными типами ошибок,
и убедитесь, что категоризация работает корректно.

### Задание 4: Вложения
Реализуйте автоматическое прикрепление скриншотов, page source и логов при падении теста.
Используйте `TestWatcher` extension для JUnit 5.

### Задание 5: Тренды
Настройте сохранение истории между запусками тестов. Запустите тесты несколько раз
с разными результатами и убедитесь, что тренды отображаются корректно.

---

## Дополнительные ресурсы

- [Allure Framework Documentation](https://docs.qameta.io/allure/)
- [Allure GitHub Repository](https://github.com/allure-framework)
- [Allure Examples](https://github.com/allure-examples)
- [Allure JUnit 5 Integration](https://docs.qameta.io/allure/#_junit_5)
- [Allure Annotations Reference](https://docs.qameta.io/allure/#_features)
- [Allure Command-Line Tool](https://docs.qameta.io/allure/#_installing_a_commandline)
