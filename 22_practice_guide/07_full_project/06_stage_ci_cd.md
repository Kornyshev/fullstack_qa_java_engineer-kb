# Этап 5: CI/CD -- автоматизированный pipeline тестирования

## Обзор

На предыдущих этапах мы создали API-тесты, UI-тесты и настроили Allure-репортинг. Но всё это
запускается локально, вручную. В реальных проектах тесты должны запускаться **автоматически**:
при каждом Pull Request, по расписанию для regression, при деплое на staging. Именно это
обеспечивает CI/CD pipeline.

На этом этапе мы настроим полноценный GitHub Actions pipeline, который:
- Запускает тесты при каждом PR в main
- Выполняет scheduled regression каждую ночь
- Собирает Allure Report как артефакт
- Публикует Allure Report на GitHub Pages
- Уведомляет о результатах

**Deliverable:** полностью автоматизированный test pipeline с Allure-отчётом на GitHub Pages.

---

## Предварительные требования

Перед началом убедитесь, что у вас:
- Репозиторий проекта на GitHub
- Тесты запускаются локально через `mvn clean test`
- Allure-результаты генерируются в `target/allure-results/`
- В `pom.xml` настроен `maven-surefire-plugin` с AspectJ Weaver

### Структура проекта на данный момент

```
conduit-test-framework/
├── .github/
│   └── workflows/
│       ├── test-on-pr.yml           # Запуск тестов при PR
│       ├── scheduled-regression.yml # Ночной regression
│       └── allure-deploy.yml        # Деплой отчёта на GitHub Pages
├── src/
│   └── test/
│       ├── java/
│       │   ├── api/                 # API-тесты
│       │   ├── ui/                  # UI-тесты
│       │   └── base/               # Базовые классы
│       └── resources/
│           └── allure.properties
├── pom.xml
└── README.md
```

---

## Шаг 1: Базовый workflow -- запуск тестов при Pull Request

### 1.1 Создание файла workflow

Создайте файл `.github/workflows/test-on-pr.yml`:

```yaml
# Workflow: запуск тестов при создании или обновлении Pull Request
name: PR Test Suite

# Триггеры: запуск при PR в main и develop
on:
  pull_request:
    branches:
      - main
      - develop
    types:
      - opened
      - synchronize
      - reopened

# Переменные окружения, общие для всех jobs
env:
  JAVA_VERSION: '17'
  MAVEN_OPTS: '-Xmx1024m'

jobs:
  api-tests:
    name: API Tests
    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      # Шаг 1: Клонируем репозиторий
      - name: Checkout repository
        uses: actions/checkout@v4

      # Шаг 2: Устанавливаем JDK
      - name: Set up JDK ${{ env.JAVA_VERSION }}
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'maven'

      # Шаг 3: Запускаем API-тесты
      - name: Run API tests
        run: mvn clean test -Dtest.suite=api -Dallure.results.directory=target/allure-results
        env:
          BASE_URL: ${{ secrets.API_BASE_URL }}
          API_TOKEN: ${{ secrets.API_TOKEN }}

      # Шаг 4: Сохраняем результаты Allure как артефакт
      - name: Upload Allure results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: allure-results-api
          path: target/allure-results/
          retention-days: 30

  ui-tests:
    name: UI Tests
    runs-on: ubuntu-latest
    timeout-minutes: 30

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up JDK ${{ env.JAVA_VERSION }}
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'maven'

      # Шаг: Установка Chrome для UI-тестов
      - name: Setup Chrome
        uses: browser-actions/setup-chrome@v1
        with:
          chrome-version: 'stable'

      # Шаг: Запуск UI-тестов в headless-режиме
      - name: Run UI tests
        run: mvn clean test -Dtest.suite=ui -Dselenide.headless=true -Dallure.results.directory=target/allure-results
        env:
          BASE_URL: ${{ secrets.APP_BASE_URL }}

      - name: Upload Allure results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: allure-results-ui
          path: target/allure-results/
          retention-days: 30

  # Job для генерации объединённого Allure-отчёта
  generate-report:
    name: Generate Allure Report
    runs-on: ubuntu-latest
    needs: [api-tests, ui-tests]
    if: always()

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      # Скачиваем результаты обоих test-suite
      - name: Download API test results
        uses: actions/download-artifact@v4
        with:
          name: allure-results-api
          path: allure-results/

      - name: Download UI test results
        uses: actions/download-artifact@v4
        with:
          name: allure-results-ui
          path: allure-results/

      # Генерируем Allure Report
      - name: Generate Allure Report
        uses: simple-elf/allure-report-action@v1.9
        with:
          allure_results: allure-results
          allure_history: allure-history
          keep_reports: 20

      # Загружаем готовый отчёт как артефакт
      - name: Upload Allure Report
        uses: actions/upload-artifact@v4
        with:
          name: allure-report
          path: allure-report/
          retention-days: 30
```

