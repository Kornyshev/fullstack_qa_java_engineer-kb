# 22_practice_guide/ — Рекомендации по практике для начинающего QA

## Обзор

Главная проблема начинающего тестировщика — «на чём тренироваться?». В отличие от разработчика, который может просто начать писать код, тестировщику нужен **объект тестирования** — приложение, API, документация. Этот раздел содержит конкретные рекомендации: что тестировать, где практиковаться, какие артефакты создавать на каждом этапе.

---

## Структура директории

```
22_practice_guide/
├── 01_practice_overview.md                # Общий план практики: с чего начать, в каком порядке
│
├── 02_manual_testing/
│   ├── 01_testing_web_apps.md             # Практика ручного тестирования на открытых веб-приложениях
│   ├── 02_writing_test_cases.md           # Практика написания тест-кейсов и чек-листов
│   ├── 03_writing_bug_reports.md          # Практика написания баг-репортов (на реальных багах)
│   ├── 04_test_design_exercises.md        # Упражнения по техникам тест-дизайна (BVA, EP, Decision Tables)
│   └── 05_exploratory_testing.md          # Практика exploratory testing с session-based подходом
│
├── 03_api_testing_practice/
│   ├── 01_postman_basics.md               # Практика с Postman: коллекции, переменные, тесты, окружения
│   ├── 02_public_apis.md                  # Список открытых API для практики + задания
│   ├── 03_rest_assured_exercises.md       # Пошаговые упражнения: от первого теста до полного сьюта
│   └── 04_api_test_project.md             # Мини-проект: полное покрытие одного публичного API автотестами
│
├── 04_ui_automation_practice/
│   ├── 01_practice_websites.md            # Список сайтов для практики UI-автоматизации (с описанием что на каждом тренировать)
│   ├── 02_selenium_first_project.md       # Пошагово: первый проект на Selenium + JUnit + POM
│   ├── 03_selenide_first_project.md       # Пошагово: первый проект на Selenide + Allure
│   ├── 04_playwright_first_project.md     # Пошагово: первый проект на Playwright Java
│   └── 05_ui_project_saucedemo.md         # Полный мини-проект: автоматизация SauceDemo от структуры до Allure-отчёта
│
├── 05_framework_practice/
│   ├── 01_framework_evolution.md          # Эволюция фреймворка: от плоских тестов к слоистой архитектуре
│   ├── 02_adding_allure.md                # Практика: подключение Allure, аннотации, скриншоты, отчёт
│   ├── 03_adding_ci_cd.md                 # Практика: настройка GitHub Actions / GitLab CI для запуска тестов
│   ├── 04_adding_parallel.md              # Практика: параллельный запуск тестов (JUnit 5 / TestNG + Selenoid)
│   └── 05_adding_docker.md               # Практика: docker-compose для тестового окружения (app + DB + Selenoid)
│
├── 06_database_practice/
│   ├── 01_sql_exercises.md                # SQL-упражнения: от SELECT до оконных функций (на реальных схемах)
│   └── 02_db_in_tests.md                  # Практика: проверка данных в БД из автотестов (JDBC + TestContainers)
│
├── 07_full_project/
│   ├── 01_project_overview.md             # Описание итогового проекта: полный фреймворк для тестирования реального приложения
│   ├── 02_stage_manual.md                 # Этап 1: Ручное тестирование — тест-план, тест-кейсы, чек-листы, баг-репорты
│   ├── 03_stage_api_automation.md         # Этап 2: API-автоматизация — REST Assured, POJO, JSON Schema
│   ├── 04_stage_ui_automation.md          # Этап 3: UI-автоматизация — Selenide/Playwright, POM, параллельный запуск
│   ├── 05_stage_reporting.md              # Этап 4: Репортинг — Allure Report, скриншоты, логи, категории
│   ├── 06_stage_ci_cd.md                  # Этап 5: CI/CD — GitHub Actions, Allure отчёт как артефакт, scheduled runs
│   └── 07_stage_advanced.md               # Этап 6: Продвинутое — Kafka testing, DB checks, Docker, мониторинг
│
└── 08_resources.md                        # Полезные ресурсы: книги, курсы, YouTube-каналы, сообщества, чаты
```

---

## Детальное описание каждого файла

### 01_practice_overview.md

**Общий план практики — дорожная карта для начинающего QA.**

Содержание:
- В каком порядке осваивать навыки (ручное → API Postman → API автоматизация → UI автоматизация → фреймворк → CI/CD)
- Сколько времени отводить на каждый этап (ориентировочно)
- Как фиксировать прогресс: GitHub-репозитории как портфолио
- Что показывать на интервью из практических работ
- Типичные ошибки: «сразу в автоматизацию» без ручного понимания, «бесконечное изучение теории» без практики

---

### 02_manual_testing/ — Практика ручного тестирования

#### 01_testing_web_apps.md

**Открытые веб-приложения для практики ручного тестирования:**

