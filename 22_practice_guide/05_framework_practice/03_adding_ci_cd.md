# Настройка CI/CD для тестового фреймворка

## Введение

Continuous Integration / Continuous Delivery (CI/CD) — это практика автоматического запуска
тестов при каждом изменении кода. Для QA-инженера это означает, что тесты запускаются
автоматически, результаты публикуются в виде отчёта, а команда получает обратную связь
без ручного вмешательства. В этом руководстве мы настроим полный pipeline для GitHub Actions
и рассмотрим альтернативу на GitLab CI.

> **Цель:** Создать CI/CD pipeline, который запускает тесты при каждом push и pull request,
> генерирует Allure-отчёт и публикует его как артефакт.

---

## Часть 1: GitHub Actions — полный pipeline

### Шаг 1: Создание структуры

```bash
# Создаём директорию для workflow-файлов
mkdir -p .github/workflows
```

### Шаг 2: Основной workflow — запуск тестов

Файл: `.github/workflows/tests.yml`

```yaml
# Название workflow — отображается во вкладке Actions на GitHub
name: Automated Tests

# Триггеры запуска pipeline
on:
  # Запуск при push в основные ветки
  push:
    branches:
      - main
      - develop

  # Запуск при создании pull request в main
  pull_request:
    branches:
      - main

  # Запуск по расписанию — ежедневный регрессионный прогон в 6:00 UTC
  schedule:
    - cron: '0 6 * * *'

  # Возможность запустить вручную из интерфейса GitHub
  workflow_dispatch:
    inputs:
      environment:
        description: 'Окружение для запуска тестов'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
      test_suite:
        description: 'Набор тестов для запуска'
        required: true
        default: 'all'
        type: choice
        options:
          - all
          - smoke
          - regression
          - api-only

# Переменные окружения, доступные во всех jobs
env:
  JAVA_VERSION: '17'
  MAVEN_OPTS: '-Xmx1024m'

jobs:
  # ============================================================
  # Job 1: Запуск тестов
  # ============================================================
  run-tests:
    name: Run Tests
    # Используем Ubuntu — стандартный runner для CI
    runs-on: ubuntu-latest

    # Таймаут — если тесты зависнут, job будет остановлен
    timeout-minutes: 30

    steps:
      # Шаг 1: Клонируем репозиторий
      - name: Checkout repository
        uses: actions/checkout@v4

      # Шаг 2: Устанавливаем Java
      - name: Set up JDK ${{ env.JAVA_VERSION }}
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'

      # Шаг 3: Кэшируем Maven-зависимости — ускоряет повторные запуски
      - name: Cache Maven dependencies
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          # Ключ кэша основан на хеше pom.xml — при изменении зависимостей кэш пересоздаётся
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: |
            ${{ runner.os }}-maven-

      # Шаг 4: Запуск Docker-контейнеров (если нужны для тестов)
      - name: Start test infrastructure
        run: |
          docker-compose -f docker-compose.test.yml up -d
          # Ждём, пока приложение станет доступным
          echo "Ожидание запуска приложения..."
          for i in {1..30}; do
            if curl -s http://localhost:3000/api/health > /dev/null 2>&1; then
              echo "Приложение готово!"
              break
            fi
            echo "Попытка $i/30..."
            sleep 5
          done

      # Шаг 5: Запуск тестов
      - name: Run tests
        run: |
          # Определяем набор тестов
          TEST_SUITE="${{ github.event.inputs.test_suite || 'all' }}"
          ENV="${{ github.event.inputs.environment || 'staging' }}"

          if [ "$TEST_SUITE" = "smoke" ]; then
            mvn clean test -Dgroups="smoke" -Denv=$ENV
          elif [ "$TEST_SUITE" = "api-only" ]; then
            mvn clean test -Dtest="*ApiTest" -Denv=$ENV
          else
            mvn clean test -Denv=$ENV
          fi
        # Продолжаем pipeline даже если тесты упали —
        # нужно сгенерировать отчёт
        continue-on-error: true

      # Шаг 6: Сохраняем результаты Allure как артефакт
      - name: Upload Allure results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: allure-results
          path: target/allure-results/
          retention-days: 30

      # Шаг 7: Сохраняем логи
      - name: Upload test logs
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: test-logs
          path: |
            target/surefire-reports/
            target/*.log

      # Шаг 8: Останавливаем инфраструктуру
      - name: Stop test infrastructure
        if: always()
        run: docker-compose -f docker-compose.test.yml down -v

  # ============================================================
  # Job 2: Генерация Allure-отчёта
  # ============================================================
  generate-report:
    name: Generate Allure Report
    needs: run-tests
    runs-on: ubuntu-latest
    if: always()

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      # Скачиваем результаты из предыдущего job
      - name: Download Allure results
        uses: actions/download-artifact@v4
        with:
          name: allure-results
          path: allure-results

      # Скачиваем историю предыдущих прогонов (если есть)
      - name: Get Allure history
        uses: actions/checkout@v4
        with:
          ref: gh-pages
          path: gh-pages
        continue-on-error: true

      # Генерируем отчёт с историей
      - name: Generate Allure Report
        uses: simple-elf/allure-report-action@v1.9
        with:
          allure_results: allure-results
          allure_history: allure-history
          keep_reports: 20

      # Публикуем отчёт на GitHub Pages
      - name: Deploy report to GitHub Pages
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_branch: gh-pages
          publish_dir: allure-history

      # Добавляем комментарий с ссылкой на отчёт в pull request
      - name: Comment PR with report link
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const reportUrl = `https://${context.repo.owner}.github.io/${context.repo.repo}/${context.runNumber}/`;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Отчёт о тестировании\n\n` +
                    `Allure Report: [Открыть отчёт](${reportUrl})\n\n` +
                    `Run: #${context.runNumber}`
            });

  # ============================================================
  # Job 3: Уведомление о результатах
  # ============================================================
  notify:
    name: Send Notification
    needs: [run-tests, generate-report]
    runs-on: ubuntu-latest
    if: always()

    steps:
      # Уведомление в Slack (требует настройки Slack Webhook)
      - name: Notify Slack
        if: env.SLACK_WEBHOOK_URL != ''
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ needs.run-tests.result }}
          text: |
            Тесты: ${{ needs.run-tests.result }}
            Ветка: ${{ github.ref_name }}
            Коммит: ${{ github.event.head_commit.message }}
            Отчёт: https://${{ github.repository_owner }}.github.io/${{ github.event.repository.name }}/
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## Часть 2: Workflow для только API-тестов

