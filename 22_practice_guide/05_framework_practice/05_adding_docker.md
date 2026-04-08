# Docker для тестового окружения

## Введение

Docker решает главную проблему тестирования: «У меня на машине работает, а у тебя нет».
С Docker каждый член команды, каждый CI-сервер запускает тесты в одинаковом окружении.
В этом руководстве мы создадим полное тестовое окружение: приложение, база данных,
Selenoid для браузеров и контейнер для запуска тестов.

> **Цель:** Развернуть полное тестовое окружение в Docker, написать docker-compose.yml
> со всеми сервисами, создать Dockerfile для тестового runner-а и запустить тесты
> в изолированной среде.

---

## Часть 1: Установка Docker

### macOS

```bash
# Вариант 1: Docker Desktop (GUI)
# Скачать с https://www.docker.com/products/docker-desktop/

# Вариант 2: Через Homebrew
brew install --cask docker

# Проверка установки
docker --version
docker-compose --version
```

### Linux (Ubuntu)

```bash
# Обновление пакетов
sudo apt-get update

# Установка зависимостей
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common

# Добавление GPG-ключа Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Добавление репозитория Docker
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Установка Docker
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Добавление текущего пользователя в группу docker (чтобы не писать sudo)
sudo usermod -aG docker $USER

# Проверка
docker --version
docker compose version
```

### Проверка работоспособности

```bash
# Запуск тестового контейнера
docker run hello-world

# Если видите "Hello from Docker!" — всё работает
```

---

## Часть 2: Архитектура тестового окружения

### Схема сервисов

```
┌─────────────────────────────────────────────────────────┐
│                    Docker Network                        │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐ │
│  │   App    │  │ PostgreSQL│  │ Selenoid │  │ Test    │ │
│  │  :3000   │  │  :5432   │  │  :4444   │  │ Runner  │ │
│  │          │◄─┤          │  │          │  │         │ │
│  │ Conduit  │  │  testdb  │  │ Chrome   │  │ Maven   │ │
│  │ Backend  │  │          │  │ Firefox  │  │ + Java  │ │
│  └──────────┘  └──────────┘  └──────────┘  └─────────┘ │
│       ▲                           ▲              │      │
│       │         ┌─────────┐       │              │      │
│       └─────────┤Selenoid │───────┘              │      │
│                 │  UI     │                      │      │
│                 │  :8080  │◄─────────────────────┘      │
│                 └─────────┘                              │
└─────────────────────────────────────────────────────────┘
```

---

## Часть 3: docker-compose.yml — основной файл