### 1.2 Разбор ключевых элементов

**`if: always()`** -- критически важная настройка. Без неё шаг загрузки артефактов будет
пропущен при падении тестов. А нам нужен отчёт **именно** когда тесты падают.

**`timeout-minutes`** -- ограничение по времени. Без него зависший тест заблокирует runner
на 6 часов (лимит GitHub по умолчанию).

**`cache: 'maven'`** -- кэширование Maven-зависимостей между запусками. Ускоряет pipeline
с ~5 минут до ~2 минут.

**`needs: [api-tests, ui-tests]`** -- job `generate-report` ждёт завершения обоих test jobs.

---

## Шаг 2: Scheduled Regression -- ночной запуск полного набора тестов

Создайте файл `.github/workflows/scheduled-regression.yml`:

```yaml
# Workflow: полный regression каждую ночь
name: Nightly Regression

on:
  # Запуск по расписанию (cron): каждый день в 02:00 UTC
  schedule:
    - cron: '0 2 * * *'

  # Ручной запуск через UI GitHub (удобно для дебага)
  workflow_dispatch:
    inputs:
      test_suite:
        description: 'Какой набор тестов запустить'
        required: true
        default: 'all'
        type: choice
        options:
          - all
          - api
          - ui
          - smoke

env:
  JAVA_VERSION: '17'

jobs:
  regression:
    name: Full Regression Suite
    runs-on: ubuntu-latest
    timeout-minutes: 60

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up JDK ${{ env.JAVA_VERSION }}
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'maven'

      - name: Setup Chrome
        uses: browser-actions/setup-chrome@v1
        with:
          chrome-version: 'stable'

      # Определяем, какой suite запускать
      - name: Set test suite
        id: suite
        run: |
          SUITE="${{ github.event.inputs.test_suite || 'all' }}"
          echo "suite=$SUITE" >> $GITHUB_OUTPUT

      # Запуск тестов
      - name: Run regression tests
        run: |
          if [ "${{ steps.suite.outputs.suite }}" = "all" ]; then
            mvn clean test -Dselenide.headless=true
          else
            mvn clean test -Dtest.suite=${{ steps.suite.outputs.suite }} -Dselenide.headless=true
          fi
        env:
          BASE_URL: ${{ secrets.API_BASE_URL }}
          APP_BASE_URL: ${{ secrets.APP_BASE_URL }}
          API_TOKEN: ${{ secrets.API_TOKEN }}

      # Загрузка Allure-результатов
      - name: Upload Allure results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: allure-results-regression
          path: target/allure-results/
          retention-days: 60

      # Генерация отчёта
      - name: Generate Allure Report
        if: always()
        uses: simple-elf/allure-report-action@v1.9
        with:
          allure_results: target/allure-results
          allure_history: allure-history
          keep_reports: 30

      # Публикация на GitHub Pages
      - name: Deploy to GitHub Pages
        if: always()
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_branch: gh-pages
          publish_dir: allure-history
```

### 2.1 Формат cron в GitHub Actions

```
┌───────────── минуты (0-59)
│ ┌───────────── часы (0-23)
│ │ ┌───────────── день месяца (1-31)
│ │ │ ┌───────────── месяц (1-12)
│ │ │ │ ┌───────────── день недели (0-6, 0 = воскресенье)
│ │ │ │ │
* * * * *
```

