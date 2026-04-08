# GitHub Actions для QA

## Обзор

GitHub Actions — это встроенная платформа CI/CD от GitHub, которая позволяет автоматизировать сборку,
тестирование и развёртывание приложений прямо из репозитория. Для QA-инженера GitHub Actions привлекателен
нативной интеграцией с Pull Requests, богатым marketplace готовых actions, простотой настройки matrix-стратегий
для кросс-браузерного тестирования, а также бесплатными минутами для публичных репозиториев. Конфигурация
workflow хранится в директории `.github/workflows/` в формате YAML.

---

## Структура Workflow

### Иерархия компонентов

```
Repository
└── .github/
    └── workflows/
        ├── tests.yml           # Workflow для тестов
        ├── nightly.yml         # Workflow для ночной регрессии
        └── deploy.yml          # Workflow для deployment

Workflow (файл .yml)
└── Jobs (задачи)
    └── Steps (шаги)
        ├── Uses: action        # Использование готового action
        └── Run: command        # Выполнение shell-команды
```

### Базовая структура workflow

```yaml
# Имя workflow (отображается в UI GitHub)
name: Test Automation

# Триггеры — когда запускать workflow
on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

# Задачи
jobs:
  test:
    # Окружение выполнения
    runs-on: ubuntu-latest

    # Шаги
    steps:
      - name: Checkout code              # Получение кода из репозитория
        uses: actions/checkout@v4

      - name: Set up JDK 17              # Установка Java
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Run tests                  # Запуск тестов
        run: mvn test
```

---

## Triggers — триггеры запуска

GitHub Actions поддерживает множество событий для запуска workflow. Для QA наиболее важны следующие.

### Push и Pull Request

```yaml
on:
  push:
    branches:
      - main
      - develop
      - 'release/**'           # Все ветки release/*
    paths:
      - 'src/**'               # Только при изменении исходного кода
      - 'pom.xml'
    paths-ignore:
      - '**/*.md'              # Игнорировать изменения в документации

  pull_request:
    branches: [ main ]
    types: [ opened, synchronize, reopened ]  # Конкретные типы событий PR
```

### Schedule — запуск по расписанию

```yaml
on:
  schedule:
    # Каждый будний день в 02:00 UTC
    - cron: '0 2 * * 1-5'

    # Каждые 6 часов
    - cron: '0 */6 * * *'
```

### Workflow Dispatch — ручной запуск с параметрами

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Окружение для тестирования'
        required: true
        default: 'staging'
        type: choice
        options:
          - dev
          - staging
          - production
      browser:
        description: 'Браузер'
        required: true
        default: 'chrome'
        type: choice
        options:
          - chrome
          - firefox
          - edge
      test_suite:
        description: 'Набор тестов'
        required: true
        default: 'smoke'
        type: choice
        options:
          - smoke
          - regression
          - full
      run_performance:
        description: 'Запустить performance тесты'
        required: false
        type: boolean
        default: false
```

### Комбинация триггеров

```yaml
on:
  # На каждый PR — smoke-тесты
  pull_request:
    branches: [ main ]

  # На merge в main — regression
  push:
    branches: [ main ]

  # Каждую ночь — полный прогон
  schedule:
    - cron: '0 2 * * 1-5'

  # Ручной запуск
  workflow_dispatch:
    inputs:
      test_suite:
        type: choice
        options: [ smoke, regression, full ]
```

---

## Matrix Strategy — кросс-браузерное тестирование

Matrix strategy — одна из мощнейших фич GitHub Actions для QA. Позволяет автоматически создать
комбинации параметров и запустить job для каждой комбинации параллельно.

### Кросс-браузерные тесты

```yaml
jobs:
  ui-tests:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        browser: [ chrome, firefox, edge ]
        resolution: [ "1920x1080", "1366x768" ]
      fail-fast: false          # Не останавливать другие jobs при провале одного
      max-parallel: 3           # Максимум 3 параллельных job

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Run UI Tests (${{ matrix.browser }} @ ${{ matrix.resolution }})
        run: |
          mvn test -Dgroups=ui \
            -Dbrowser=${{ matrix.browser }} \
            -Dscreen.resolution=${{ matrix.resolution }} \
            -Dselenide.remote=http://localhost:4444/wd/hub

      - name: Upload Test Results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results-${{ matrix.browser }}-${{ matrix.resolution }}
          path: target/allure-results/
```

### Include и Exclude в Matrix

```yaml
strategy:
  matrix:
    os: [ ubuntu-latest, windows-latest, macos-latest ]
    java: [ '11', '17', '21' ]
    exclude:
      # Не тестировать Java 11 на macOS
      - os: macos-latest
        java: '11'
    include:
      # Добавить специфичную комбинацию с дополнительным параметром
      - os: ubuntu-latest
        java: '17'
        experimental: true
