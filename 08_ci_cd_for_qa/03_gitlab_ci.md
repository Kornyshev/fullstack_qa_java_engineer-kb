# GitLab CI для QA

## Обзор

GitLab CI/CD — встроенная система непрерывной интеграции и доставки в платформе GitLab. Главное преимущество —
нативная интеграция с репозиторием: не нужно настраивать внешние инструменты, webhook-и и плагины.
Конфигурация pipeline описывается в файле `.gitlab-ci.yml` в корне репозитория. Для QA-инженера GitLab CI
предоставляет мощные возможности: артефакты с отчётами, environments для разных тестовых окружений, services
для запуска зависимостей (БД, Selenoid), а также встроенную поддержку merge request pipelines, что идеально
подходит для запуска тестов на каждый Pull Request.

---

## Структура .gitlab-ci.yml

### Базовые элементы

```yaml
# Образ Docker по умолчанию для всех jobs
image: maven:3.9-eclipse-temurin-17

# Определение этапов pipeline (порядок выполнения)
stages:
  - build
  - test
  - report
  - deploy

# Глобальные переменные
variables:
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"
  BASE_URL: "https://staging.example.com"

# Глобальный кэш для зависимостей
cache:
  key: "${CI_COMMIT_REF_SLUG}"
  paths:
    - .m2/repository/
```

### Полный пример для тестового проекта

```yaml
image: maven:3.9-eclipse-temurin-17

stages:
  - build
  - unit-test
  - api-test
  - ui-test
  - report

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"
  BASE_URL: "https://staging.example.com"
  BROWSER: "chrome"
  SELENOID_URL: "http://selenoid:4444/wd/hub"

cache:
  key: "${CI_COMMIT_REF_SLUG}"
  paths:
    - .m2/repository/
    - target/

# ============================================
# Этап: Сборка
# ============================================
build:
  stage: build
  script:
    - mvn clean compile -DskipTests -q
  artifacts:
    paths:
      - target/
    expire_in: 1 hour

# ============================================
# Этап: Unit-тесты
# ============================================
unit-tests:
  stage: unit-test
  script:
    - mvn test -Dgroups=unit -Dmaven.test.failure.ignore=true
  artifacts:
    when: always                          # Сохранять всегда (даже при провале)
    paths:
      - target/surefire-reports/
      - target/allure-results/
    reports:
      junit: target/surefire-reports/*.xml  # Встроенная интеграция JUnit в GitLab
    expire_in: 7 days

# ============================================
# Этап: API-тесты
# ============================================
api-tests:
  stage: api-test
  script:
    - mvn test -Dgroups=api -Dbase.url=${BASE_URL} -Dmaven.test.failure.ignore=true
  artifacts:
    when: always
    paths:
      - target/allure-results/
    reports:
      junit: target/surefire-reports/*.xml
    expire_in: 7 days
  rules:
    # Запускать на merge requests и на ветке main
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

# ============================================
# Этап: UI-тесты
# ============================================
ui-tests:
  stage: ui-test
  image: maven:3.9-eclipse-temurin-17
  services:
    # Selenoid как сервис для запуска браузеров
    - name: aerokube/selenoid:latest
      alias: selenoid
  script:
    - mvn test -Dgroups=ui
        -Dbase.url=${BASE_URL}
        -Dbrowser=${BROWSER}
        -Dselenide.remote=${SELENOID_URL}
        -Dmaven.test.failure.ignore=true
  artifacts:
    when: always
    paths:
      - target/allure-results/
      - target/screenshots/
      - target/video/
    reports:
      junit: target/surefire-reports/*.xml
    expire_in: 7 days
  rules:
    # UI-тесты — только на main и по расписанию (тяжёлые)
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_PIPELINE_SOURCE == "schedule"'

# ============================================
# Этап: Allure Report
# ============================================
allure-report:
  stage: report
  image: frankescobar/allure-docker-service
  script:
    - allure generate target/allure-results -o public/allure-report --clean
  artifacts:
    paths:
      - public/allure-report/
    expire_in: 30 days
  when: always                            # Запускать всегда, даже если тесты упали
```

---

## Jobs — задачи

Job — это базовый элемент pipeline. Каждый job выполняется в отдельном контейнере (или на runner).

### Ключевые директивы job