Примеры:
- `0 2 * * *` -- каждый день в 02:00 UTC
- `0 2 * * 1-5` -- каждый будний день в 02:00 UTC
- `0 */6 * * *` -- каждые 6 часов
- `30 8 * * 1` -- каждый понедельник в 08:30 UTC

### 2.2 Параметр `workflow_dispatch`

Позволяет запускать workflow вручную через UI GitHub (кнопка "Run workflow" во вкладке Actions).
Это незаменимо для:
- Дебага самого workflow
- Экстренного regression перед hotfix-релизом
- Запуска конкретного test suite по требованию менеджера

---

## Шаг 3: Публикация Allure Report на GitHub Pages

### 3.1 Настройка GitHub Pages

1. Перейдите в **Settings** -> **Pages** в вашем репозитории
2. В разделе **Source** выберите **Deploy from a branch**
3. Выберите ветку **gh-pages** и папку **/ (root)**
4. Нажмите **Save**

### 3.2 Выделенный workflow для деплоя отчёта

Создайте файл `.github/workflows/allure-deploy.yml`:

```yaml
# Workflow: генерация и деплой Allure Report на GitHub Pages
name: Allure Report Deploy

on:
  # Запускается после завершения workflow PR Test Suite
  workflow_run:
    workflows: ["PR Test Suite", "Nightly Regression"]
    types:
      - completed

# Права для деплоя на GitHub Pages
permissions:
  contents: write
  pages: write

jobs:
  deploy-report:
    name: Deploy Allure Report
    runs-on: ubuntu-latest
    if: github.event.workflow_run.conclusion != 'cancelled'

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      # Скачиваем артефакты из завершившегося workflow
      - name: Download Allure results
        uses: dawidd6/action-download-artifact@v3
        with:
          run_id: ${{ github.event.workflow_run.id }}
          name: allure-results-.*
          name_is_regexp: true
          path: allure-results/

      # Скачиваем историю отчётов из gh-pages (для трендов)
      - name: Get Allure history
        uses: actions/checkout@v4
        with:
          ref: gh-pages
          path: gh-pages-history
        continue-on-error: true

      - name: Copy history
        run: |
          if [ -d gh-pages-history/history ]; then
            cp -r gh-pages-history/history allure-results/history
          fi

      # Генерация Allure Report
      - name: Generate Allure Report
        uses: simple-elf/allure-report-action@v1.9
        with:
          allure_results: allure-results
          allure_history: allure-history
          keep_reports: 30

      # Деплой на GitHub Pages
      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_branch: gh-pages
          publish_dir: allure-history
```

### 3.3 Сохранение истории отчётов

Ключевой момент -- сохранение **истории** между запусками. Это позволяет Allure показывать:
- **Trend** -- график прохождения тестов по времени
- **Duration** -- изменение времени выполнения
- **Retries** -- статистику перезапусков

Для этого мы каждый раз:
1. Скачиваем текущий контент ветки `gh-pages`
2. Копируем папку `history/` в `allure-results/`
3. Генерируем новый отчёт (Allure подхватит историю)
4. Публикуем обратно на `gh-pages`

---

## Шаг 4: Настройка Secrets

### 4.1 Как добавить секреты в GitHub

1. Перейдите в **Settings** -> **Secrets and variables** -> **Actions**
2. Нажмите **New repository secret**
3. Задайте имя и значение

### 4.2 Список необходимых секретов

| Secret | Описание | Пример значения |
|--------|----------|-----------------|
| `API_BASE_URL` | Базовый URL API приложения | `https://api.realworld.io/api` |
| `APP_BASE_URL` | URL фронтенда для UI-тестов | `https://demo.realworld.io` |
| `API_TOKEN` | Токен авторизации для API | `eyJhbGciOi...` |
| `GITHUB_TOKEN` | Автоматически доступен | Не нужно создавать |

### 4.3 Использование секретов в коде тестов

Секреты передаются в тесты через переменные окружения. В Java-коде:

