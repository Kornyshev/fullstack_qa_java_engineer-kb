# ReportPortal

## Обзор

ReportPortal — это open-source платформа для агрегации, анализа и визуализации результатов
тестирования в реальном времени. Разработана компанией EPAM Systems и отличается от других
инструментов отчётности применением машинного обучения (ML) для автоматической классификации
ошибок и выявления нестабильных тестов.

Ключевые особенности ReportPortal:
- **Real-time reporting** — результаты появляются по мере выполнения тестов, не дожидаясь завершения
- **ML-анализ ошибок** — система автоматически классифицирует причины падений
- **Централизованное хранилище** — все результаты хранятся в БД с полной историей
- **Кроссфреймворковость** — поддержка JUnit 5, TestNG, Cucumber, pytest, Robot Framework и др.
- **Дашборды и виджеты** — настраиваемые панели мониторинга
- **Flaky test detection** — автоматическое выявление нестабильных тестов

ReportPortal — серверное приложение, разворачиваемое через Docker Compose или Kubernetes.
В отличие от Allure Report (статические HTML-файлы), ReportPortal работает как полноценный веб-сервис.

---

## Архитектура ReportPortal

```
┌──────────────────────────────────────────────────────────────┐
│                       ReportPortal                            │
│                                                               │
│  ┌──────────┐  ┌───────────┐  ┌───────────┐  ┌──────────┐  │
│  │    UI    │  │    API    │  │  Analyzer │  │ Migrations│  │
│  │ (React) │  │ (Java/   │  │   (ML,    │  │           │  │
│  │         │  │  Spring)  │  │  Python)  │  │           │  │
│  └────┬────┘  └─────┬─────┘  └─────┬─────┘  └──────────┘  │
│       │             │              │                         │
│  ┌────┴─────────────┴──────────────┴────────────────────┐   │
│  │                   RabbitMQ                            │   │
│  └──────────────────────┬───────────────────────────────┘   │
│                         │                                    │
│  ┌──────────┐  ┌───────┴──────┐  ┌──────────────────────┐  │
│  │PostgreSQL│  │Elasticsearch │  │   MinIO (S3)         │  │
│  │(данные)  │  │  (поиск,    │  │   (вложения:         │  │
│  │          │  │   анализ)   │  │    скриншоты, логи)  │  │
│  └──────────┘  └──────────────┘  └──────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
         ↑                ↑                  ↑
   ┌─────┴────┐    ┌─────┴────┐      ┌─────┴─────┐
   │  JUnit 5 │    │  TestNG  │      │   pytest   │
   │  Agent   │    │  Agent   │      │   Agent    │
   └──────────┘    └──────────┘      └───────────┘
```

### Основные компоненты

| Компонент | Технология | Назначение |
|-----------|-----------|------------|
| **API Service** | Java / Spring Boot | REST API, основная бизнес-логика |
| **UI Service** | React | Веб-интерфейс пользователя |
| **Analyzer** | Python / ML | ML-анализ ошибок, паттерн-матчинг |
| **PostgreSQL** | PostgreSQL 14+ | Хранение данных о запусках, тестах, пользователях |
| **Elasticsearch** | Elasticsearch 8 | Полнотекстовый поиск, ML-анализ логов |
| **RabbitMQ** | RabbitMQ | Очередь сообщений между сервисами |
| **MinIO** | S3-совместимое | Хранение бинарных вложений (скриншоты, видео) |

---

## ML-анализ ошибок (Auto-Analysis)

### Как это работает

Ключевая фишка ReportPortal — автоматический анализ причин падения тестов
с помощью машинного обучения:

```
1. Тест падает → результат попадает в ReportPortal
2. Analyzer сравнивает stack trace и сообщение ошибки с историческими данными
3. На основе ML-модели (Elasticsearch + TF-IDF) определяется вероятная причина
4. Тест автоматически получает классификацию дефекта
```

### Типы дефектов (Defect Types)

ReportPortal предоставляет систему классификации дефектов:

| Категория | Описание | Цвет |
|-----------|----------|------|
| **Product Bug (PB)** | Дефект продукта | Красный |
| **Automation Bug (AB)** | Ошибка в тестовом коде | Жёлтый |
| **System Issue (SI)** | Проблема инфраструктуры (сеть, сервер, окружение) | Синий |
| **To Investigate (TI)** | Требует ручного анализа | Серый |
| **No Defect (ND)** | Не является дефектом | Зелёный |

### Процесс анализа

1. **Ручной анализ (начальный этап)** — QA вручную классифицирует первые падения,
   обучая систему
2. **Auto-Analysis** — система начинает автоматически классифицировать похожие ошибки
3. **Pattern Analysis** — выявление паттернов (например, «все тесты с `TimeoutException`
   на `staging` — это System Issue»)