```yaml
my-test-job:
  stage: test                # Этап pipeline, к которому принадлежит job
  image: maven:3.9           # Docker-образ для выполнения
  services:                  # Дополнительные контейнеры (БД, Selenoid и т.д.)
    - postgres:15
  variables:                 # Переменные, специфичные для этого job
    DB_HOST: postgres
  before_script:             # Команды перед основным скриптом
    - echo "Подготовка окружения"
  script:                    # Основные команды (обязательная секция)
    - mvn test
  after_script:              # Команды после основного скрипта (выполняются всегда)
    - echo "Очистка"
  artifacts:                 # Артефакты для сохранения
    paths:
      - target/allure-results/
    when: always
    expire_in: 7 days
  cache:                     # Кэш для ускорения повторных запусков
    paths:
      - .m2/repository/
  tags:                      # Теги для выбора runner
    - docker
    - linux
  rules:                     # Условия запуска job
    - if: '$CI_COMMIT_BRANCH == "main"'
  timeout: 30m               # Таймаут выполнения
  retry: 2                   # Количество повторных попыток при провале
  allow_failure: false       # Разрешить провал без блокировки pipeline
```

---

## Runners — исполнители

GitLab Runner — это агент, который выполняет jobs. Runners бывают нескольких типов.

### Shared Runners

- Предоставляются GitLab.com (или администратором GitLab)
- Доступны всем проектам в группе
- Подходят для стандартных задач (сборка, unit-тесты)
- Могут иметь очередь (ожидание свободного runner)

### Specific (Project) Runners

- Привязаны к конкретному проекту
- Настраиваются командой/QA-инженером
- Подходят для специфичных задач (UI-тесты с Selenoid, performance-тесты)
- Гарантируют ресурсы без очереди

### Group Runners

- Доступны всем проектам в группе
- Удобны для организации: один runner на команду

### Регистрация Runner

```bash
# Установка GitLab Runner
sudo apt-get install gitlab-runner

# Регистрация runner с Docker executor
sudo gitlab-runner register \
  --url https://gitlab.company.com/ \
  --registration-token YOUR_TOKEN \
  --executor docker \
  --docker-image maven:3.9-eclipse-temurin-17 \
  --description "QA Test Runner" \
  --tag-list "docker,qa,selenium" \
  --docker-privileged \
  --docker-volumes "/var/run/docker.sock:/var/run/docker.sock"
```

### Выбор Runner через Tags

```yaml
ui-tests:
  tags:
    - qa           # Выбрать runner с тегом "qa"
    - selenium     # и тегом "selenium"
  script:
    - mvn test -Dgroups=ui
```

---

## Artifacts — артефакты

Артефакты позволяют сохранять файлы между jobs и после завершения pipeline.

```yaml
test-job:
  script:
    - mvn test -Dmaven.test.failure.ignore=true
  artifacts:
    when: always              # always | on_success | on_failure
    paths:                    # Файлы для сохранения
      - target/allure-results/
      - target/screenshots/
    reports:
      junit: target/surefire-reports/*.xml   # JUnit-отчёт (встроенный виджет GitLab)
    exclude:                  # Исключить файлы
      - target/classes/
    expire_in: 7 days         # Срок хранения
    name: "test-results-${CI_COMMIT_SHORT_SHA}"  # Имя архива
```

**Важно:** секция `reports: junit` позволяет GitLab отображать результаты тестов прямо в Merge Request —
упавшие тесты видны без перехода в pipeline.

---

## Variables — переменные

### Предопределённые переменные GitLab CI

| Переменная | Описание | Пример |
|-----------|----------|--------|
| `CI_COMMIT_BRANCH` | Текущая ветка | `main`, `feature/login` |
| `CI_COMMIT_SHORT_SHA` | Короткий хэш коммита | `a3b2c1d` |
| `CI_PIPELINE_SOURCE` | Источник запуска pipeline | `push`, `merge_request_event`, `schedule` |
| `CI_MERGE_REQUEST_SOURCE_BRANCH_NAME` | Ветка MR | `feature/login` |
| `CI_PROJECT_DIR` | Путь к проекту | `/builds/myproject` |
| `CI_JOB_TOKEN` | Токен для API GitLab | (автоматический) |
| `CI_ENVIRONMENT_NAME` | Имя окружения | `staging`, `production` |

### Пользовательские переменные

```yaml
# В .gitlab-ci.yml
variables:
  BASE_URL: "https://staging.example.com"    # Доступна всем jobs
  BROWSER: "chrome"

# Защищённые переменные (Settings → CI/CD → Variables)
# API_TOKEN, DB_PASSWORD — не хранятся в коде, только в настройках GitLab
```