```

---

## Actions — готовые компоненты

Actions — это переиспользуемые компоненты из GitHub Marketplace. Для QA наиболее полезны следующие.

### Основные actions

```yaml
steps:
  # Получение кода из репозитория
  - name: Checkout
    uses: actions/checkout@v4

  # Установка Java
  - name: Set up JDK
    uses: actions/setup-java@v4
    with:
      java-version: '17'
      distribution: 'temurin'
      cache: 'maven'              # Встроенное кэширование Maven-зависимостей

  # Кэширование (для нестандартных случаев)
  - name: Cache Maven packages
    uses: actions/cache@v4
    with:
      path: ~/.m2/repository
      key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
      restore-keys: |
        ${{ runner.os }}-maven-

  # Загрузка артефактов
  - name: Upload artifacts
    uses: actions/upload-artifact@v4
    with:
      name: test-results
      path: target/allure-results/
      retention-days: 14

  # Скачивание артефактов (из другого job)
  - name: Download artifacts
    uses: actions/download-artifact@v4
    with:
      name: test-results
```

---

## Artifacts — артефакты

### Upload и Download

```yaml
jobs:
  run-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Run tests
        run: mvn test -Dmaven.test.failure.ignore=true

      # Сохранение результатов тестов
      - name: Upload Allure Results
        if: always()                     # Сохранять даже при провале тестов
        uses: actions/upload-artifact@v4
        with:
          name: allure-results
          path: target/allure-results/
          retention-days: 14

      # Сохранение скриншотов
      - name: Upload Screenshots
        if: failure()                    # Только при провале
        uses: actions/upload-artifact@v4
        with:
          name: screenshots
          path: target/screenshots/
          retention-days: 7

  # Отдельный job для генерации отчёта
  generate-report:
    needs: run-tests                     # Зависимость от предыдущего job
    runs-on: ubuntu-latest
    if: always()                         # Запускать всегда

    steps:
      - name: Download Allure Results
        uses: actions/download-artifact@v4
        with:
          name: allure-results
          path: allure-results/

      - name: Generate Allure Report
        uses: simple-elf/allure-report-action@master
        with:
          allure_results: allure-results
          allure_history: allure-history

      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: allure-history
          publish_branch: gh-pages
```

---

## Secrets — управление секретами

GitHub Secrets — безопасное хранение конфиденциальных данных (токенов, паролей, ключей).

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    env:
      # Привязка секретов к переменным окружения
      DB_PASSWORD: ${{ secrets.STAGING_DB_PASSWORD }}
      API_TOKEN: ${{ secrets.API_AUTH_TOKEN }}
    steps:
      - name: Run tests with credentials
        run: |
          mvn test \
            -Ddb.password=${{ secrets.STAGING_DB_PASSWORD }} \
            -Dapi.token=${{ secrets.API_AUTH_TOKEN }}

      # GITHUB_TOKEN — встроенный токен, доступен автоматически
      - name: Comment on PR
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: 'Тесты выполнены! Смотрите отчёт в артефактах.'
            })
```

**Уровни секретов:**
- **Repository secrets** — доступны только в текущем репозитории
- **Environment secrets** — доступны только при deploy в указанный environment
- **Organization secrets** — доступны всем репозиториям организации

---

## Allure Report на GitHub Pages

Развёртывание Allure Report на GitHub Pages — стандартный подход для QA-проектов на GitHub.

### Полный workflow для Allure Report

```yaml
name: Tests with Allure Report

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:

# Права для deployment на GitHub Pages
permissions:
  contents: write
  pages: write
  id-token: write

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'maven'

      - name: Run tests
        run: mvn test -Dmaven.test.failure.ignore=true
        continue-on-error: true

      - name: Get Allure history
        uses: actions/checkout@v4
        if: always()
        continue-on-error: true
        with:
          ref: gh-pages
          path: gh-pages

      - name: Generate Allure Report
        uses: simple-elf/allure-report-action@master
        if: always()
        with:
          allure_results: target/allure-results
          allure_history: allure-history
          keep_reports: 20               # Хранить 20 последних отчётов

      - name: Deploy to GitHub Pages
        if: always()
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: allure-history
          publish_branch: gh-pages
```

После настройки GitHub Pages (Settings → Pages → Source: gh-pages branch) отчёт доступен по адресу:
`https://<username>.github.io/<repository>/`

---

## Полный пример Workflow для тестового проекта