Файл: `.github/workflows/api-tests.yml`

```yaml
name: API Tests (Fast Feedback)

on:
  pull_request:
    branches: [main, develop]
    # Запуск только при изменении определённых файлов
    paths:
      - 'src/test/java/api/**'
      - 'src/test/java/models/**'
      - 'pom.xml'

jobs:
  api-tests:
    name: API Tests
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Cache Maven
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}

      # API-тесты не требуют браузера — быстрый запуск
      - name: Run API tests
        run: mvn clean test -Dtest="*ApiTest" -Denv=staging
        continue-on-error: true

      - name: Upload results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: api-test-results
          path: target/allure-results/
```

---

## Часть 3: Секреты и переменные окружения

### Настройка секретов в GitHub

```
GitHub Repository → Settings → Secrets and variables → Actions

Добавить секреты:
- STAGING_URL          → https://staging.myapp.com
- TEST_USER_EMAIL      → test@staging.com
- TEST_USER_PASSWORD   → s3cur3P@ss
- SLACK_WEBHOOK_URL    → https://hooks.slack.com/services/...
- DB_CONNECTION_STRING  → jdbc:postgresql://...
```

### Использование секретов в workflow

```yaml
steps:
  - name: Run tests with secrets
    run: mvn clean test -Denv=staging
    env:
      # Секреты передаются как переменные окружения
      BASE_URL: ${{ secrets.STAGING_URL }}
      TEST_EMAIL: ${{ secrets.TEST_USER_EMAIL }}
      TEST_PASSWORD: ${{ secrets.TEST_USER_PASSWORD }}
```

### Обновление ConfigReader для поддержки переменных окружения

```java
public class ConfigReader {

    // Приоритет: переменная окружения > системное свойство > properties-файл
    public static String getBaseUrl() {
        // Сначала проверяем переменную окружения (для CI/CD)
        String envValue = System.getenv("BASE_URL");
        if (envValue != null && !envValue.isEmpty()) {
            return envValue;
        }
        // Затем системное свойство (для -D параметров)
        String sysProp = System.getProperty("base.url");
        if (sysProp != null && !sysProp.isEmpty()) {
            return sysProp;
        }
        // Наконец, значение из properties-файла
        return properties.getProperty("base.url");
    }
}
```

---

## Часть 4: GitLab CI — альтернативная конфигурация

Файл: `.gitlab-ci.yml`