| Приложение | URL | Что тренировать |
|-----------|-----|-----------------|
| SauceDemo | saucedemo.com | Логин, каталог, корзина, чекаут — классический e-commerce |
| The Internet (Heroku) | the-internet.herokuapp.com | Набор изолированных страниц: формы, drag-and-drop, alerts, iframes |
| DemoQA | demoqa.com | Формы, виджеты, взаимодействия — для отработки разных UI-элементов |
| Conduit (RealWorld) | demo.realworld.io | Полноценный блог-движок: регистрация, публикации, комменты, лайки |
| Juice Shop (OWASP) | juice-shop.herokuapp.com | Интернет-магазин с намеренными уязвимостями — для security testing |
| Reqres | reqres.in | Fake REST API с UI — для API-практики |
| OpenCart Demo | demo.opencart.com | Полноценный интернет-магазин — для комплексного тестирования |

Для каждого приложения: описание, что конкретно тестировать, пример задания.

#### 02_writing_test_cases.md

**Практические задания по написанию тест-кейсов:**
- Задание 1: Написать 10 тест-кейсов для формы регистрации (позитивные + негативные)
- Задание 2: Написать чек-лист для страницы корзины e-commerce
- Задание 3: Составить тест-кейсы для процесса оформления заказа (multi-step)
- Шаблон тест-кейса с обязательными полями
- Примеры хороших и плохих тест-кейсов
- Уровни детализации: когда подробный тест-кейс, когда чек-лист

#### 03_writing_bug_reports.md

**Практика написания баг-репортов:**
- Список заведомо «сломанных» приложений с известными багами
- Задание: найти баги, оформить баг-репорты по шаблону
- Шаблон баг-репорта: Summary, Steps to Reproduce, Expected/Actual Result, Severity, Priority, Environment, Attachments
- Примеры хороших и плохих баг-репортов
- Практика определения Severity vs Priority

#### 04_test_design_exercises.md

**Упражнения по техникам тест-дизайна:**
- EP + BVA: поле «возраст» (1–120), поле «email», поле «пароль» (8–20 символов)
- Decision Table: скидочная система (VIP + промокод + праздник)
- State Transition: статусы заказа (Created → Paid → Shipped → Delivered → Returned)
- Pairwise: тестирование формы с 4 полями по 3 значения (PICT tool)
- Каждое упражнение с решением

#### 05_exploratory_testing.md

**Практика exploratory testing:**
- Что такое session-based test management (SBTM)
- Шаблон сессии: charter, time box, notes, bugs found
- Задание: 30-минутная exploratory session на SauceDemo
- Как документировать находки
- Когда exploratory testing эффективнее скриптового

---

### 03_api_testing_practice/ — Практика API-тестирования

#### 01_postman_basics.md

**Первые шаги в API-тестировании через Postman:**
- Установка Postman, создание workspace
- Первый запрос: GET к публичному API
- Коллекции, переменные (environment, collection, global)
- Написание тестов в Postman (JavaScript): проверка статуса, тела, заголовков
- Runner: запуск коллекции, data-driven через CSV
- Экспорт коллекции для портфолио

#### 02_public_apis.md

**Список открытых API для практики:**

| API | URL | Что тренировать |
|-----|-----|-----------------|
| JSONPlaceholder | jsonplaceholder.typicode.com | CRUD: posts, comments, users — идеально для начала |
| Reqres | reqres.in | Регистрация, логин, пользователи — с документацией |
| PetStore (Swagger) | petstore.swagger.io | Полный OpenAPI spec, CRUD для магазина питомцев |
| RestCountries | restcountries.com | GET-запросы с фильтрацией — для отработки query params |
| CoinGecko | api.coingecko.com | Криптовалюты — реальный API с rate limiting |
| OpenWeatherMap | openweathermap.org/api | Погода — нужен API key, реальный сценарий |
| GitHub API | api.github.com | OAuth, пагинация, вложенные ресурсы — продвинутый уровень |

Для каждого API: описание, примеры запросов, конкретные задания.

#### 03_rest_assured_exercises.md

**Пошаговые упражнения по REST Assured:**
- Упражнение 1: GET-запрос, проверка статуса и тела
- Упражнение 2: POST с JSON-телом, десериализация ответа в POJO
- Упражнение 3: Параметризованные тесты с @MethodSource
- Упражнение 4: Аутентификация (Bearer token)
- Упражнение 5: JSON Schema Validation
- Упражнение 6: Request/Response Specification для DRY
- Каждое упражнение: задание → подсказка → полное решение

#### 04_api_test_project.md

**Мини-проект: полное покрытие JSONPlaceholder API:**
- Структура проекта (Maven, packages, config)
- POJO-модели для всех сущностей
- Тесты на все CRUD-операции для /posts, /users, /comments
- Негативные сценарии: 404, невалидные данные
- Allure-репортинг
- README с описанием и инструкцией запуска

---

### 04_ui_automation_practice/ — Практика UI-автоматизации

#### 01_practice_websites.md