```yaml
name: QA Test Automation Pipeline

on:
  pull_request:
    branches: [ main ]
  push:
    branches: [ main ]
  schedule:
    - cron: '0 2 * * 1-5'               # Ночная регрессия в 02:00 по будням
  workflow_dispatch:
    inputs:
      test_suite:
        description: 'Набор тестов'
        type: choice
        default: 'smoke'
        options: [ smoke, regression, full ]
      browser:
        description: 'Браузер'
        type: choice
        default: 'chrome'
        options: [ chrome, firefox ]
      base_url:
        description: 'URL окружения'
        type: string
        default: 'https://staging.example.com'

permissions:
  contents: write
  pages: write
  checks: write

env:
  BASE_URL: ${{ github.event.inputs.base_url || 'https://staging.example.com' }}
  JAVA_VERSION: '17'

jobs:
  # =============================================
  # Job 1: Сборка и Unit-тесты (на каждый PR)
  # =============================================
  build-and-unit-tests:
    name: Build & Unit Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'maven'

      - name: Build project
        run: mvn clean compile -DskipTests -q

      - name: Run Unit Tests
        run: mvn test -Dgroups=unit -Dmaven.test.failure.ignore=true

      - name: Publish Unit Test Results
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: Unit Test Results
          path: target/surefire-reports/*.xml
          reporter: java-junit

      - name: Upload Allure Results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: unit-test-results
          path: target/allure-results/

  # =============================================
  # Job 2: API-тесты
  # =============================================
  api-tests:
    name: API Tests
    needs: build-and-unit-tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'maven'

      - name: Determine test suite
        id: suite
        run: |
          if [ "${{ github.event_name }}" == "pull_request" ]; then
            echo "suite=smoke" >> $GITHUB_OUTPUT
          elif [ "${{ github.event_name }}" == "schedule" ]; then
            echo "suite=full" >> $GITHUB_OUTPUT
          else
            echo "suite=${{ github.event.inputs.test_suite || 'regression' }}" >> $GITHUB_OUTPUT
          fi

      - name: Run API Tests (${{ steps.suite.outputs.suite }})
        run: |
          mvn test -Dgroups=api \
            -Dbase.url=${{ env.BASE_URL }} \
            -Dtest.suite=${{ steps.suite.outputs.suite }} \
            -Dmaven.test.failure.ignore=true

      - name: Upload Allure Results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: api-test-results
          path: target/allure-results/

  # =============================================
  # Job 3: UI-тесты (кросс-браузерные)
  # =============================================
  ui-tests:
    name: UI Tests (${{ matrix.browser }})
    needs: build-and-unit-tests
    if: github.event_name == 'push' || github.event_name == 'schedule' || github.event_name == 'workflow_dispatch'
    runs-on: ubuntu-latest
    strategy:
      matrix:
        browser: ${{ github.event_name == 'schedule' && fromJSON('["chrome","firefox"]') || fromJSON('["chrome"]') }}
      fail-fast: false

    services:
      selenoid:
        image: aerokube/selenoid:latest
        ports:
          - 4444:4444
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'maven'

      - name: Run UI Tests on ${{ matrix.browser }}
        run: |
          mvn test -Dgroups=ui \
            -Dbrowser=${{ matrix.browser }} \
            -Dbase.url=${{ env.BASE_URL }} \
            -Dselenide.remote=http://localhost:4444/wd/hub \
            -Dmaven.test.failure.ignore=true

      - name: Upload Allure Results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: ui-test-results-${{ matrix.browser }}
          path: target/allure-results/

      - name: Upload Screenshots
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: screenshots-${{ matrix.browser }}
          path: target/screenshots/

  # =============================================
  # Job 4: Генерация Allure Report
  # =============================================
  allure-report:
    name: Generate Allure Report
    needs: [ api-tests, ui-tests ]
    if: always()
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Download all test results
        uses: actions/download-artifact@v4
        with:
          path: allure-results/
          pattern: '*-test-results*'
          merge-multiple: true

      - name: Get Allure history
        uses: actions/checkout@v4
        continue-on-error: true
        with:
          ref: gh-pages
          path: gh-pages

      - name: Generate Allure Report
        uses: simple-elf/allure-report-action@master
        with:
          allure_results: allure-results
          allure_history: allure-history
          keep_reports: 30

      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: allure-history
          publish_branch: gh-pages
```

---

## Связь с тестированием

GitHub Actions предоставляет QA-инженеру ряд уникальных возможностей:

- **Pull Request Checks:** статус тестов отображается прямо в PR, блокируя merge при провале
  (настраивается через Branch Protection Rules)
