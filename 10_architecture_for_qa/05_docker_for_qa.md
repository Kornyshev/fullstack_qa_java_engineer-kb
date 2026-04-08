# Docker для QA

## Обзор

Docker — это платформа контейнеризации, которая позволяет упаковывать приложения со всеми
зависимостями в изолированные контейнеры. Для QA-инженера Docker решает главную проблему:
«у меня на машине работает, а у вас нет». С Docker тестовое окружение идентично
production-окружению, легко воспроизводимо и может быть поднято одной командой.
Docker стал стандартом в индустрии, и умение им пользоваться — обязательное требование
для современного QA-инженера.

---

## Containers vs VMs

### Виртуальные машины (VMs)

```
┌─────────────────────────────────────┐
│          Host OS (Linux/macOS)      │
├─────────────────────────────────────┤
│            Hypervisor               │
├───────────┬───────────┬─────────────┤
│   VM 1    │   VM 2    │    VM 3     │
│ ┌───────┐ │ ┌───────┐ │ ┌─────────┐ │
│ │Guest  │ │ │Guest  │ │ │ Guest   │ │
│ │OS     │ │ │OS     │ │ │ OS      │ │
│ │(Linux)│ │ │(Win)  │ │ │ (Linux) │ │
│ ├───────┤ │ ├───────┤ │ ├─────────┤ │
│ │ App 1 │ │ │ App 2 │ │ │  App 3  │ │
│ └───────┘ │ └───────┘ │ └─────────┘ │
└───────────┴───────────┴─────────────┘
```

- Полная ОС внутри каждой VM
- Занимает гигабайты памяти
- Запуск: минуты
- Полная изоляция (гипервизор)

### Контейнеры (Docker)

```
┌─────────────────────────────────────┐
│          Host OS (Linux)            │
├─────────────────────────────────────┤
│          Docker Engine              │
├───────────┬───────────┬─────────────┤
│Container 1│Container 2│ Container 3 │
│ ┌───────┐ │ ┌───────┐ │ ┌─────────┐ │
│ │ App 1 │ │ │ App 2 │ │ │  App 3  │ │
│ │+ deps │ │ │+ deps │ │ │ + deps  │ │
│ └───────┘ │ └───────┘ │ └─────────┘ │
└───────────┴───────────┴─────────────┘
```

- Используют ядро host OS
- Занимают мегабайты
- Запуск: секунды
- Изоляция на уровне процессов (namespaces, cgroups)

### Сравнение

| Аспект | VM | Container |
|--------|-----|-----------|
| Размер | Гигабайты | Мегабайты |
| Время запуска | Минуты | Секунды |
| Изоляция | Полная (гипервизор) | Уровень процессов |
| ОС | Своя для каждой VM | Общее ядро host |
| Накладные расходы | Высокие | Минимальные |
| Плотность | Единицы на хост | Десятки/сотни на хост |

---

## Основные концепции Docker

### Image (образ)

Image — неизменяемый шаблон для создания контейнеров. Содержит ОС, зависимости и приложение.
Состоит из слоёв (layers) — каждая инструкция в Dockerfile создаёт новый слой.

```
┌──────────────────────┐
│   Application JAR    │  ← Layer 4 (COPY)
├──────────────────────┤
│   Java 17 (JDK)      │  ← Layer 3 (RUN apt install)
├──────────────────────┤
│   System packages     │  ← Layer 2 (RUN apt update)
├──────────────────────┤
│   Ubuntu 22.04 base   │  ← Layer 1 (FROM)
└──────────────────────┘
```

### Container (контейнер)

Container — запущенный экземпляр image. Можно создать много контейнеров из одного image.
Аналогия: image — это класс, container — это объект (инстанс).

### Registry

Registry — хранилище images. Docker Hub — публичный registry. Также существуют приватные
(AWS ECR, Google GCR, GitLab Container Registry).

---

## Dockerfile

Dockerfile — текстовый файл с инструкциями для сборки Docker image.

### Основные инструкции