**Сайты для практики UI-автоматизации с описанием, что на каждом тренировать:**

| Сайт | Что тренировать |
|------|-----------------|
| SauceDemo | Логин, каталог, корзина, чекаут — полный e-commerce flow |
| The Internet | Изолированные элементы: checkboxes, dropdowns, drag-and-drop, file upload, alerts, iframes, shadow DOM |
| DemoQA | Формы, таблицы, виджеты, interactions — разнообразные UI-компоненты |
| Automation Practice | automationpractice.com — устаревший, но популярный для POM-практики |
| UI Playground | uitestingplayground.com — специально для отработки сложных ожиданий и динамических элементов |
| Conduit (RealWorld) | Полноценное SPA — для E2E сценариев |

#### 02_selenium_first_project.md — 05_ui_project_saucedemo.md

**Пошаговые гайды по созданию первого UI-проекта на каждом инструменте:**
- Создание проекта (Maven/Gradle)
- Подключение зависимостей
- Базовый Page Object (BasePage)
- Первая страница (LoginPage)
- Первый тест
- Добавление конфигурации (браузер, URL, timeout)
- Запуск и анализ результатов
- Финальный проект SauceDemo: полное покрытие с Allure

---

### 05_framework_practice/ — Эволюция фреймворка

#### 01_framework_evolution.md

**От плоских тестов к архитектуре:**
- Этап 1: Тесты без POM (всё в одном файле) — почему это плохо
- Этап 2: Page Object Model — выносим логику страниц
- Этап 3: Конфигурация — выносим URL, браузер, timeouts
- Этап 4: Утилиты — генерация данных, работа с файлами
- Этап 5: Многослойность — UI + API + DB в одном фреймворке
- На каждом этапе: проблема → решение → пример кода

#### 02_adding_allure.md — 05_adding_docker.md

**Практика расширения фреймворка:**
- Каждый файл — пошаговый гайд по добавлению одной capability
- Allure: зависимости → аннотации → скриншоты → отчёт
- CI/CD: .yml файл → stages → артефакты → scheduled run
- Параллельный запуск: конфигурация → thread safety → Selenoid
- Docker: docker-compose → app + DB → Selenoid → всё вместе

---

### 06_database_practice/ — Практика SQL

#### 01_sql_exercises.md

**SQL-упражнения на реальных схемах:**
- Рекомендуемые платформы: SQLBolt, W3Schools SQL, HackerRank SQL, LeetCode Database
- Упражнения по нарастающей: SELECT → WHERE → JOIN → GROUP BY → подзапросы → оконные функции
- Каждое упражнение привязано к контексту тестирования: «проверить, что заказ создался в БД»

#### 02_db_in_tests.md

**Практика: проверка данных в БД из автотестов:**
- JDBC: подключение, выполнение запросов, маппинг результатов
- TestContainers: поднять PostgreSQL в Docker, запустить миграции, выполнить тесты
- Пример: API-тест создаёт заказ → SQL-проверка, что запись появилась в БД с правильными данными

---

### 07_full_project/ — Итоговый проект

**Полный фреймворк тестирования для реального приложения — проект для портфолио.**

Рекомендуемое тестируемое приложение: **Conduit (RealWorld)** — open-source блог-платформа с REST API + UI.

#### 02_stage_manual.md — 07_stage_advanced.md

**6 этапов, каждый добавляет новый уровень:**

| Этап | Что делаем | Артефакт |
|------|-----------|----------|
| 1. Ручное | Тест-план, тест-кейсы, чек-листы, баг-репорты | Google Docs / TestRail |
| 2. API | REST Assured тесты на все эндпоинты Conduit API | Java-проект + Allure |
| 3. UI | Selenide/Playwright тесты на ключевые сценарии | POM + скриншоты |
| 4. Reporting | Allure Report с @Epic/@Feature/@Story, категории | Allure-отчёт |
| 5. CI/CD | GitHub Actions: запуск на PR, scheduled regression, Allure Pages | .github/workflows/ |
| 6. Advanced | DB-проверки, Docker-compose, Kafka (если доступно) | Полный фреймворк |

Каждый этап: пошаговая инструкция, ожидаемый результат, чеклист «готово, если...».

---

### 08_resources.md

**Полезные ресурсы для самостоятельного обучения:**

- **Книги:** «Тестирование dot com» (Савин), «Rapid Software Testing» (Bach, Bolton), «Java. Библиотека профессионала» (Хорстманн)
- **YouTube:** Баранцев, Тучс, Автоматизация с нуля, Artsiom Rusau QA Life
- **Курсы:** Stepik (бесплатные и платные), EPAM Learn, Baeldung
- **Сообщества:** Telegram-чаты (QA — pair, Автоматизация QA), Reddit r/QualityAssurance
- **Платформы для практики:** SQLBolt, HackerRank, LeetCode, Codewars
- **Документация:** Selenium, Selenide, Playwright, REST Assured, JUnit 5, Allure — официальные docs