```yaml
# docker-compose.yml — полное тестовое окружение
version: '3.8'

services:
  # ==========================================
  # PostgreSQL — база данных приложения
  # ==========================================
  postgres:
    image: postgres:15-alpine
    container_name: test-postgres
    environment:
      # Параметры базы данных
      POSTGRES_DB: conduit
      POSTGRES_USER: conduit_user
      POSTGRES_PASSWORD: conduit_pass
    ports:
      # Проброс порта для доступа с хоста (для отладки)
      - "5432:5432"
    volumes:
      # SQL-скрипты для инициализации БД выполняются автоматически
      - ./db/init:/docker-entrypoint-initdb.d
      # Персистентное хранилище данных
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      # Проверка готовности БД — другие сервисы ждут этого
      test: ["CMD-SHELL", "pg_isready -U conduit_user -d conduit"]
      interval: 5s
      timeout: 3s
      retries: 10
    networks:
      - test-network

  # ==========================================
  # Приложение — Conduit Backend
  # ==========================================
  app:
    image: conduit-backend:latest
    # Или сборка из исходников:
    # build:
    #   context: ./backend
    #   dockerfile: Dockerfile
    container_name: test-app
    environment:
      # Подключение к PostgreSQL внутри Docker-сети
      DATABASE_URL: postgresql://conduit_user:conduit_pass@postgres:5432/conduit
      JWT_SECRET: test-secret-key-for-jwt-tokens
      NODE_ENV: test
      PORT: 3000
    ports:
      - "3000:3000"
    depends_on:
      postgres:
        # Ждём, пока PostgreSQL пройдёт healthcheck
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/api/health"]
      interval: 10s
      timeout: 5s
      retries: 10
    networks:
      - test-network

  # ==========================================
  # Selenoid — браузеры в Docker-контейнерах
  # ==========================================
  selenoid:
    image: aerokube/selenoid:latest
    container_name: test-selenoid
    volumes:
      # Конфигурация браузеров
      - ./selenoid/browsers.json:/etc/selenoid/browsers.json
      # Docker socket — Selenoid управляет контейнерами браузеров
      - /var/run/docker.sock:/var/run/docker.sock
      # Директория для видеозаписей тестов
      - ./selenoid/video:/opt/selenoid/video
      # Логи браузеров
      - ./selenoid/logs:/opt/selenoid/logs
    environment:
      # Максимальное количество одновременных браузеров
      - OVERRIDE_VIDEO_OUTPUT_DIR=/opt/selenoid/video
    command:
      - "-conf"
      - "/etc/selenoid/browsers.json"
      - "-video-output-dir"
      - "/opt/selenoid/video"
      - "-log-output-dir"
      - "/opt/selenoid/logs"
      # Лимит одновременных сессий
      - "-limit"
      - "8"
      # Таймаут бездействия — закрывать браузер, если тест завис
      - "-timeout"
      - "5m0s"
    ports:
      - "4444:4444"
    networks:
      - test-network

  # ==========================================
  # Selenoid UI — веб-интерфейс для мониторинга
  # ==========================================
  selenoid-ui:
    image: aerokube/selenoid-ui:latest
    container_name: test-selenoid-ui
    ports:
      - "8080:8080"
    depends_on:
      - selenoid
    command:
      - "--selenoid-uri"
      - "http://selenoid:4444"
    networks:
      - test-network

  # ==========================================
  # Allure Report Server — просмотр отчётов
  # ==========================================
  allure:
    image: frankescobar/allure-docker-service:latest
    container_name: test-allure
    environment:
      CHECK_RESULTS_EVERY_SECONDS: 5
      KEEP_HISTORY: "TRUE"
      KEEP_HISTORY_LATEST: 25
    ports:
      - "5050:5050"
    volumes:
      # Результаты тестов монтируются сюда
      - allure_results:/app/allure-results
      - allure_reports:/app/default-reports
    networks:
      - test-network

# ==========================================
# Volumes — именованные тома
# ==========================================
volumes:
  postgres_data:
    driver: local
  allure_results:
    driver: local
  allure_reports:
    driver: local

# ==========================================
# Network — общая сеть для всех сервисов
# ==========================================
networks:
  test-network:
    driver: bridge
```

---

## Часть 4: Dockerfile для тестового runner-а

```dockerfile
# Dockerfile.tests — контейнер для запуска тестов
# Базовый образ с Maven и Java 17
FROM maven:3.9-eclipse-temurin-17 AS test-runner

# Метаданные
LABEL maintainer="qa-team"
LABEL description="Контейнер для запуска автотестов"

# Рабочая директория внутри контейнера
WORKDIR /app

# Копируем pom.xml отдельно — для кэширования зависимостей
COPY pom.xml .

# Скачиваем все зависимости (этот слой кэшируется, если pom.xml не менялся)
RUN mvn dependency:go-offline -B

# Копируем исходный код тестов
COPY src/ ./src/

# Копируем конфигурационные файлы
COPY testng.xml ./
COPY src/test/resources/ ./src/test/resources/

# Переменные окружения по умолчанию
ENV BASE_URL=http://app:3000
ENV SELENOID_URL=http://selenoid:4444/wd/hub
ENV DB_URL=jdbc:postgresql://postgres:5432/conduit
ENV DB_USER=conduit_user
ENV DB_PASSWORD=conduit_pass
ENV BROWSER=chrome
ENV HEADLESS=true
ENV RUN_MODE=remote
ENV ALLURE_RESULTS_DIR=/app/target/allure-results

# Точка входа — запуск тестов
ENTRYPOINT ["mvn", "clean", "test"]

# Аргументы по умолчанию (можно переопределить при запуске)
CMD ["-Denv=docker"]
```

### Добавление runner-а в docker-compose.yml

```yaml
  # ==========================================
  # Test Runner — контейнер для запуска тестов
  # ==========================================
  test-runner:
    build:
      context: .
      dockerfile: Dockerfile.tests
    container_name: test-runner
    environment:
      BASE_URL: http://app:3000
      SELENOID_URL: http://selenoid:4444/wd/hub
      DB_URL: jdbc:postgresql://postgres:5432/conduit
      DB_USER: conduit_user
      DB_PASSWORD: conduit_pass
      BROWSER: chrome
      RUN_MODE: remote
    depends_on:
      app:
        condition: service_healthy
      selenoid:
        condition: service_started
    volumes:
      # Результаты тестов доступны на хосте
      - ./test-results:/app/target/allure-results
      # Для Allure-сервера
      - allure_results:/app/target/allure-results
    networks:
      - test-network
```

---

## Часть 5: Конфигурация Selenoid — browsers.json