```dockerfile
# FROM — базовый образ (отправная точка)
FROM eclipse-temurin:17-jdk-alpine

# LABEL — метаданные
LABEL maintainer="qa-team@example.com"

# WORKDIR — рабочая директория внутри контейнера
WORKDIR /app

# COPY — копирование файлов с хоста в контейнер
COPY target/app.jar /app/app.jar
COPY config/ /app/config/

# ADD — как COPY, но умеет распаковывать архивы и скачивать по URL
ADD https://example.com/config.tar.gz /app/

# RUN — выполнение команды при сборке образа
RUN apt-get update && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*

# ENV — переменные окружения
ENV SPRING_PROFILES_ACTIVE=prod
ENV JAVA_OPTS="-Xmx512m"

# EXPOSE — документирует порт (не открывает его!)
EXPOSE 8080

# CMD — команда по умолчанию при запуске контейнера
CMD ["java", "-jar", "/app/app.jar"]

# ENTRYPOINT — фиксированная точка входа (CMD становится аргументами)
ENTRYPOINT ["java", "-jar"]
CMD ["/app/app.jar"]
```

### Пример: Dockerfile для Spring Boot приложения

```dockerfile
# Многоступенчатая сборка (multi-stage build)

# Этап 1: Сборка приложения
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /build
COPY pom.xml .
# Скачиваем зависимости отдельно (кэширование слоёв)
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

# Этап 2: Финальный образ (только JRE, без Maven и исходников)
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
# Копируем собранный JAR из этапа builder
COPY --from=builder /build/target/*.jar app.jar
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
```

**Преимущества multi-stage build:**
- Финальный образ меньше (нет Maven, исходников, тестов)
- Безопаснее (меньше инструментов для потенциального злоумышленника)

### Пример: Dockerfile для тестового окружения

```dockerfile
# Образ для запуска автотестов
FROM maven:3.9-eclipse-temurin-17

WORKDIR /tests

# Устанавливаем утилиты для отладки
RUN apt-get update && apt-get install -y \
    curl \
    jq \
    netcat-openbsd \
    && rm -rf /var/lib/apt/lists/*

# Копируем описание зависимостей и скачиваем их
COPY pom.xml .
RUN mvn dependency:go-offline

# Копируем тесты
COPY src/test ./src/test

# Точка входа — запуск тестов
ENTRYPOINT ["mvn", "test"]
# По умолчанию запускаем все тесты; можно переопределить при запуске
CMD ["-Dtest=**/*Test"]
```

---

## Docker Commands

### Основные команды

```bash
# === Работа с образами ===

# Собрать образ из Dockerfile
docker build -t my-app:1.0 .

# Скачать образ из registry
docker pull postgres:15

# Список локальных образов
docker images

# Удалить образ
docker rmi my-app:1.0

# === Работа с контейнерами ===

# Запустить контейнер
docker run -d --name my-postgres \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=secret \
  postgres:15

# Флаги:
#   -d            — запуск в фоновом режиме (detached)
#   --name        — имя контейнера
#   -p 5432:5432  — маппинг портов (хост:контейнер)
#   -e            — переменная окружения
#   -v            — маппинг volume (хост:контейнер)
#   --rm          — удалить контейнер после остановки
#   --network     — подключить к сети

# Список запущенных контейнеров
docker ps

# Список всех контейнеров (включая остановленные)
docker ps -a

# Логи контейнера
docker logs my-postgres
docker logs -f my-postgres          # Follow — как tail -f
docker logs --tail 100 my-postgres  # Последние 100 строк

# Выполнить команду внутри запущенного контейнера
docker exec -it my-postgres bash
docker exec my-postgres psql -U postgres -c "SELECT 1"

# Остановить контейнер
docker stop my-postgres

# Запустить остановленный контейнер
docker start my-postgres

# Удалить контейнер
docker rm my-postgres

# Остановить и удалить все контейнеры
docker stop $(docker ps -q) && docker rm $(docker ps -aq)
```

### Полезные команды для QA

```bash
# Посмотреть IP-адрес контейнера
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' my-postgres

# Скопировать файл из контейнера на хост
docker cp my-app:/app/logs/app.log ./app.log

# Скопировать файл с хоста в контейнер
docker cp test-data.sql my-postgres:/tmp/test-data.sql

# Посмотреть потребление ресурсов
docker stats

# Посмотреть порты контейнера
docker port my-app

# Очистить неиспользуемые ресурсы (images, containers, volumes, networks)
docker system prune -a
```

