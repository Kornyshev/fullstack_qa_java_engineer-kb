# Описание итогового проекта

## Введение

Итоговый проект — это комплексный тестовый фреймворк для реального веб-приложения.
Он объединяет все навыки, полученные в процессе обучения: ручное тестирование,
API-автоматизацию, UI-автоматизацию, работу с базой данных, репортинг и CI/CD.
Результат — полноценный проект для портфолио, который демонстрирует уровень
fullstack QA-инженера.

> **Цель:** Создать production-ready тестовый фреймворк, который можно показать
> работодателю как подтверждение ваших навыков.

---

## Приложение под тестирование: Conduit (RealWorld)

### Что такое Conduit

Conduit — это открытая блог-платформа, реализация спецификации
[RealWorld](https://github.com/gothinkster/realworld). Это реальное приложение
с REST API, базой данных и пользовательским интерфейсом.

### Функциональность Conduit

| Модуль | Функции |
|--------|---------|
| **Аутентификация** | Регистрация, логин, логаут, JWT-токены |
| **Профиль** | Просмотр, редактирование, аватар, био |
| **Статьи** | Создание, редактирование, удаление, просмотр |
| **Теги** | Добавление к статьям, фильтрация по тегам |
| **Комментарии** | Добавление, удаление, просмотр |
| **Подписки** | Подписка/отписка от авторов |
| **Лайки** | Добавление/удаление из избранного |
| **Лента** | Глобальная лента, персональная лента (подписки) |

### Развёртывание приложения

```bash
# Вариант 1: Готовый онлайн-инстанс
# https://demo.realworld.io (может быть недоступен)

# Вариант 2: Локальный запуск через Docker
git clone https://github.com/gothinkster/realworld-example-apps.git
cd realworld-example-apps

# Запуск backend (Node.js + PostgreSQL)
docker-compose up -d

# Приложение доступно:
# API:  http://localhost:3000/api
# UI:   http://localhost:4100

# Вариант 3: Собственный docker-compose (рекомендуется)
# Используйте docker-compose.yml из раздела "Docker для тестового окружения"
```

### API-документация

Основные эндпоинты Conduit API:

```
POST   /api/users           — Регистрация
POST   /api/users/login     — Авторизация
GET    /api/user             — Текущий пользователь
PUT    /api/user             — Обновление профиля

GET    /api/articles         — Список статей
POST   /api/articles         — Создание статьи
GET    /api/articles/:slug   — Получение статьи
PUT    /api/articles/:slug   — Обновление статьи
DELETE /api/articles/:slug   — Удаление статьи

POST   /api/articles/:slug/comments      — Добавить комментарий
GET    /api/articles/:slug/comments      — Список комментариев
DELETE /api/articles/:slug/comments/:id  — Удалить комментарий

POST   /api/articles/:slug/favorite      — Добавить в избранное
DELETE /api/articles/:slug/favorite      — Удалить из избранного

POST   /api/profiles/:username/follow    — Подписаться
DELETE /api/profiles/:username/follow    — Отписаться

GET    /api/tags             — Список тегов
```

---

## Технологический стек проекта

### Основные технологии

| Категория | Технология | Назначение |
|-----------|-----------|------------|
| Язык | Java 17 | Основной язык автоматизации |
| Сборка | Maven 3.9+ | Управление зависимостями, запуск тестов |
| Тест-фреймворк | JUnit 5 | Запуск и организация тестов |
| API-тестирование | REST Assured 5.x | HTTP-запросы, валидация ответов |
| UI-тестирование | Selenide 7.x | Взаимодействие с браузером |
| Assertions | AssertJ 3.x | Читаемые проверки |
| JSON | Jackson 2.x | Сериализация/десериализация |
| Генерация данных | JavaFaker 2.x | Тестовые данные |
| Репортинг | Allure 2.25+ | Отчёты о тестировании |
| Контейнеры | Docker + TestContainers | Изолированное окружение |
| CI/CD | GitHub Actions | Автоматический запуск тестов |
| Параллелизм | JUnit 5 Parallel + Selenoid | Параллельный запуск |
| БД | PostgreSQL + JDBC | Верификация данных |

### Версии зависимостей (pom.xml)

```xml
<properties>
    <java.version>17</java.version>
    <junit.version>5.10.2</junit.version>
    <rest-assured.version>5.4.0</rest-assured.version>
    <selenide.version>7.2.3</selenide.version>
    <allure.version>2.25.0</allure.version>
    <assertj.version>3.25.3</assertj.version>
    <jackson.version>2.17.0</jackson.version>
    <faker.version>2.1.0</faker.version>
    <testcontainers.version>1.19.7</testcontainers.version>
    <aspectj.version>1.9.21</aspectj.version>
    <hikaricp.version>5.1.0</hikaricp.version>
    <postgresql.version>42.7.3</postgresql.version>
</properties>
```

---

## Структура проекта

```
conduit-test-framework/
├── pom.xml
├── README.md
├── .github/
│   └── workflows/
│       ├── tests.yml                  # CI/CD pipeline
│       └── api-tests.yml             # Быстрый API-прогон
├── docker-compose.yml                 # Тестовое окружение
├── Dockerfile.tests                   # Контейнер для запуска тестов
├── selenoid/
│   └── browsers.json                  # Конфигурация Selenoid
│
├── src/test/java/
│   ├── api/                           # API-слой
│   │   ├── client/
│   │   │   ├── ApiClient.java         # Базовый HTTP-клиент
│   │   │   ├── AuthApi.java           # Эндпоинты авторизации
│   │   │   ├── ArticleApi.java        # Эндпоинты статей
│   │   │   ├── CommentApi.java        # Эндпоинты комментариев
│   │   │   └── ProfileApi.java        # Эндпоинты профилей
│   │   ├── models/                    # POJO-модели для API
│   │   │   ├── request/
│   │   │   │   ├── UserRequest.java
│   │   │   │   ├── ArticleRequest.java
│   │   │   │   └── CommentRequest.java
│   │   │   └── response/
│   │   │       ├── UserResponse.java
│   │   │       ├── ArticleResponse.java
│   │   │       ├── CommentResponse.java
│   │   │       └── ErrorResponse.java
│   │   └── tests/                     # API-тесты
│   │       ├── AuthApiTest.java
│   │       ├── ArticleApiTest.java
│   │       ├── CommentApiTest.java
│   │       ├── ProfileApiTest.java
│   │       └── TagApiTest.java
│   │
│   ├── ui/                            # UI-слой
│   │   ├── pages/                     # Page Objects
│   │   │   ├── BasePage.java
│   │   │   ├── HomePage.java
│   │   │   ├── LoginPage.java
│   │   │   ├── RegisterPage.java
│   │   │   ├── ArticlePage.java
│   │   │   ├── EditorPage.java
│   │   │   ├── ProfilePage.java
│   │   │   └── SettingsPage.java
│   │   └── tests/                     # UI-тесты
│   │       ├── LoginUiTest.java
│   │       ├── RegisterUiTest.java
│   │       ├── ArticleUiTest.java
│   │       ├── CommentUiTest.java
│   │       └── ProfileUiTest.java
│   │
│   ├── db/                            # Database-слой
│   │   ├── DatabaseConnection.java
│   │   ├── DatabaseClient.java
│   │   ├── repository/
│   │   │   ├── UserRepository.java
│   │   │   └── ArticleRepository.java
│   │   └── entity/
│   │       ├── UserEntity.java
│   │       └── ArticleEntity.java
│   │
│   ├── integration/                   # Интеграционные тесты (все слои)
│   │   ├── ArticleE2ETest.java
│   │   └── UserFlowE2ETest.java
│   │
│   ├── config/                        # Конфигурация
│   │   └── ConfigReader.java
│   │
│   └── utils/                         # Утилиты
│       ├── DataGenerator.java
│       ├── RetryUtils.java
│       ├── DatabaseWaiter.java
│       └── AllureUtils.java
│
├── src/test/resources/
│   ├── config.properties              # Локальная конфигурация
│   ├── config-staging.properties      # Staging-окружение
│   ├── config-docker.properties       # Docker-окружение
│   ├── allure.properties              # Настройки Allure
│   ├── junit-platform.properties      # Параллельный запуск
│   ├── categories.json                # Категории для Allure
│   ├── json-schemas/                  # JSON Schema для валидации
│   │   ├── user-response.json
│   │   ├── article-response.json
│   │   └── error-response.json
│   └── db/
│       └── init-schema.sql            # Схема для TestContainers
│
└── docs/                              # Документация (опционально)
    ├── test-plan.md
    └── test-cases.md
```

---

## Этапы выполнения проекта

### Обзор этапов

| Этап | Название | Продолжительность | Основной навык |
|------|----------|-------------------|----------------|
| 1 | Ручное тестирование | 1 неделя | Тест-дизайн, документация |
| 2 | API-автоматизация | 2 недели | REST Assured, POJO, JSON Schema |
| 3 | UI-автоматизация | 2 недели | Selenide, Page Object Model |
| 4 | Репортинг (Allure) | 3-4 дня | Allure Report, аннотации |
| 5 | CI/CD | 3-4 дня | GitHub Actions, Docker |
| 6 | Финализация | 3-4 дня | Рефакторинг, документация |

### Общая продолжительность: 6-8 недель

---

## Этап 1: Ручное тестирование (подробно в файле 02_stage_manual.md)

**Цель:** Разработать полную тестовую документацию.

**Результат:**
- Тест-план
- 50+ тест-кейсов
- Чек-листы для каждого модуля
- Шаблон баг-репорта с примерами

---

## Этап 2: API-автоматизация (подробно в файле 03_stage_api_automation.md)

**Цель:** Автоматизировать все API-эндпоинты.

**Результат:**
- 40+ API-тестов
- POJO-модели для request/response
- JSON Schema валидация
- Allure-аннотации для API

---

## Этап 3: UI-автоматизация (подробно в файле 04_stage_ui_automation.md)

**Цель:** Автоматизировать ключевые UI-сценарии.

**Результат:**
- 20+ UI-тестов
- Page Object Model (8+ страниц)
- Скриншоты при падении
- Параллельный запуск

---

## Этап 4: Репортинг (подробно в файле 05_stage_reporting.md)

**Цель:** Настроить полноценный Allure Report.

**Результат:**
- Иерархия Epic/Feature/Story
- Скриншоты и логи при падении
- Categories и environment
- История прогонов

---

## Этап 5: CI/CD

**Цель:** Автоматизировать запуск тестов.

**Результат:**
- GitHub Actions workflow
- Docker-окружение
- Scheduled regression
- Allure Report на GitHub Pages

---

## Этап 6: Финализация

**Цель:** Подготовить проект для портфолио.

**Результат:**
- README.md с описанием проекта, инструкцией по запуску и скриншотами
- Чистый код (рефакторинг, именование, комментарии)
- Все тесты проходят стабильно
- CI/CD работает корректно

---

## Критерии готовности проекта (Definition of Done)

### Минимальные требования (MVP)

- [ ] 40+ API-тестов, покрывающих все эндпоинты
- [ ] 15+ UI-тестов для основных пользовательских сценариев
- [ ] Page Object Model для всех тестируемых страниц
- [ ] POJO-модели для API request/response
- [ ] Конфигурация вынесена в properties-файлы
- [ ] Allure Report с аннотациями @Epic/@Feature/@Story
- [ ] README.md с инструкцией по запуску
- [ ] Все тесты проходят (green build)

### Продвинутые требования

- [ ] JSON Schema валидация ответов API
- [ ] Верификация данных в БД (JDBC)
- [ ] Параллельный запуск тестов
- [ ] Docker-окружение (docker-compose.yml)
- [ ] CI/CD pipeline (GitHub Actions)
- [ ] Selenoid для параллельных браузеров
- [ ] TestContainers для изолированной БД
- [ ] Allure Report с историей прогонов на GitHub Pages

### Экспертные требования

- [ ] Многослойные E2E-тесты (API + UI + DB)
- [ ] Генерация тестовых данных (Faker)
- [ ] Retry-механизм для нестабильных операций
- [ ] Cross-browser тестирование
- [ ] Allure Categories с классификацией дефектов
- [ ] Scheduled regression в CI/CD
- [ ] Уведомления о результатах (Slack/Email)
- [ ] Документация тест-плана и тест-кейсов

---

## Как представить проект работодателю

### GitHub Repository

```
Название: conduit-test-framework
Описание: Comprehensive test automation framework for Conduit (RealWorld)
          blog platform — API + UI + DB testing with Allure reporting and CI/CD

Topics: qa, test-automation, java, rest-assured, selenide, allure,
        docker, github-actions, page-object-model
```

### Ключевые элементы README.md

1. **Заголовок и бейджи** — статус CI, процент покрытия
2. **Описание** — что тестирует фреймворк, какие технологии использует
3. **Скриншоты** — Allure Report, Selenoid UI, CI/CD pipeline
4. **Quick Start** — как клонировать и запустить за 2 минуты
5. **Архитектура** — диаграмма слоёв фреймворка
6. **Тестовое покрытие** — таблица эндпоинтов и статус автоматизации

### На собеседовании

Будьте готовы рассказать:
- Почему выбрали именно эту архитектуру
- Как обеспечивается потокобезопасность при параллельном запуске
- Какие проблемы возникали и как их решали
- Как устроен CI/CD pipeline
- Какие подходы к тест-дизайну использовали

---

## Практическое задание: Начало проекта

### Шаг 1: Создание репозитория

1. Создайте репозиторий на GitHub: `conduit-test-framework`
2. Инициализируйте Maven-проект с необходимыми зависимостями
3. Создайте структуру пакетов
4. Настройте .gitignore

### Шаг 2: Первый тест

1. Разверните Conduit локально
2. Напишите первый API-тест: регистрация пользователя
3. Напишите первый UI-тест: логин
4. Убедитесь, что оба теста проходят

### Шаг 3: Базовая инфраструктура

1. Создайте `ConfigReader` и `config.properties`
2. Создайте `DataGenerator` для уникальных тестовых данных
3. Создайте `BaseTest` с setUp/tearDown
4. Сделайте первый коммит

---

## Чек-лист самопроверки

- [ ] Понимаю функциональность Conduit и все API-эндпоинты
- [ ] Определён технологический стек и версии зависимостей
- [ ] Знаю структуру проекта и назначение каждого пакета
- [ ] Понимаю последовательность этапов и их взаимосвязь
- [ ] Создал репозиторий на GitHub
- [ ] Развернул Conduit локально (или в Docker)
- [ ] Написал и запустил первый тест
- [ ] Готов приступить к этапу ручного тестирования