**Приоритет переменных (от наибольшего к наименьшему):**
1. Trigger variables (API)
2. Schedule variables
3. Project-level variables (Settings → CI/CD)
4. Group-level variables
5. Instance-level variables
6. `.gitlab-ci.yml` variables
7. Предопределённые переменные

---

## Cache — кэш

Кэш ускоряет pipeline, сохраняя файлы между запусками (обычно — зависимости).

```yaml
# Глобальный кэш
cache:
  key: "${CI_COMMIT_REF_SLUG}"         # Ключ кэша — по ветке
  paths:
    - .m2/repository/                   # Maven-зависимости
    - node_modules/                     # NPM-зависимости (если есть фронтенд)
  policy: pull-push                     # pull — только читать, push — только писать

# Кэш конкретного job
build:
  cache:
    key:
      files:
        - pom.xml                       # Ключ основан на содержимом pom.xml
    paths:
      - .m2/repository/
```

**Разница Cache vs Artifacts:**
- **Cache** — для ускорения (зависимости). Может быть не найден — pipeline всё равно работает.
- **Artifacts** — для передачи данных между jobs и сохранения результатов. Гарантированно доступны.

---

## Rules / Only / Except — условия запуска

### Современный подход: `rules`

```yaml
api-tests:
  script:
    - mvn test -Dgroups=api
  rules:
    # На merge request — запускать smoke
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      variables:
        TEST_SUITE: "smoke"

    # На main ветку — запускать regression
    - if: '$CI_COMMIT_BRANCH == "main"'
      variables:
        TEST_SUITE: "regression"

    # По расписанию — полный набор
    - if: '$CI_PIPELINE_SOURCE == "schedule"'
      variables:
        TEST_SUITE: "full"

    # При изменении файлов тестов — всегда запускать
    - changes:
        - "src/test/**/*"
        - "pom.xml"
```

### Устаревший подход: `only` / `except`

```yaml
# Устаревший синтаксис (не рекомендуется для новых проектов)
nightly-regression:
  only:
    - schedules
  except:
    - tags
```

---

## Services — Docker-сервисы

Services позволяют запускать дополнительные Docker-контейнеры рядом с job (базы данных, message brokers, Selenoid).

```yaml
integration-tests:
  stage: test
  image: maven:3.9-eclipse-temurin-17
  services:
    # PostgreSQL для интеграционных тестов
    - name: postgres:15
      alias: db
      variables:
        POSTGRES_DB: testdb
        POSTGRES_USER: testuser
        POSTGRES_PASSWORD: testpass

    # Redis (если используется)
    - name: redis:7-alpine
      alias: cache-redis

  variables:
    # Подключение к сервисам по alias
    DB_URL: "jdbc:postgresql://db:5432/testdb"
    DB_USER: "testuser"
    DB_PASSWORD: "testpass"
    REDIS_HOST: "cache-redis"

  script:
    - mvn test -Dgroups=integration
        -Ddb.url=${DB_URL}
        -Ddb.user=${DB_USER}
        -Ddb.password=${DB_PASSWORD}
```

---

## Environments — окружения

Environments позволяют отслеживать, на какое окружение был сделан deploy, и управлять deployment-ами.

```yaml
deploy-staging:
  stage: deploy
  script:
    - deploy_to_staging.sh
  environment:
    name: staging
    url: https://staging.example.com
    on_stop: stop-staging                 # Job для остановки окружения

run-tests-staging:
  stage: test
  script:
    - mvn test -Dbase.url=https://staging.example.com
  environment:
    name: staging
    action: verify                        # Отметка о верификации окружения
```

---

## Связь с тестированием

GitLab CI предоставляет QA-инженеру ряд уникальных преимуществ:

- **Встроенные JUnit-отчёты в Merge Request:** упавшие тесты видны прямо в интерфейсе MR, без перехода
  в pipeline. Это ускоряет code review — ревьюер сразу видит, какие тесты сломались.
- **Test Cases и Test Reports:** GitLab отображает список тестов, время выполнения, историю стабильности.
- **Merge Request Pipelines:** тесты запускаются на merge-результате (main + feature branch), что
  обнаруживает конфликты до фактического merge.
- **Environments:** отслеживание, какая версия задеплоена на каком окружении, и возможность отката.
- **Review Apps:** автоматическое создание временного окружения для каждого Merge Request.

---

## Типичные ошибки