```yaml
# Базовый Docker-образ с Java и Maven
image: maven:3.9-eclipse-temurin-17

# Этапы pipeline
stages:
  - build          # Сборка проекта
  - test           # Запуск тестов
  - report         # Генерация отчёта
  - notify         # Уведомления

# Глобальные переменные
variables:
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"

# Кэширование Maven-зависимостей между pipeline
cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - .m2/repository/

# ===========================================
# Этап: Сборка
# ===========================================
build:
  stage: build
  script:
    - mvn clean compile -DskipTests
  # Артефакты передаются между этапами
  artifacts:
    paths:
      - target/

# ===========================================
# Этап: Тесты
# ===========================================

# Smoke-тесты — быстрая проверка при каждом push
smoke-tests:
  stage: test
  script:
    - mvn test -Dgroups="smoke" -Denv=staging
  artifacts:
    when: always
    paths:
      - target/allure-results/
      - target/surefire-reports/
    expire_in: 7 days
  # Запуск при каждом push
  rules:
    - if: '$CI_PIPELINE_SOURCE == "push"'

# Полный регрессионный прогон — по расписанию и вручную
regression-tests:
  stage: test
  script:
    - mvn test -Denv=staging
  artifacts:
    when: always
    paths:
      - target/allure-results/
    expire_in: 30 days
  timeout: 45 minutes
  rules:
    # Запуск по расписанию (настраивается в GitLab CI/CD → Schedules)
    - if: '$CI_PIPELINE_SOURCE == "schedule"'
    # Запуск вручную
    - if: '$CI_PIPELINE_SOURCE == "web"'
  # Использование сервисов — PostgreSQL и Selenoid
  services:
    - name: postgres:15
      alias: test-db
      variables:
        POSTGRES_DB: testdb
        POSTGRES_USER: test
        POSTGRES_PASSWORD: test
    - name: aerokube/selenoid:latest
      alias: selenoid

# API-тесты — при изменении API-тестов
api-tests:
  stage: test
  script:
    - mvn test -Dtest="*ApiTest" -Denv=staging
  artifacts:
    when: always
    paths:
      - target/allure-results/
    expire_in: 7 days
  rules:
    - if: '$CI_MERGE_REQUEST_ID'
      changes:
        - src/test/java/api/**
        - src/test/java/models/**

# ===========================================
# Этап: Отчёт
# ===========================================
allure-report:
  stage: report
  image: frankescobar/allure-docker-service
  script:
    - allure generate target/allure-results -o allure-report --clean
  artifacts:
    paths:
      - allure-report/
    expire_in: 30 days
  when: always
  # Allure-отчёт публикуется как GitLab Pages
  pages:
    stage: report
    script:
      - mkdir -p public
      - cp -r allure-report/* public/
    artifacts:
      paths:
        - public
    when: always

# ===========================================
# Этап: Уведомление
# ===========================================
notify-slack:
  stage: notify
  image: curlimages/curl:latest
  script:
    - |
      curl -X POST "$SLACK_WEBHOOK" \
        -H 'Content-type: application/json' \
        -d "{
          \"text\": \"Pipeline: $CI_PIPELINE_URL\nСтатус: $CI_JOB_STATUS\nВетка: $CI_COMMIT_REF_NAME\"
        }"
  when: always
  rules:
    - if: '$SLACK_WEBHOOK'
```

---

## Часть 5: Матрица запуска — тесты на разных браузерах

```yaml
# Добавьте в .github/workflows/tests.yml
jobs:
  cross-browser-tests:
    name: Tests on ${{ matrix.browser }}
    runs-on: ubuntu-latest
    # Матричная стратегия — параллельный запуск на разных браузерах
    strategy:
      matrix:
        browser: [chrome, firefox, edge]
      # Не останавливать другие браузеры, если один упал
      fail-fast: false

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Run tests on ${{ matrix.browser }}
        run: mvn clean test -Dbrowser=${{ matrix.browser }} -Dheadless=true
        continue-on-error: true

      - name: Upload results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          # Уникальное имя артефакта для каждого браузера
          name: allure-results-${{ matrix.browser }}
          path: target/allure-results/
```

---

## Практическое задание

### Задание 1: Базовый pipeline на GitHub Actions

1. Создайте файл `.github/workflows/tests.yml`
2. Настройте триггеры: push в main, pull request, ручной запуск
3. Добавьте шаги: checkout, setup Java, cache Maven, run tests
4. Настройте upload артефактов с результатами Allure
5. Сделайте push и убедитесь, что pipeline запустился

**Ожидаемый результат:** Зелёный (или красный, если тесты падают) статус pipeline
во вкладке Actions на GitHub. Артефакт с результатами Allure доступен для скачивания.

### Задание 2: Allure Report на GitHub Pages

1. Добавьте job для генерации Allure-отчёта
2. Настройте публикацию на GitHub Pages
3. Добавьте комментарий со ссылкой на отчёт в pull request
4. Проверьте, что отчёт доступен по URL

### Задание 3: Scheduled regression

1. Добавьте триггер `schedule` с cron-выражением для ежедневного запуска
2. Настройте уведомление в Slack (или email) о результатах
3. Добавьте матричный запуск на Chrome и Firefox
4. Проверьте, что scheduled run запускается корректно

**Критерии оценки:**
- Pipeline запускается автоматически при push и pull request
- Allure-отчёт генерируется и доступен как артефакт
- Секреты не хранятся в коде — используются GitHub Secrets
- Pipeline содержит комментарии, объясняющие каждый шаг
- При падении тестов pipeline не прерывается до генерации отчёта

---

## Чек-лист самопроверки

- [ ] Создан `.github/workflows/tests.yml` с корректной структурой
- [ ] Настроены триггеры: push, pull_request, schedule, workflow_dispatch
- [ ] Maven-зависимости кэшируются между запусками
- [ ] Результаты тестов сохраняются как артефакты
- [ ] Allure-отчёт генерируется и публикуется
- [ ] Секреты настроены в GitHub Settings
- [ ] ConfigReader поддерживает переменные окружения для CI/CD
- [ ] Pipeline продолжает работу после падения тестов (`continue-on-error: true`)
- [ ] Есть scheduled job для ежедневного регрессионного прогона
- [ ] Понимаю разницу между GitHub Actions и GitLab CI