---

## Docker Compose

Docker Compose — инструмент для определения и запуска многоконтейнерных приложений.
Конфигурация описывается в файле `docker-compose.yml`.

### Структура docker-compose.yml

```yaml
version: '3.8'

services:
  # === Приложение ===
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://db:5432/myapp
      - SPRING_DATASOURCE_USERNAME=postgres
      - SPRING_DATASOURCE_PASSWORD=secret
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app-network
    restart: unless-stopped

  # === База данных ===
  db:
    image: postgres:15
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  # === Redis (кэш) ===
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  postgres-data:
```

### Ключевые секции

**services** — определение контейнеров:
- `image` — готовый образ из registry
- `build` — сборка из Dockerfile
- `ports` — маппинг портов `"хост:контейнер"`
- `environment` — переменные окружения
- `volumes` — маппинг директорий и named volumes
- `depends_on` — порядок запуска (с `condition: service_healthy` — ждать healthcheck)
- `restart` — политика перезапуска (`no`, `always`, `unless-stopped`, `on-failure`)
- `healthcheck` — проверка готовности сервиса

**networks** — сети для изоляции и коммуникации между контейнерами:
- Контейнеры в одной сети видят друг друга по имени сервиса (DNS)
- `db` — hostname для PostgreSQL внутри сети

**volumes** — персистентное хранение данных:
- Named volumes (`postgres-data`) — управляются Docker
- Bind mounts (`./init-scripts:/path`) — конкретная директория хоста

### Команды Docker Compose

```bash
# Запустить все сервисы (в фоне)
docker compose up -d

# Запустить с пересборкой образов
docker compose up -d --build

# Остановить все сервисы
docker compose down

# Остановить и удалить volumes (осторожно — удаляет данные!)
docker compose down -v

# Логи всех сервисов
docker compose logs

# Логи конкретного сервиса (follow mode)
docker compose logs -f app

# Состояние сервисов
docker compose ps

# Масштабирование сервиса (запуск нескольких инстансов)
docker compose up -d --scale app=3

# Выполнить команду в сервисе
docker compose exec db psql -U postgres -d myapp

# Перезапустить один сервис
docker compose restart app
```

---

## Тестовое окружение с Docker Compose

### Полноценная среда: App + DB + Selenoid

```yaml
# docker-compose.test.yml — среда для E2E-тестов
version: '3.8'

services:
  # Тестируемое приложение
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=test
      - SPRING_DATASOURCE_URL=jdbc:postgresql://db:5432/testdb
      - SPRING_DATASOURCE_USERNAME=postgres
      - SPRING_DATASOURCE_PASSWORD=test
    depends_on:
      db:
        condition: service_healthy
    networks:
      - test-network

  # PostgreSQL для тестов
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: test
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 10
    networks:
      - test-network

  # Selenoid — Selenium Grid в Docker
  selenoid:
    image: aerokube/selenoid:latest
    ports:
      - "4444:4444"
    volumes:
      - ./selenoid:/etc/selenoid
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - test-network

  # Selenoid UI — интерфейс для наблюдения за тестами
  selenoid-ui:
    image: aerokube/selenoid-ui:latest
    ports:
      - "8081:8080"
    depends_on:
      - selenoid
    command: ["--selenoid-uri", "http://selenoid:4444"]
    networks:
      - test-network

  # Allure Report сервер
  allure:
    image: frankescobar/allure-docker-service:latest
    ports:
      - "5050:5050"
    volumes:
      - ./allure-results:/app/allure-results
    networks:
      - test-network

networks:
  test-network:
    driver: bridge
```

### Скрипт запуска тестов