4. **Unique Error Analysis** — выявление новых, ранее не встречавшихся ошибок

### Настройка Auto-Analysis

```
Проект → Settings → Analysis
├── Auto Analysis:         ON
├── Analysis Mode:         ALL (или LAUNCH_NAME, CURRENT_LAUNCH)
├── Minimum Should Match:  80% (порог совпадения)
├── Number of Logs:        5 (количество строк лога для анализа)
└── Unique Error:          ON (выявление уникальных ошибок)
```

---

## Дашборды и виджеты

### Типы виджетов

ReportPortal предоставляет богатый набор виджетов для дашбордов:

**Статистические виджеты:**
- **Launch Statistics Chart** — статистика запуска (bar chart / pie chart)
- **Overall Statistics** — общая статистика по всем запускам
- **Investigated Percentage** — процент проанализированных ошибок

**Трендовые виджеты:**
- **Launch Execution and Issue Statistic** — тренд результатов по запускам
- **Failed Cases Trend** — тренд упавших тестов
- **Test Cases Growth Trend** — рост количества тестов

**Аналитические виджеты:**
- **Most Failed Test Cases** — наиболее часто падающие тесты
- **Flaky Test Cases** — нестабильные тесты
- **Longest Launch Durations** — самые долгие запуски
- **Unique Bugs Table** — таблица уникальных ошибок
- **Component Health** — здоровье компонентов системы

### Пример конфигурации дашборда для QA Lead

```
QA Dashboard
├── Pass Rate Trend (линейный график за 30 дней)
├── Defect Distribution (круговая диаграмма: PB / AB / SI)
├── Top 10 Flaky Tests (таблица)
├── Top 10 Most Failed Tests (таблица)
├── Investigated % (статистика: сколько ошибок проанализировано)
├── Average Launch Duration Trend (линейный график)
└── Unique Errors (таблица новых уникальных ошибок)
```

---

## Real-time Reporting

В отличие от Allure Report, где отчёт генерируется после завершения всех тестов,
ReportPortal отображает результаты в реальном времени:

```
Тест запускается → Шаг 1 выполняется → результат сразу виден в UI
                → Шаг 2 выполняется → результат сразу виден в UI
                → Тест падает → скриншот и лог сразу доступны
```

Преимущества real-time отчётности:
- Не нужно ждать завершения всех тестов для анализа первых падений
- Можно отслеживать прогресс длительных тестовых запусков
- Раннее обнаружение массовых падений (например, окружение упало)

---

## Интеграция с тестовыми фреймворками

### JUnit 5

```xml
<!-- Зависимость для ReportPortal JUnit 5 agent -->
<dependency>
    <groupId>com.epam.reportportal</groupId>
    <artifactId>agent-java-junit5</artifactId>
    <version>5.3.2</version>
    <scope>test</scope>
</dependency>

<!-- ReportPortal клиент (logback-интеграция) -->
<dependency>
    <groupId>com.epam.reportportal</groupId>
    <artifactId>logger-java-logback</artifactId>
    <version>5.2.2</version>
    <scope>test</scope>
</dependency>
```

Файл конфигурации `src/test/resources/reportportal.properties`:

```properties
# URL сервера ReportPortal
rp.endpoint=http://reportportal.company.com:8080
# UUID пользователя (для аутентификации)
rp.uuid=your-api-token-here
# Название запуска
rp.launch=Regression Tests
# Название проекта
rp.project=my_project
# Включить/выключить отправку
rp.enable=true
# Режим запуска: DEFAULT или DEBUG
rp.mode=DEFAULT
# Описание запуска
rp.description=Regression test run on staging
# Атрибуты (теги)
rp.attributes=env:staging;browser:chrome;build:1234
```

Регистрация agent через ServiceLoader —
файл `META-INF/services/org.junit.jupiter.api.extension.Extension`:

```
com.epam.reportportal.junit5.ReportPortalExtension
```

### TestNG

```xml
<dependency>
    <groupId>com.epam.reportportal</groupId>
    <artifactId>agent-java-testng</artifactId>
    <version>5.4.2</version>
    <scope>test</scope>
</dependency>
```

Подключение listener в `testng.xml`:

```xml
<suite name="Suite">
    <listeners>
        <listener class-name="com.epam.reportportal.testng.ReportPortalTestNGListener"/>
    </listeners>
    <test name="Regression">
        <classes>
            <class name="com.example.tests.LoginTests"/>
        </classes>
    </test>
</suite>
```

### Прикрепление логов и скриншотов