```java
public class TestConfig {
    // Читаем URL из переменной окружения, если она задана;
    // иначе используем значение по умолчанию для локального запуска
    public static final String BASE_URL = System.getenv("BASE_URL") != null
            ? System.getenv("BASE_URL")
            : "http://localhost:8080/api";

    public static final String API_TOKEN = System.getenv("API_TOKEN") != null
            ? System.getenv("API_TOKEN")
            : "local-dev-token";

    public static final String APP_BASE_URL = System.getenv("APP_BASE_URL") != null
            ? System.getenv("APP_BASE_URL")
            : "http://localhost:3000";
}
```

### 4.4 Environments (опционально)

GitHub Environments позволяют группировать секреты по окружениям (staging, production)
и добавлять approval rules:

```yaml
jobs:
  staging-tests:
    runs-on: ubuntu-latest
    environment: staging    # Используются секреты из environment "staging"
    steps:
      - name: Run tests
        run: mvn clean test
        env:
          BASE_URL: ${{ secrets.BASE_URL }}  # Берётся из environment "staging"
```

---

## Шаг 5: Полный workflow -- все этапы в одном файле

Для удобства приведём единый, самодостаточный workflow, объединяющий все этапы.

Файл `.github/workflows/complete-pipeline.yml`:

```yaml
name: Complete Test Pipeline

on:
  pull_request:
    branches: [main, develop]
  schedule:
    - cron: '0 2 * * 1-5'
  workflow_dispatch:
    inputs:
      test_suite:
        description: 'Test suite to run'
        required: false
        default: 'all'
        type: choice
        options: [all, api, ui, smoke]

permissions:
  contents: write
  pages: write
  checks: write
  pull-requests: write

env:
  JAVA_VERSION: '17'
  ALLURE_RESULTS: target/allure-results

jobs:
  # ============================================================
  # Job 1: API-тесты
  # ============================================================
  api-tests:
    name: API Tests
    runs-on: ubuntu-latest
    timeout-minutes: 15
    # Пропускаем API-тесты, если явно запрошены только UI
    if: >
      github.event.inputs.test_suite == 'all' ||
      github.event.inputs.test_suite == 'api' ||
      github.event.inputs.test_suite == 'smoke' ||
      github.event.inputs.test_suite == ''

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'maven'

      - name: Run API tests
        run: mvn clean test -Dgroups=api -Dallure.results.directory=${{ env.ALLURE_RESULTS }}
        env:
          BASE_URL: ${{ secrets.API_BASE_URL }}
          API_TOKEN: ${{ secrets.API_TOKEN }}

      - name: Upload results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: allure-results-api
          path: ${{ env.ALLURE_RESULTS }}/
          retention-days: 30

  # ============================================================
  # Job 2: UI-тесты
  # ============================================================
  ui-tests:
    name: UI Tests
    runs-on: ubuntu-latest
    timeout-minutes: 30
    if: >
      github.event.inputs.test_suite == 'all' ||
      github.event.inputs.test_suite == 'ui' ||
      github.event.inputs.test_suite == 'smoke' ||
      github.event.inputs.test_suite == ''

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'maven'

      - name: Setup Chrome
        uses: browser-actions/setup-chrome@v1
        with:
          chrome-version: 'stable'

      - name: Run UI tests
        run: |
          mvn clean test \
            -Dgroups=ui \
            -Dselenide.headless=true \
            -Dselenide.browser=chrome \
            -Dallure.results.directory=${{ env.ALLURE_RESULTS }}
        env:
          APP_BASE_URL: ${{ secrets.APP_BASE_URL }}

      - name: Upload results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: allure-results-ui
          path: ${{ env.ALLURE_RESULTS }}/
          retention-days: 30

  # ============================================================
  # Job 3: Сборка Allure Report + деплой на GitHub Pages
  # ============================================================
  allure-report:
    name: Allure Report
    runs-on: ubuntu-latest
    needs: [api-tests, ui-tests]
    if: always()

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      # Скачиваем все allure-results артефакты
      - name: Download all results
        uses: actions/download-artifact@v4
        with:
          pattern: allure-results-*
          path: allure-results/
          merge-multiple: true

      # Скачиваем историю с gh-pages для графиков трендов
      - name: Get previous report history
        uses: actions/checkout@v4
        with:
          ref: gh-pages
          path: gh-pages
        continue-on-error: true

      - name: Copy history to results
        run: |
          if [ -d gh-pages/history ]; then
            echo "Копируем историю предыдущих отчётов..."
            cp -r gh-pages/history allure-results/history
          else
            echo "История не найдена, это первый запуск."
          fi

      # Генерация отчёта
      - name: Generate Allure Report
        uses: simple-elf/allure-report-action@v1.9
        with:
          allure_results: allure-results
          allure_history: allure-history
          keep_reports: 30

      # Деплой на GitHub Pages
      - name: Deploy report to GitHub Pages
        if: always()
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_branch: gh-pages
          publish_dir: allure-history

      # Добавляем комментарий к PR со ссылкой на отчёт
      - name: Comment PR with report link
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const reportUrl = `https://${context.repo.owner}.github.io/${context.repo.repo}/`;
            const body = `## Allure Report\n\nОтчёт о тестировании доступен по ссылке: ${reportUrl}\n\n` +
              `*Workflow:* \`${context.workflow}\`\n` +
              `*Run:* [#${context.runNumber}](${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId})`;
            github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: body
            });