1. **Не используют `when: always` для артефактов.** По умолчанию артефакты сохраняются только при
   успешном завершении. Тестовые отчёты нужны именно при провале.

2. **Не настроен `reports: junit`.** Без этой секции результаты тестов не отображаются в Merge Request.
   Нужно убедиться, что путь к XML-файлам правильный.

3. **Неправильный кэш.** Кэш по `CI_COMMIT_REF_SLUG` создаёт отдельный кэш для каждой ветки, что
   приводит к холодному старту на новых ветках. Лучше использовать ключ на основе `pom.xml` или `package-lock.json`.

4. **Services без alias.** Без `alias` обращаться к сервису нужно по имени образа с заменой `/` и `:` на `-`.
   Всегда указывайте alias для читаемости.

5. **Отсутствие `maven.test.failure.ignore=true`.** Maven завершается с ошибкой при упавших тестах,
   и последующие jobs (генерация отчёта) не запускаются.

6. **Смешивание `rules` и `only/except`.** Эти директивы несовместимы в одном job. Используйте только `rules`.

7. **Слишком широкий `expire_in`.** Артефакты с большим сроком хранения занимают дисковое пространство.
   Для тестовых отчётов 7-14 дней обычно достаточно.

8. **Нет `retry` для нестабильных jobs.** Для UI-тестов разумно установить `retry: 1` или `retry: 2`,
   чтобы flaky-тесты не блокировали pipeline.

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Что такое `.gitlab-ci.yml`? Где он должен находиться?
2. Что такое stage и job в GitLab CI? Как они связаны?
3. Как сохранить результаты тестов (артефакты) после выполнения pipeline?
4. Что такое GitLab Runner? Какие типы runners вы знаете?
5. Как запустить pipeline вручную?

### 🟡 Средний уровень
6. Как настроить условный запуск job (например, только для merge request)?
7. В чём разница между `cache` и `artifacts`?
8. Как запустить тесты с зависимостями (БД, Redis) с помощью `services`?
9. Как настроить отображение результатов тестов в Merge Request?
10. Как организовать разные наборы тестов для разных событий (PR, merge, nightly)?
11. Как передать переменные в pipeline при ручном запуске?

### 🔴 Продвинутый уровень
12. Как настроить динамические environments для Review Apps с автоматическим запуском тестов?
13. Как реализовать child pipelines и включение шаблонов (`include`) для масштабируемой конфигурации?
14. Как настроить DAG (Directed Acyclic Graph) через `needs` для оптимизации порядка выполнения?
15. Как использовать GitLab Container Registry для хранения Docker-образов с тестовым окружением?
16. Как настроить multi-project pipeline для тестирования микросервисов?

---

## Практические задания

### Задание 1: Базовый .gitlab-ci.yml
Напишите `.gitlab-ci.yml` для Maven-проекта с автотестами:
- Stages: build, test, report
- Unit-тесты и API-тесты в разных jobs
- Артефакты: Allure results и JUnit XML
- Кэш для Maven-зависимостей

### Задание 2: Условный запуск
Расширьте конфигурацию из задания 1:
- На Merge Request: только unit-тесты и smoke API
- На merge в main: полная регрессия + UI-тесты
- По расписанию (ночь): кросс-браузерные тесты

### Задание 3: Services
Добавьте к pipeline интеграционные тесты, которым нужна PostgreSQL:
- Сервис `postgres:15` с настроенными переменными
- Правильная передача connection string в тесты
- Обработка случая, когда БД ещё не готова (health check)

### Задание 4: Миграция с Jenkins
Вам дан Jenkinsfile (declarative pipeline) для тестового проекта. Перепишите его в `.gitlab-ci.yml`,
сохранив ту же логику: параметры, параллельные этапы, артефакты, уведомления.

---

## Дополнительные ресурсы

- [GitLab CI/CD Documentation](https://docs.gitlab.com/ee/ci/) — официальная документация
- [GitLab CI/CD YAML Reference](https://docs.gitlab.com/ee/ci/yaml/) — полный справочник по синтаксису
- [GitLab CI/CD Examples](https://docs.gitlab.com/ee/ci/examples/) — примеры конфигураций
- [Predefined Variables](https://docs.gitlab.com/ee/ci/variables/predefined_variables.html) — все встроенные переменные
- [GitLab Runner Documentation](https://docs.gitlab.com/runner/) — настройка runners
- [Testing in GitLab](https://docs.gitlab.com/ee/ci/testing/) — тестирование в GitLab CI