```java
import com.epam.reportportal.service.ReportPortal;
import java.io.File;
import java.util.Date;

public class ReportPortalUtils {

    // Прикрепление скриншота к текущему шагу
    public static void attachScreenshot(WebDriver driver, String name) {
        File screenshot = ((TakesScreenshot) driver)
                .getScreenshotAs(OutputType.FILE);
        ReportPortal.emitLog(
                name,
                "INFO",
                new Date(),
                screenshot
        );
    }

    // Прикрепление текстового лога
    public static void log(String message) {
        ReportPortal.emitLog(message, "INFO", new Date());
    }

    // Прикрепление ошибки
    public static void logError(String message) {
        ReportPortal.emitLog(message, "ERROR", new Date());
    }
}
```

---

## Интеграция с CI/CD

### Jenkins Pipeline

```groovy
pipeline {
    agent any
    stages {
        stage('Test') {
            steps {
                sh 'mvn clean test -Drp.enable=true -Drp.launch="Jenkins Build #${BUILD_NUMBER}"'
            }
        }
    }
    post {
        always {
            // ReportPortal получает результаты в real-time через agent
            // Дополнительных шагов не требуется
            echo "Results available in ReportPortal"
        }
    }
}
```

### GitHub Actions

```yaml
name: Tests with ReportPortal
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      - name: Run tests
        env:
          RP_ENDPOINT: ${{ secrets.RP_ENDPOINT }}
          RP_UUID: ${{ secrets.RP_UUID }}
        run: |
          mvn clean test \
            -Drp.endpoint=$RP_ENDPOINT \
            -Drp.uuid=$RP_UUID \
            -Drp.launch="GitHub Actions #${{ github.run_number }}" \
            -Drp.project=my_project
```

---

## Flaky Test Detection

ReportPortal автоматически выявляет нестабильные тесты на основе истории:

```
Тест X: PASS → PASS → FAIL → PASS → FAIL → PASS
         → Flaky Rate: 33% → отмечается как Flaky
```

Критерии определения flaky-тестов:
- Тест, который при одинаковых условиях то проходит, то падает
- Настраиваемый порог flaky rate
- История за последние N запусков
- Автоматическая пометка в дашборде

---

## Сравнение ReportPortal с Allure

| Характеристика | ReportPortal | Allure Report | Allure TestOps |
|----------------|-------------|---------------|----------------|
| **Лицензия** | Open Source (Apache 2.0) | Open Source (Apache 2.0) | Коммерческая |
| **Тип** | Серверная платформа | Статические HTML-файлы | Серверная TMS |
| **Real-time** | Да | Нет | Да |
| **ML-анализ** | Да (Auto-Analysis) | Нет | Базовый |
| **Дашборды** | Да (настраиваемые) | Нет (только виджеты в отчёте) | Да |
| **TMS** | Нет (только отчётность) | Нет | Да |
| **Классификация дефектов** | PB/AB/SI/TI/ND | failed/broken | Категории |
| **Flaky Detection** | ML-based | Базовый (retries) | Да |
| **Развёртывание** | Docker / K8s (тяжёлое) | Файлы (лёгкое) | Docker / K8s |
| **Ресурсы** | 8+ GB RAM, PostgreSQL, ES | Минимальные | 16+ GB RAM |
| **Кривая обучения** | Средняя | Низкая | Средняя |
| **Интеграция с Jira** | Плагин | Только ссылки | Двусторонняя |
| **Ручные тест-кейсы** | Нет | Нет | Да |

---

## Развёртывание ReportPortal

### Docker Compose

```bash
# Скачиваем docker-compose файл
curl -LO https://raw.githubusercontent.com/reportportal/reportportal/master/docker-compose.yml

# Запускаем
docker compose up -d

# ReportPortal доступен на http://localhost:8080
# Логин по умолчанию: superadmin / erebus
```

### Минимальные требования к ресурсам

| Компонент | CPU | RAM | Диск |
|-----------|-----|-----|------|
| API Service | 2 core | 3 GB | — |
| Analyzer | 2 core | 2 GB | — |
| UI Service | 0.5 core | 256 MB | — |
| PostgreSQL | 2 core | 4 GB | 50 GB+ |
| Elasticsearch | 2 core | 4 GB | 50 GB+ |
| RabbitMQ | 1 core | 512 MB | 1 GB |
| MinIO | 1 core | 512 MB | 50 GB+ |
| **Итого** | **~10 core** | **~14 GB** | **~150 GB** |

---

## Когда выбирать ReportPortal

### ReportPortal подходит, если:
- Большой объём тестов (тысячи тестов, ежедневные запуски)
- Нужен автоматический анализ ошибок (ML)
- Важен real-time мониторинг тестовых запусков
- Команда готова поддерживать серверную инфраструктуру
- Нужны продвинутые дашборды для менеджмента
- Есть проблема с flaky-тестами и нужен системный подход