```json
{
  "chrome": {
    "default": "125.0",
    "versions": {
      "125.0": {
        "image": "selenoid/vnc:chrome_125.0",
        "port": "4444",
        "path": "/",
        "tmpfs": {
          "/tmp": "size=512m"
        },
        "shmSize": 1073741824,
        "env": [
          "SCREEN_RESOLUTION=1920x1080x24",
          "ENABLE_VNC=true"
        ]
      }
    }
  },
  "firefox": {
    "default": "126.0",
    "versions": {
      "126.0": {
        "image": "selenoid/vnc:firefox_126.0",
        "port": "4444",
        "path": "/wd/hub",
        "tmpfs": {
          "/tmp": "size=512m"
        },
        "shmSize": 1073741824,
        "env": [
          "SCREEN_RESOLUTION=1920x1080x24",
          "ENABLE_VNC=true"
        ]
      }
    }
  }
}
```

### Предварительная загрузка Docker-образов браузеров

```bash
# Selenoid использует Docker-образы для браузеров — их нужно скачать заранее
docker pull selenoid/vnc:chrome_125.0
docker pull selenoid/vnc:firefox_126.0
docker pull selenoid/video-recorder:latest-release

# Проверка загруженных образов
docker images | grep selenoid
```

---

## Часть 6: Пошаговый запуск

### Шаг 1: Подготовка структуры файлов

```bash
# Структура проекта
project/
├── docker-compose.yml          # Основной файл с сервисами
├── Dockerfile.tests            # Dockerfile для тестового runner-а
├── pom.xml                     # Maven-конфигурация
├── src/
│   └── test/
│       ├── java/               # Тестовый код
│       └── resources/
│           └── config-docker.properties  # Конфигурация для Docker
├── selenoid/
│   ├── browsers.json           # Конфигурация браузеров Selenoid
│   ├── video/                  # Директория для видеозаписей
│   └── logs/                   # Директория для логов
├── db/
│   └── init/
│       └── 01_schema.sql       # SQL-скрипт инициализации БД
└── test-results/               # Результаты тестов (mount)
```

### Шаг 2: Файл конфигурации для Docker-окружения

Файл: `src/test/resources/config-docker.properties`

```properties
# URL приложения — имя сервиса из docker-compose
base.url=http://app:3000
# Selenoid — удалённый WebDriver
selenoid.url=http://selenoid:4444/wd/hub
run.mode=remote
# База данных — имя сервиса из docker-compose
db.url=jdbc:postgresql://postgres:5432/conduit
db.user=conduit_user
db.password=conduit_pass
# Браузер
browser=chrome
headless=true
# Таймауты (в секундах)
timeout.explicit=15
timeout.page.load=30
```

### Шаг 3: Запуск окружения

```bash
# 1. Запускаем инфраструктуру (без тестового runner-а)
docker-compose up -d postgres app selenoid selenoid-ui allure

# 2. Проверяем статус сервисов
docker-compose ps

# 3. Ожидаемый вывод:
# NAME             STATUS           PORTS
# test-postgres    Up (healthy)     0.0.0.0:5432->5432/tcp
# test-app         Up (healthy)     0.0.0.0:3000->3000/tcp
# test-selenoid    Up               0.0.0.0:4444->4444/tcp
# test-selenoid-ui Up               0.0.0.0:8080->8080/tcp
# test-allure      Up               0.0.0.0:5050->5050/tcp

# 4. Проверяем доступность приложения
curl http://localhost:3000/api/health

# 5. Открываем Selenoid UI в браузере
# http://localhost:8080
```

### Шаг 4: Запуск тестов

```bash
# Вариант 1: Запуск тестов в контейнере
docker-compose up test-runner

# Вариант 2: Запуск тестов локально против Docker-окружения
mvn clean test -Denv=docker \
  -Dbase.url=http://localhost:3000 \
  -Dselenoid.url=http://localhost:4444/wd/hub \
  -Drun.mode=remote

# Вариант 3: Запуск конкретного набора тестов
docker-compose run --rm test-runner \
  mvn clean test -Dgroups="smoke" -Denv=docker

# Вариант 4: Запуск с переопределением браузера
docker-compose run --rm test-runner \
  mvn clean test -Dbrowser=firefox -Denv=docker
```

### Шаг 5: Просмотр результатов

```bash
# Allure-отчёт доступен в браузере
# http://localhost:5050

# Видеозаписи тестов
ls selenoid/video/

# Логи тестов
docker-compose logs test-runner

# Логи приложения (для отладки падений)
docker-compose logs app
```