- **Matrix strategy:** автоматическое создание кросс-браузерных, кросс-платформенных тестовых матриц
- **Workflow dispatch:** ручной запуск тестов с параметрами (окружение, браузер, набор тестов) —
  удобно для ad hoc тестирования
- **Marketplace:** тысячи готовых actions: генерация отчётов, уведомления, deployment
- **Бесплатные минуты:** для open source проектов GitHub Actions бесплатен
- **GitHub Pages:** бесплатный хостинг для Allure Report

---

## Типичные ошибки

1. **Забывают `if: always()`.** По умолчанию step или job не запускается, если предыдущий упал.
   Для загрузки артефактов и генерации отчётов нужно `if: always()`.

2. **Не используют `continue-on-error: true` для тестов.** Без этого workflow завершается при первом
   упавшем тесте, и артефакты не сохраняются. Альтернатива: `maven.test.failure.ignore=true`.

3. **Не кэшируют зависимости.** Без `cache: 'maven'` в `setup-java` или без `actions/cache` каждый
   запуск скачивает все зависимости заново — лишние 2-5 минут.

4. **`fail-fast: true` в matrix.** По умолчанию при провале одного варианта матрицы все остальные
   отменяются. Для тестирования нужно `fail-fast: false`.

5. **Неправильные permissions.** Для deployment на GitHub Pages нужны `contents: write` и `pages: write`.
   Без них deployment завершается ошибкой 403.

6. **Дублирование конфигурации.** Одинаковые steps копируются между workflows. Решение: composite
   actions или reusable workflows.

7. **Не используют `workflow_dispatch`.** Без ручного триггера нельзя запустить тесты на конкретном
   окружении или с конкретным набором тестов из UI GitHub.

8. **Секреты в логах.** GitHub автоматически маскирует secrets, но если секрет передаётся как часть
   URL, он может оказаться в логах. Используйте `::add-mask::` для дополнительной маскировки.

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Что такое GitHub Actions? Где хранятся конфигурации workflow?
2. Какие триггеры запуска workflow вы знаете?
3. Что такое action? Приведите примеры стандартных actions.
4. Как сохранить артефакты (результаты тестов) в GitHub Actions?
5. Что такое `runs-on` и какие runners доступны?

### 🟡 Средний уровень
6. Как настроить кросс-браузерное тестирование с помощью matrix strategy?
7. Как организовать ручной запуск workflow с параметрами (workflow_dispatch)?
8. В чём разница между `secrets` и `variables` в GitHub Actions?
9. Как настроить зависимости между jobs (`needs`)?
10. Как развернуть Allure Report на GitHub Pages?
11. Как настроить Branch Protection Rules для блокировки merge при провале тестов?

### 🔴 Продвинутый уровень
12. Как создать reusable workflow для переиспользования в нескольких репозиториях?
13. Как использовать environments с approval gates для deployment?
14. Как оптимизировать время workflow с помощью concurrency и caching?
15. Как создать composite action для инкапсуляции типовых шагов тестирования?
16. Как настроить self-hosted runner для специфичных задач (Selenoid, устройства)?

---

## Практические задания

### Задание 1: Базовый Workflow
Создайте `.github/workflows/tests.yml` для Maven-проекта:
- Триггеры: push в main, pull request
- Steps: checkout, setup-java (с кэшем), запуск тестов, загрузка артефактов
- Allure Report как артефакт

### Задание 2: Кросс-браузерное тестирование
Создайте workflow с matrix strategy:
- Браузеры: Chrome, Firefox, Edge
- Java: 17, 21
- `fail-fast: false`
- Отдельные артефакты для каждой комбинации

### Задание 3: Allure Report на GitHub Pages
Расширьте workflow из задания 1:
- Генерация Allure Report через action
- Deployment на GitHub Pages
- Хранение истории (последние 20 отчётов)

### Задание 4: Полный Pipeline
Создайте production-ready workflow:
- На PR: smoke-тесты (быстрые)
- На merge в main: полная регрессия + deploy на staging
- По расписанию: кросс-браузерные + performance
- Ручной запуск с параметрами
- Уведомления в Slack при провале

---

## Дополнительные ресурсы

- [GitHub Actions Documentation](https://docs.github.com/en/actions) — официальная документация
- [GitHub Actions Marketplace](https://github.com/marketplace?type=actions) — каталог готовых actions
- [Workflow Syntax Reference](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions) — полный справочник
- [Allure Report Action](https://github.com/simple-elf/allure-report-action) — action для Allure
- [GitHub Pages Deployment](https://github.com/peaceiris/actions-gh-pages) — deployment на GitHub Pages
- [Reusable Workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows) — переиспользуемые workflows