### ReportPortal НЕ подходит, если:
- Маленькая команда и небольшой объём тестов
- Нет ресурсов на поддержку серверной инфраструктуры
- Достаточно статических HTML-отчётов (Allure Report)
- Нужна TMS (ReportPortal — не TMS, для тест-кейсов нужен отдельный инструмент)

---

## Связь с тестированием

ReportPortal решает ключевые проблемы QA-процесса в средних и крупных проектах:

- **Автоматизация триажа** — ML-анализ сокращает время ручного разбора падений на 50-80%
- **Прозрачность** — real-time дашборды дают актуальную картину качества
- **Выявление паттернов** — анализ трендов помогает находить системные проблемы
- **Борьба с flaky-тестами** — автоматическое выявление и отслеживание нестабильных тестов
- **Стандартизация** — единая платформа для всех команд и фреймворков
- **Экономия времени** — auto-analysis заменяет рутинный ручной разбор ошибок

---

## Типичные ошибки

1. **Недооценка ресурсов** — ReportPortal требует значительных серверных ресурсов (PostgreSQL + Elasticsearch + MinIO)
2. **Игнорирование обучения ML** — Auto-Analysis нужно «обучить», вручную классифицируя первые сотни ошибок
3. **Не настроены дашборды** — данные есть, но никто не создал осмысленные дашборды
4. **Слишком много логов** — отправка всех логов в ReportPortal перегружает Elasticsearch
5. **Не настроена ротация данных** — база данных растёт бесконтрольно, дисковое пространство заканчивается
6. **Не обновляют ReportPortal** — старые версии содержат баги и уязвимости
7. **Не используют атрибуты (tags)** — теряется возможность фильтрации запусков по окружению, браузеру, билду
8. **Один проект на всё** — разные команды пишут в один проект, данные перемешиваются
9. **Не настроена интеграция с CI/CD** — результаты загружаются вручную
10. **Путают с TMS** — ReportPortal не заменяет TestRail или Allure TestOps для управления тест-кейсами

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Что такое ReportPortal? Чем он отличается от Allure Report?
2. Что означает «real-time reporting»? Почему это важно?
3. Какие типы дефектов существуют в ReportPortal (PB, AB, SI, TI, ND)?
4. Как подключить ReportPortal к JUnit 5 проекту?

### 🟡 Средний уровень
5. Как работает Auto-Analysis в ReportPortal? Какие технологии используются?
6. Что такое flaky test detection и как ReportPortal определяет нестабильные тесты?
7. Какие виджеты можно настроить на дашборде ReportPortal? Какие наиболее полезны?
8. Сравните ReportPortal и Allure Report. Когда какой инструмент выбрать?
9. Какие компоненты входят в архитектуру ReportPortal?

### 🔴 Продвинутый уровень
10. Опишите процесс обучения ML-модели Auto-Analysis. Как повысить точность классификации?
11. Как оптимизировать производительность ReportPortal при большом объёме данных?
12. Как настроить ReportPortal в Kubernetes для продуктивного использования?
13. Как реализовать кастомные виджеты и интеграции через ReportPortal API?
14. Какие метрики из ReportPortal полезны для оценки зрелости QA-процесса?

---

## Практические задания

### Задание 1: Развёртывание ReportPortal
Разверните ReportPortal локально через Docker Compose. Создайте проект, настройте
пользователей и права доступа. Изучите интерфейс.

### Задание 2: Интеграция с JUnit 5
Подключите ReportPortal agent к существующему JUnit 5 проекту. Запустите тесты
и убедитесь, что результаты отображаются в реальном времени.

### Задание 3: Auto-Analysis
Запустите тесты несколько раз, вручную классифицируйте ошибки по категориям
(PB, AB, SI). Включите Auto-Analysis и проверьте, что система начинает
автоматически классифицировать повторяющиеся ошибки.

### Задание 4: Дашборд
Создайте дашборд для QA Lead с виджетами: Pass Rate Trend, Flaky Tests,
Most Failed Tests, Defect Distribution. Настройте фильтры по окружению.

### Задание 5: Сравнительный отчёт
Настройте параллельно Allure Report и ReportPortal для одного проекта.
Запустите одни и те же тесты, сравните информативность отчётов.
Подготовьте рекомендацию для команды.

---

## Дополнительные ресурсы

- [ReportPortal Documentation](https://reportportal.io/docs/)
- [ReportPortal GitHub](https://github.com/reportportal)
- [ReportPortal Docker Compose](https://github.com/reportportal/reportportal/blob/master/docker-compose.yml)
- [JUnit 5 Agent](https://github.com/reportportal/agent-java-junit5)
- [TestNG Agent](https://github.com/reportportal/agent-java-testng)
- [ReportPortal REST API](https://reportportal.io/docs/dev-guides/APIDocs)
- [ReportPortal Blog](https://reportportal.io/blog)