```bash
#!/bin/bash
# run-tests.sh — скрипт для запуска тестовой среды и тестов

set -e

echo "=== Поднимаем тестовое окружение ==="
docker compose -f docker-compose.test.yml up -d --build

echo "=== Ждём готовности приложения ==="
until curl -sf http://localhost:8080/actuator/health > /dev/null 2>&1; do
    echo "Ожидаем запуск приложения..."
    sleep 2
done
echo "Приложение готово!"

echo "=== Запускаем тесты ==="
mvn test -Dselenide.remote=http://localhost:4444/wd/hub \
         -Dselenide.baseUrl=http://app:8080 \
         -Dtest.db.url=jdbc:postgresql://localhost:5432/testdb

echo "=== Собираем результаты ==="
# Результаты тестов уже в ./allure-results (volume mount)

echo "=== Останавливаем окружение ==="
docker compose -f docker-compose.test.yml down

echo "=== Готово! Allure отчёт: http://localhost:5050 ==="
```

---

## Связь с тестированием

Docker является фундаментальным инструментом для QA:

1. **Воспроизводимость среды** — тестовое окружение идентично на любой машине
2. **Изоляция тестов** — каждый запуск в чистом окружении
3. **Testcontainers** — программный запуск контейнеров из тестов (БД, Kafka, Redis)
4. **CI/CD** — тесты запускаются в Docker-контейнерах в pipeline
5. **Параллелизм** — запуск нескольких тестовых окружений одновременно
6. **Selenoid** — Selenium Grid в Docker (браузеры в контейнерах)

---

## Типичные ошибки

1. **Не используют `.dockerignore`** — в образ попадают `node_modules`, `.git`, тестовые данные
2. **Один слой RUN для каждой команды** — раздутый образ, медленная сборка
3. **Запуск от root** — уязвимость безопасности (используйте `USER` в Dockerfile)
4. **Hardcoded пароли** — пароли прямо в docker-compose (используйте `.env` или secrets)
5. **Не используют healthcheck** — `depends_on` без condition не ждёт готовности сервиса
6. **Забывают про volumes** — данные теряются при `docker compose down`
7. **Не очищают ресурсы** — диск забивается неиспользуемыми образами и volumes
8. **Используют `latest` tag** — непредсказуемая версия образа, тесты могут сломаться

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Что такое Docker? В чём разница между контейнером и виртуальной машиной?
2. Что такое Docker image и Docker container?
3. Какие основные инструкции Dockerfile вы знаете?
4. Как запустить контейнер с PostgreSQL?
5. Как посмотреть логи контейнера?

### 🟡 Средний уровень
6. В чём разница между `CMD` и `ENTRYPOINT`?
7. Что такое multi-stage build и зачем он нужен?
8. Как работает docker-compose? Для чего используются `networks` и `volumes`?
9. Как организовать тестовое окружение с Docker Compose (приложение + БД)?
10. Что такое healthcheck и зачем он нужен в `depends_on`?
11. Как передать переменные окружения в контейнер?

### 🔴 Продвинутый уровень
12. Как оптимизировать размер Docker image?
13. Как организовать параллельный запуск тестов в Docker?
14. Как настроить Selenoid в Docker Compose для E2E тестов?
15. Как работает networking в Docker? В чём разница bridge, host и overlay networks?
16. Как интегрировать Docker с CI/CD pipeline для автотестов?

---

## Практические задания

### Задание 1: Первый Dockerfile
Напишите Dockerfile для простого Java-приложения (или Spring Boot). Соберите образ
и запустите контейнер. Проверьте, что приложение доступно.

### Задание 2: docker-compose с БД
Создайте `docker-compose.yml` с:
- PostgreSQL (с healthcheck)
- pgAdmin (для визуального управления БД)
Подключитесь к БД через pgAdmin.

### Задание 3: Полная тестовая среда
Создайте docker-compose с:
- Backend-приложение (Spring Boot)
- PostgreSQL
- Selenoid (для UI-тестов)
Запустите E2E-тесты через эту среду.

### Задание 4: Отладка контейнера
1. Запустите контейнер с приложением, которое падает при старте
2. Используя `docker logs`, `docker exec`, `docker inspect` найдите причину падения
3. Исправьте проблему и перезапустите

---

## Дополнительные ресурсы

- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Dockerfile Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Testcontainers](https://testcontainers.com/)
- [Selenoid Documentation](https://aerokube.com/selenoid/latest/)
- [Docker для начинающих (YouTube, Артём Шумейко)](https://www.youtube.com/watch?v=_uZQtRyF6Eg)