```

---

## Шаг 6: Разделение тестов по suite через JUnit 5 Tags

Чтобы workflow мог запускать отдельные наборы тестов, используем JUnit 5 `@Tag`:

```java
import org.junit.jupiter.api.Tag;

// Все API-тесты помечаем тегом "api"
@Tag("api")
public class UserApiTest {

    @Test
    @Tag("smoke")
    void shouldGetCurrentUser() {
        // Этот тест входит и в api-suite, и в smoke-suite
    }

    @Test
    void shouldUpdateUserProfile() {
        // Этот тест входит только в api-suite
    }
}

// Все UI-тесты помечаем тегом "ui"
@Tag("ui")
public class LoginPageTest {

    @Test
    @Tag("smoke")
    void shouldLoginWithValidCredentials() {
        // Входит и в ui-suite, и в smoke-suite
    }
}
```

Настройка maven-surefire-plugin для фильтрации по тегам:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.5</version>
    <configuration>
        <!-- Передаём группы через -Dgroups=api,smoke -->
        <groups>${groups}</groups>
        <argLine>
            -javaagent:"${settings.localRepository}/org/aspectj/aspectjweaver/${aspectj.version}/aspectjweaver-${aspectj.version}.jar"
        </argLine>
    </configuration>
</plugin>
```

---

## Шаг 7: Уведомления о результатах

### 7.1 Уведомление в Slack (опционально)

Добавьте шаг в конец workflow:

```yaml
      - name: Notify Slack
        if: always()
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: |
            Regression results: ${{ job.status }}
            Report: https://${{ github.repository_owner }}.github.io/${{ github.event.repository.name }}/
          fields: repo,message,commit,author
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### 7.2 Уведомление в Telegram (опционально)

```yaml
      - name: Notify Telegram
        if: always()
        uses: appleboy/telegram-action@master
        with:
          to: ${{ secrets.TELEGRAM_CHAT_ID }}
          token: ${{ secrets.TELEGRAM_BOT_TOKEN }}
          message: |
            Regression: ${{ job.status }}
            Branch: ${{ github.ref_name }}
            Report: https://${{ github.repository_owner }}.github.io/${{ github.event.repository.name }}/
```

---

## Практические упражнения

### Упражнение 1: Создание базового pipeline

**Задание:**
1. Создайте файл `.github/workflows/test-on-pr.yml` с запуском `mvn clean test`
2. Создайте Pull Request с любым изменением
3. Убедитесь, что workflow запустился и тесты прошли
4. Найдите артефакты Allure в разделе **Actions** -> **Artifacts**

**Чек-лист готовности:**
- [ ] Workflow запускается при создании PR
- [ ] Тесты проходят в CI
- [ ] Артефакт с Allure-результатами доступен для скачивания
- [ ] В случае падения теста workflow показывает статус "failed"

### Упражнение 2: Настройка GitHub Pages

**Задание:**
1. Включите GitHub Pages (Settings -> Pages -> gh-pages branch)
2. Добавьте job с деплоем Allure Report на GitHub Pages
3. Запустите workflow и откройте отчёт по URL `https://<user>.github.io/<repo>/`
4. Проверьте, что графики трендов появляются после второго запуска