### Шаг 6: Остановка и очистка

```bash
# Остановка всех сервисов
docker-compose down

# Остановка с удалением данных (volumes)
docker-compose down -v

# Удаление всех образов проекта
docker-compose down --rmi all

# Очистка неиспользуемых ресурсов Docker
docker system prune -f
```

---

## Часть 7: Полезные команды Docker

### Отладка

```bash
# Просмотр логов конкретного сервиса в реальном времени
docker-compose logs -f app

# Вход внутрь контейнера (для отладки)
docker exec -it test-app /bin/sh

# Проверка сетевого взаимодействия между контейнерами
docker exec -it test-runner ping app

# Просмотр использования ресурсов
docker stats

# Проверка, что контейнеры видят друг друга в сети
docker network inspect test-network
```

### Пересборка после изменений

```bash
# Пересборка тестового runner-а (при изменении тестов)
docker-compose build test-runner

# Пересборка без кэша (при изменении pom.xml)
docker-compose build --no-cache test-runner

# Пересборка и запуск
docker-compose up --build test-runner
```

---

## Часть 8: docker-compose для CI/CD

Файл: `docker-compose.ci.yml` — облегчённая версия для CI

```yaml
version: '3.8'

# Используется вместе с основным файлом:
# docker-compose -f docker-compose.yml -f docker-compose.ci.yml up

services:
  # В CI не нужен Selenoid UI — экономим ресурсы
  selenoid-ui:
    deploy:
      replicas: 0

  # В CI не нужен Allure-сервер — отчёт генерируется отдельно
  allure:
    deploy:
      replicas: 0

  # Переопределяем настройки для CI
  test-runner:
    environment:
      HEADLESS: "true"
      BROWSER: chrome
      # В CI используем все доступные ядра
      MAVEN_OPTS: "-Xmx2g"
```

### Использование в GitHub Actions

```yaml
- name: Run tests in Docker
  run: |
    docker-compose -f docker-compose.yml -f docker-compose.ci.yml up -d postgres app selenoid
    # Ждём готовности
    docker-compose -f docker-compose.yml exec -T app curl --retry 10 --retry-delay 5 http://localhost:3000/api/health
    # Запускаем тесты
    docker-compose -f docker-compose.yml -f docker-compose.ci.yml up --exit-code-from test-runner test-runner
```

---

## Практическое задание

### Задание 1: Базовое окружение

1. Установите Docker на свою машину
2. Создайте `docker-compose.yml` с PostgreSQL и приложением
3. Запустите `docker-compose up -d` и убедитесь, что приложение доступно
4. Подключитесь к PostgreSQL через DBeaver/pgAdmin и проверьте структуру БД

**Ожидаемый результат:** Приложение запущено в Docker и доступно по адресу
`http://localhost:3000`, база данных содержит необходимые таблицы.

### Задание 2: Selenoid

1. Добавьте Selenoid и Selenoid UI в docker-compose.yml
2. Создайте browsers.json с конфигурацией Chrome
3. Загрузите Docker-образ браузера
4. Откройте Selenoid UI и проверьте доступные браузеры
5. Модифицируйте DriverManager для работы с RemoteWebDriver
6. Запустите один тест и наблюдайте его в Selenoid UI

### Задание 3: Полный pipeline в Docker

1. Создайте `Dockerfile.tests` для тестового runner-а
2. Добавьте `test-runner` в docker-compose.yml
3. Запустите полный прогон: `docker-compose up test-runner`
4. Проверьте результаты в директории `test-results/`
5. Откройте Allure-отчёт через Allure-сервер на `http://localhost:5050`

**Критерии оценки:**
- Все сервисы стартуют без ошибок (`docker-compose ps` — все Healthy/Up)
- Тесты запускаются внутри контейнера и подключаются к Selenoid
- Результаты тестов доступны на хост-машине через volume mount
- Окружение полностью воспроизводимо (любой может клонировать и запустить)

---

## Чек-лист самопроверки

- [ ] Docker и docker-compose установлены и работают
- [ ] docker-compose.yml содержит все необходимые сервисы
- [ ] PostgreSQL запускается с healthcheck и init-скриптами
- [ ] Приложение стартует после готовности БД (depends_on + condition)
- [ ] Selenoid настроен и запускает браузеры в контейнерах
- [ ] browsers.json содержит конфигурацию Chrome и Firefox
- [ ] Dockerfile.tests создан для тестового runner-а
- [ ] config-docker.properties содержит адреса сервисов Docker
- [ ] Тесты запускаются внутри Docker и проходят
- [ ] `docker-compose down -v` полностью очищает окружение