**Чек-лист готовности:**
- [ ] Allure Report доступен по URL на GitHub Pages
- [ ] Графики трендов отображаются (после 2+ запусков)
- [ ] История последних 20-30 отчётов сохраняется
- [ ] Отчёт обновляется при каждом новом запуске

### Упражнение 3: Scheduled regression

**Задание:**
1. Добавьте `schedule` триггер с cron-выражением
2. Добавьте `workflow_dispatch` для ручного запуска
3. Запустите workflow вручную через кнопку "Run workflow" в GitHub UI
4. Проверьте, что можно выбрать test suite (all / api / ui / smoke)

**Чек-лист готовности:**
- [ ] Workflow запускается по расписанию (проверьте утром следующего дня)
- [ ] Ручной запуск работает через UI GitHub
- [ ] Можно выбрать конкретный test suite при ручном запуске
- [ ] Результаты публикуются на GitHub Pages

### Упражнение 4: Полный pipeline с комментарием к PR

**Задание:**
1. Реализуйте полный `complete-pipeline.yml` из Шага 5
2. Добавьте автоматический комментарий к PR со ссылкой на отчёт
3. Создайте PR и проверьте, что появился комментарий с ссылкой
4. Перейдите по ссылке и убедитесь, что отчёт корректен

**Чек-лист готовности:**
- [ ] При PR запускаются API и UI тесты параллельно
- [ ] После тестов генерируется Allure Report
- [ ] Отчёт деплоится на GitHub Pages
- [ ] В PR появляется комментарий со ссылкой на отчёт
- [ ] Pipeline корректно обрабатывает падение тестов (отчёт всё равно создаётся)

---

## Типичные проблемы и решения

### Проблема: "Permission denied" при деплое на gh-pages

**Решение:** добавьте блок `permissions` в workflow:

```yaml
permissions:
  contents: write
  pages: write
```

Также в Settings -> Actions -> General -> Workflow permissions выберите
"Read and write permissions".

### Проблема: Chrome не запускается в CI

**Решение:** добавьте флаги для headless-режима:

```java
// В конфигурации Selenide
Configuration.headless = true;
Configuration.browser = "chrome";

// Или для Selenium напрямую
ChromeOptions options = new ChromeOptions();
options.addArguments("--headless=new");
options.addArguments("--no-sandbox");
options.addArguments("--disable-dev-shm-usage");  // Критично для CI
```

### Проблема: Тесты проходят локально, но падают в CI

**Частые причины:**
1. **Таймауты** -- CI-runner медленнее локальной машины. Увеличьте timeout'ы.
2. **Locale** -- на CI другая локаль. Даты, валюта могут отображаться иначе.
3. **Разрешение экрана** -- добавьте `Configuration.browserSize = "1920x1080"`.
4. **Переменные окружения** -- секреты не заданы или заданы неверно.
5. **Порядок тестов** -- в CI тесты могут запускаться в другом порядке.

### Проблема: Allure Report не показывает историю

**Решение:** убедитесь, что папка `history/` копируется из предыдущего отчёта.
Добавьте шаг checkout ветки `gh-pages` и копирование `history/` в `allure-results/`.

---

## Итоговый чек-лист этапа

После выполнения всех упражнений вы должны иметь:

- [ ] `.github/workflows/` с рабочими workflow-файлами
- [ ] Автоматический запуск тестов при каждом Pull Request
- [ ] Ночной scheduled regression
- [ ] Allure Report на GitHub Pages с историей трендов
- [ ] Настроенные secrets для URL и токенов
- [ ] Разделение тестов по suite через JUnit 5 `@Tag`
- [ ] Корректная обработка падений (отчёт генерируется всегда)
- [ ] (Опционально) Уведомления в Slack / Telegram

**Результат:** при каждом PR автоматически запускаются тесты, генерируется Allure-отчёт
и публикуется на GitHub Pages. Ссылка на отчёт автоматически появляется в комментарии
к PR. Ночью запускается полный regression. Это именно тот уровень автоматизации,
который ожидается от QA-инженера на реальном проекте.
