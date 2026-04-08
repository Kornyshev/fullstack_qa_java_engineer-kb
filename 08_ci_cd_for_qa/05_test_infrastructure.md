# Тестовая инфраструктура

## Обзор

Тестовая инфраструктура — это совокупность инструментов, сервисов и окружений, которые обеспечивают
запуск автоматизированных тестов. Для QA-инженера умение настраивать и поддерживать тестовую инфраструктуру —
навык, который отличает просто автоматизатора от полноценного QA-инженера. Ключевые компоненты: Selenoid
(или Selenium Grid) для запуска браузеров, Docker для контейнеризации, TestContainers для интеграционного
тестирования, Docker Compose для оркестрации тестовых окружений. Всё это работает внутри CI/CD pipeline,
обеспечивая повторяемые и масштабируемые тестовые прогоны.

---

## Selenoid

### Что такое Selenoid

Selenoid — это легковесная альтернатива Selenium Grid, написанная на Go. Selenoid запускает каждую
сессию браузера в отдельном Docker-контейнере, что обеспечивает полную изоляцию, стабильность и простоту
управления версиями браузеров.

### Архитектура Selenoid

```
┌──────────────────────────────────────────────────────────────────┐
│                     Тестовый Framework                           │
│              (Selenide / Selenium WebDriver)                     │
│                                                                  │
│         RemoteWebDriver → http://selenoid:4444/wd/hub           │
└─────────────────────────────┬────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│                        SELENOID                                  │
│                   (Go-приложение)                                │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Session Manager                              │   │
│  │  - Принимает запросы на создание сессии                   │   │
│  │  - Запускает Docker-контейнер с нужным браузером         │   │
│  │  - Управляет лимитами (max sessions)                     │   │
│  │  - Записывает видео (опционально)                        │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  Активные сессии:                                               │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐   │
│  │ Chrome    │  │ Chrome    │  │ Firefox   │  │ Edge      │   │
│  │ 125.0     │  │ 125.0     │  │ 126.0     │  │ 125.0     │   │
│  │ container │  │ container │  │ container │  │ container │   │
│  │ (сессия 1)│  │ (сессия 2)│  │ (сессия 3)│  │ (сессия 4)│   │
│  └───────────┘  └───────────┘  └───────────┘  └───────────┘   │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│                    SELENOID UI                                   │
│              http://localhost:8080                                │
│                                                                  │
│  - Визуальное наблюдение за сессиями в реальном времени (VNC)   │
│  - Статистика: количество активных сессий, очередь              │
│  - Просмотр видеозаписей выполнения тестов                      │
│  - Логи браузерных сессий                                       │
└──────────────────────────────────────────────────────────────────┘
```

### Настройка Selenoid через Docker Compose

**docker-compose.yml:**

```yaml
version: '3.8'

services:
  selenoid:
    image: aerokube/selenoid:latest
    container_name: selenoid
    ports:
      - "4444:4444"
    volumes:
      # Конфигурация браузеров
      - ./config/browsers.json:/etc/selenoid/browsers.json:ro
      # Docker socket для запуска контейнеров с браузерами
      - /var/run/docker.sock:/var/run/docker.sock
      # Директория для видеозаписей
      - ./video:/opt/selenoid/video
      # Логи
      - ./logs:/opt/selenoid/logs
    environment:
      - OVERRIDE_VIDEO_OUTPUT_DIR=${PWD}/video
    command:
      - "-conf"
      - "/etc/selenoid/browsers.json"
      - "-video-output-dir"
      - "/opt/selenoid/video"
      - "-log-output-dir"
      - "/opt/selenoid/logs"
      - "-limit"                         # Максимальное количество одновременных сессий
      - "10"
      - "-timeout"                       # Таймаут неактивной сессии
      - "3m"
      - "-service-startup-timeout"       # Таймаут запуска контейнера
      - "2m"
    networks:
      - selenoid

  selenoid-ui:
    image: aerokube/selenoid-ui:latest
    container_name: selenoid-ui
    ports:
      - "8080:8080"
    command:
      - "--selenoid-uri"
      - "http://selenoid:4444"
    depends_on:
      - selenoid
    networks:
      - selenoid

networks:
  selenoid:
    driver: bridge
```

### browsers.json — конфигурация браузеров

```json
{
  "chrome": {
    "default": "125.0",
    "versions": {
      "125.0": {
        "image": "selenoid/vnc_chrome:125.0",
        "port": "4444",
        "path": "/",
        "env": ["LANG=ru_RU.UTF-8", "LANGUAGE=ru:en", "TZ=Europe/Moscow"],
        "tmpfs": {"/tmp": "size=512m"},
        "shmSize": 2147483648
      },
      "124.0": {
        "image": "selenoid/vnc_chrome:124.0",
        "port": "4444",
        "path": "/"
      }
    }
  },
  "firefox": {
    "default": "126.0",
    "versions": {
      "126.0": {
        "image": "selenoid/vnc_firefox:126.0",
        "port": "4444",
        "path": "/wd/hub"
      }
    }
  },
  "edge": {
    "default": "125.0",
    "versions": {
      "125.0": {
        "image": "browsers/edge:125.0",
        "port": "4444",
        "path": "/"
      }
    }
  }
}
```

**Важные параметры browsers.json:**
- `image` — Docker-образ с браузером (образы `selenoid/vnc_*` содержат VNC для наблюдения)
- `port` — порт WebDriver внутри контейнера
- `path` — путь к WebDriver (для Chrome — `/`, для Firefox — `/wd/hub`)
- `tmpfs` — временная файловая система в памяти (ускоряет работу)
- `shmSize` — размер shared memory (для Chrome рекомендуется >= 2 GB)

### Selenoid UI

Selenoid UI доступен по адресу `http://localhost:8080` и предоставляет:

- **Capabilities:** визуальная форма для создания capabilities (удобно для отладки)
- **Sessions:** список активных сессий с возможностью наблюдения через VNC
- **Video:** просмотр записанных видео выполнения тестов
- **Logs:** логи браузерных сессий
- **Statistics:** графики загрузки (количество сессий во времени)

### GGR (Go Grid Router) — масштабирование

Для больших команд одного Selenoid недостаточно. GGR — это балансировщик нагрузки, который распределяет
запросы между несколькими инстансами Selenoid.

```
┌───────────────────────────────────┐
│          Тесты (100 потоков)       │
└───────────────┬───────────────────┘
                │
                ▼
┌───────────────────────────────────┐
│          GGR (Load Balancer)       │
│      http://ggr:4444/wd/hub       │
└──────┬────────┬────────┬──────────┘
       │        │        │
       ▼        ▼        ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│ Selenoid │ │ Selenoid │ │ Selenoid │
│ Host 1   │ │ Host 2   │ │ Host 3   │
│ (10 сесс)│ │ (10 сесс)│ │ (10 сесс)│
└──────────┘ └──────────┘ └──────────┘
```

**Конфигурация GGR (quota/test.xml):**

```xml
<qa:browsers xmlns:qa="urn:config.gridrouter.qatools.ru">
    <browser name="chrome" defaultVersion="125.0">
        <version number="125.0">
            <region name="region-1">
                <host name="selenoid-host-1.local" port="4444" count="10"/>
                <host name="selenoid-host-2.local" port="4444" count="10"/>
            </region>
        </version>
    </browser>
</qa:browsers>
```

---

## Docker Selenium Grid

### Selenium Grid 4

Selenium Grid — это официальный инструмент от проекта Selenium для распределённого запуска тестов.
В версии 4 архитектура была полностью переписана с использованием компонентного подхода.

### Архитектура Selenium Grid 4

```
┌──────────────────────────────────────────────────────────────┐
│                    Selenium Grid 4                            │
│                                                              │
│  ┌──────────┐  ┌───────────┐  ┌───────────┐  ┌──────────┐  │
│  │  Router  │──│ Distributor│──│  Session   │──│  Event   │  │
│  │          │  │           │  │   Map      │  │   Bus    │  │
│  └──────────┘  └───────────┘  └───────────┘  └──────────┘  │
│       │                                                      │
│       ▼                                                      │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                      Nodes                            │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐     │   │
│  │  │ Chrome Node│  │Firefox Node│  │ Edge Node  │     │   │
│  │  │ (Docker)   │  │ (Docker)   │  │ (Docker)   │     │   │
│  │  └────────────┘  └────────────┘  └────────────┘     │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

### Docker Compose для Selenium Grid

```yaml
version: '3.8'

services:
  # Hub (Router + Distributor + Session Map + Event Bus)
  selenium-hub:
    image: selenium/hub:4.20
    container_name: selenium-hub
    ports:
      - "4442:4442"              # Event Bus (publish)
      - "4443:4443"              # Event Bus (subscribe)
      - "4444:4444"              # Router (основной URL)
    environment:
      - SE_NODE_MAX_SESSIONS=5
      - SE_SESSION_REQUEST_TIMEOUT=300
      - SE_SESSION_RETRY_INTERVAL=2

  # Chrome Node
  chrome-node:
    image: selenium/node-chrome:125.0
    container_name: chrome-node
    shm_size: '2gb'
    depends_on:
      - selenium-hub
    environment:
      - SE_EVENT_BUS_HOST=selenium-hub
      - SE_EVENT_BUS_PUBLISH_PORT=4442
      - SE_EVENT_BUS_SUBSCRIBE_PORT=4443
      - SE_NODE_MAX_SESSIONS=5
      - SE_NODE_OVERRIDE_MAX_SESSIONS=true
      - SE_VNC_NO_PASSWORD=1
    ports:
      - "7900:7900"              # noVNC — наблюдение через браузер

  # Firefox Node
  firefox-node:
    image: selenium/node-firefox:126.0
    container_name: firefox-node
    shm_size: '2gb'
    depends_on:
      - selenium-hub
    environment:
      - SE_EVENT_BUS_HOST=selenium-hub
      - SE_EVENT_BUS_PUBLISH_PORT=4442
      - SE_EVENT_BUS_SUBSCRIBE_PORT=4443
      - SE_NODE_MAX_SESSIONS=5
      - SE_NODE_OVERRIDE_MAX_SESSIONS=true

  # Edge Node (опционально)
  edge-node:
    image: selenium/node-edge:125.0
    container_name: edge-node
    shm_size: '2gb'
    depends_on:
      - selenium-hub
    environment:
      - SE_EVENT_BUS_HOST=selenium-hub
      - SE_EVENT_BUS_PUBLISH_PORT=4442
      - SE_EVENT_BUS_SUBSCRIBE_PORT=4443
      - SE_NODE_MAX_SESSIONS=3
```

### Selenoid vs Selenium Grid — сравнение

| Аспект | Selenoid | Selenium Grid 4 |
|--------|----------|-----------------|
| Язык | Go | Java |
| Ресурсы | Минимальные (~50 MB RAM) | Высокие (~512 MB+ RAM на Hub) |
| Изоляция | Контейнер на каждую сессию | Контейнер на Node (несколько сессий) |
| Видеозапись | Встроенная | Требует дополнительной настройки |
| UI мониторинг | Selenoid UI | Grid Console |
| Масштабирование | GGR | Kubernetes / Docker Swarm |
| Простота настройки | Очень простая | Средняя |
| Поддержка мобильных | Через Appium | Через Appium |

---

## TestContainers в CI

### Что такое TestContainers

TestContainers — это Java-библиотека, которая позволяет запускать Docker-контейнеры прямо из тестового
кода. Идеально подходит для интеграционных тестов, где нужна реальная БД, message broker или другой сервис.

### Примеры использования

```java
// Контейнер с PostgreSQL для интеграционных тестов
@Testcontainers
class DatabaseIntegrationTest {

    // Автоматический запуск контейнера перед тестами, остановка после
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
            .withDatabaseName("testdb")
            .withUsername("testuser")
            .withPassword("testpass")
            .withInitScript("schema.sql");  // SQL-скрипт инициализации

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        // Динамическая настройка Spring Boot на использование контейнера
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Test
    void shouldSaveAndRetrieveUser() {
        // Тест использует реальную PostgreSQL в контейнере
        // ...
    }
}
```

```java
// Контейнер с Redis
@Container
static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);

// Контейнер с Kafka
@Container
static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));

// Контейнер с Selenoid для UI-тестов
@Container
static BrowserWebDriverContainer<?> chrome = new BrowserWebDriverContainer<>()
        .withCapabilities(new ChromeOptions())
        .withRecordingMode(RECORD_ALL, new File("target/video"));
```

### TestContainers в CI/CD — Docker-in-Docker

В CI/CD pipeline TestContainers требует доступ к Docker daemon. Есть несколько подходов.

**Подход 1: Docker Socket Mounting (рекомендуемый)**

```yaml
# GitLab CI
integration-tests:
  image: maven:3.9-eclipse-temurin-17
  variables:
    DOCKER_HOST: "unix:///var/run/docker.sock"
    TESTCONTAINERS_RYUK_DISABLED: "true"  # Отключить Ryuk в CI (он управляет очисткой)
  script:
    - mvn test -Dgroups=integration
  tags:
    - docker-socket  # Runner с примонтированным Docker socket
```

```yaml
# GitHub Actions
integration-tests:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin'
    # Docker уже доступен на ubuntu-latest runner
    - name: Run Integration Tests
      run: mvn test -Dgroups=integration
```

**Подход 2: Docker-in-Docker (DinD)**

```yaml
# GitLab CI с DinD
integration-tests:
  image: maven:3.9-eclipse-temurin-17
  services:
    - name: docker:dind
      alias: docker
  variables:
    DOCKER_HOST: "tcp://docker:2375"
    DOCKER_TLS_CERTDIR: ""
    TESTCONTAINERS_HOST_OVERRIDE: "docker"
  script:
    - mvn test -Dgroups=integration
```

---

## Управление тестовыми окружениями

### Docker Compose для Full-Stack окружения

Docker Compose позволяет развернуть всё приложение (backend, frontend, БД, очереди) на одной машине
для E2E-тестирования.

```yaml
version: '3.8'

services:
  # ==========================================
  # Приложение (Backend)
  # ==========================================
  backend:
    image: company/backend:${APP_VERSION:-latest}
    container_name: app-backend
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=test
      - DB_URL=jdbc:postgresql://db:5432/appdb
      - DB_USER=appuser
      - DB_PASSWORD=apppass
      - REDIS_HOST=redis
      - KAFKA_BOOTSTRAP_SERVERS=kafka:9092
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 10s
      timeout: 5s
      retries: 10
    networks:
      - app-network

  # ==========================================
  # Приложение (Frontend)
  # ==========================================
  frontend:
    image: company/frontend:${APP_VERSION:-latest}
    container_name: app-frontend
    ports:
      - "3000:80"
    environment:
      - API_URL=http://backend:8080
    depends_on:
      - backend
    networks:
      - app-network

  # ==========================================
  # База данных
  # ==========================================
  db:
    image: postgres:15
    container_name: app-db
    environment:
      - POSTGRES_DB=appdb
      - POSTGRES_USER=appuser
      - POSTGRES_PASSWORD=apppass
    volumes:
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 5s
      timeout: 3s
      retries: 5
    networks:
      - app-network

  # ==========================================
  # Redis (кэш)
  # ==========================================
  redis:
    image: redis:7-alpine
    container_name: app-redis
    ports:
      - "6379:6379"
    networks:
      - app-network

  # ==========================================
  # Selenoid (для UI-тестов)
  # ==========================================
  selenoid:
    image: aerokube/selenoid:latest
    container_name: selenoid
    ports:
      - "4444:4444"
    volumes:
      - ./config/browsers.json:/etc/selenoid/browsers.json:ro
      - /var/run/docker.sock:/var/run/docker.sock
      - ./video:/opt/selenoid/video
    command:
      - "-conf"
      - "/etc/selenoid/browsers.json"
      - "-limit"
      - "5"
      - "-container-network"
      - "app-network"             # Браузеры в той же сети, что и приложение
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

### Скрипт для запуска тестов в CI

```bash
#!/bin/bash
# Скрипт запуска полного E2E-тестирования

set -e

echo "=== Запуск тестового окружения ==="
docker compose -f docker-compose.test.yml up -d

echo "=== Ожидание готовности приложения ==="
# Ожидание health check backend
for i in $(seq 1 60); do
    if curl -s http://localhost:8080/actuator/health | grep -q '"UP"'; then
        echo "Backend готов!"
        break
    fi
    if [ "$i" -eq 60 ]; then
        echo "Таймаут ожидания backend!"
        docker compose -f docker-compose.test.yml logs backend
        exit 1
    fi
    sleep 2
done

echo "=== Запуск тестов ==="
mvn test \
    -Dbase.url=http://localhost:3000 \
    -Dapi.url=http://localhost:8080 \
    -Dselenide.remote=http://localhost:4444/wd/hub \
    -Dmaven.test.failure.ignore=true

TEST_EXIT_CODE=$?

echo "=== Сохранение артефактов ==="
# Копирование видео и логов
mkdir -p test-artifacts
cp -r video/ test-artifacts/ 2>/dev/null || true
cp -r target/allure-results/ test-artifacts/ 2>/dev/null || true
cp -r target/screenshots/ test-artifacts/ 2>/dev/null || true

echo "=== Остановка тестового окружения ==="
docker compose -f docker-compose.test.yml down -v

exit $TEST_EXIT_CODE
```

---

## Infrastructure as Code для тестовых окружений

### Принципы IaC для QA

Infrastructure as Code (IaC) — подход, при котором конфигурация инфраструктуры хранится в виде кода
в репозитории и версионируется наравне с кодом приложения.

**Преимущества для QA:**
- **Воспроизводимость:** одинаковое окружение при каждом запуске
- **Версионирование:** откат к предыдущей конфигурации
- **Review:** конфигурация проходит code review, как и код тестов
- **Самодокументирование:** конфигурация описывает окружение

### Структура проекта

```
test-infrastructure/
├── docker-compose.yml              # Основное окружение
├── docker-compose.selenoid.yml     # Selenoid-конфигурация
├── docker-compose.monitoring.yml   # Мониторинг (Grafana, Prometheus)
├── config/
│   ├── browsers.json               # Конфигурация браузеров Selenoid
│   ├── ggr/
│   │   └── quota/                  # Конфигурация GGR
│   └── allure/
│       └── allure.yml              # Конфигурация Allure Server
├── scripts/
│   ├── setup.sh                    # Скрипт начальной настройки
│   ├── run-tests.sh                # Скрипт запуска тестов
│   ├── pull-browsers.sh            # Скрипт загрузки образов браузеров
│   └── cleanup.sh                  # Очистка
├── db/
│   ├── init.sql                    # Начальная схема БД
│   └── test-data.sql               # Тестовые данные
└── Makefile                        # Удобные команды
```

### Makefile для управления инфраструктурой

```makefile
# Makefile для управления тестовой инфраструктурой

.PHONY: up down test clean browsers

# Запуск полного окружения
up:
	docker compose -f docker-compose.yml \
	               -f docker-compose.selenoid.yml up -d
	@echo "Ожидание готовности..."
	@sleep 10
	@echo "Окружение запущено!"
	@echo "Приложение: http://localhost:3000"
	@echo "Selenoid UI: http://localhost:8080"

# Остановка
down:
	docker compose -f docker-compose.yml \
	               -f docker-compose.selenoid.yml down -v

# Запуск тестов
test:
	mvn test \
		-Dbase.url=http://localhost:3000 \
		-Dselenide.remote=http://localhost:4444/wd/hub \
		-Dmaven.test.failure.ignore=true

# Генерация Allure Report
report:
	allure serve target/allure-results

# Загрузка образов браузеров
browsers:
	docker pull selenoid/vnc_chrome:125.0
	docker pull selenoid/vnc_firefox:126.0

# Полная очистка
clean:
	docker compose down -v --remove-orphans
	docker system prune -f
	rm -rf video/ logs/ target/
```

---

## Связь с тестированием

Тестовая инфраструктура — это фундамент, на котором строится вся автоматизация тестирования:

- **Selenoid / Selenium Grid:** без них невозможны стабильные UI-тесты в CI/CD. Локальный браузер
  в CI-runner — анти-паттерн (нет UI, разные версии, нестабильность).
- **TestContainers:** позволяют писать интеграционные тесты без зависимости от внешних сервисов.
  Каждый запуск тестов получает чистую БД / Kafka / Redis.
- **Docker Compose:** обеспечивает запуск полного стека приложения для E2E-тестов. QA может
  воспроизвести любой баг, подняв точную копию окружения.
- **IaC:** конфигурация инфраструктуры версионируется и проходит review, как и тестовый код.

---

## Типичные ошибки

1. **Недостаточный `shm_size` для Chrome.** Chrome использует shared memory для рендеринга.
   Без `shm_size: 2gb` (или `tmpfs`) Chrome падает с ошибкой "session crashed". Это самая частая
   проблема при настройке Selenoid и Selenium Grid.

2. **Контейнеры браузеров не в одной сети с приложением.** Если Selenoid запускает контейнер
   с Chrome, а приложение работает в другой Docker-сети, браузер не может достучаться до приложения.
   Решение: `--container-network` в Selenoid.

3. **Нет health checks.** Тесты стартуют раньше, чем приложение готово принимать запросы.
   Решение: health checks в docker-compose + ожидание в скрипте запуска.

4. **Забывают скачать Docker-образы браузеров.** Selenoid не скачивает образы автоматически.
   Нужно заранее выполнить `docker pull` для всех версий из `browsers.json`.

5. **`TESTCONTAINERS_RYUK_DISABLED` не установлен в CI.** Ryuk — компонент TestContainers для
   очистки контейнеров. В CI может не иметь прав на управление Docker. Решение: отключить Ryuk в CI.

6. **Docker-in-Docker без привилегий.** TestContainers и Selenoid требуют доступа к Docker daemon.
   В CI нужно либо монтировать `/var/run/docker.sock`, либо использовать DinD с `--privileged`.

7. **Нет лимитов на ресурсы.** Без `--limit` Selenoid может запустить больше контейнеров, чем
   выдержит сервер, и всё зависнет. Всегда устанавливайте лимиты.

8. **Hardcoded-версии браузеров.** Версии браузеров в `browsers.json` не обновляются годами.
   Решение: регулярное обновление и автоматизация через скрипты.

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Что такое Selenoid? Чем он отличается от Selenium Grid?
2. Для чего нужен файл `browsers.json` в Selenoid?
3. Что такое TestContainers? Для каких тестов они используются?
4. Зачем нужен Docker Compose в контексте тестирования?
5. Что такое Selenoid UI? Какие возможности он предоставляет?

### 🟡 Средний уровень
6. Как настроить Selenoid в Docker Compose для CI/CD pipeline?
7. Как обеспечить, чтобы контейнеры браузеров имели доступ к тестируемому приложению?
8. Как использовать TestContainers для тестов с PostgreSQL? Приведите пример.
9. В чём разница между Docker Socket Mounting и Docker-in-Docker для TestContainers в CI?
10. Как организовать запись видео выполнения тестов в Selenoid?
11. Как масштабировать Selenoid для большой команды (GGR)?

### 🔴 Продвинутый уровень
12. Как спроектировать тестовую инфраструктуру для микросервисного приложения (10+ сервисов)?
13. Как организовать параллельный запуск 100+ UI-тестов без деградации стабильности?
14. Как реализовать динамическое создание тестовых окружений (ephemeral environments)?
15. Как мониторить тестовую инфраструктуру (Selenoid, Selenium Grid) в production?
16. Как организовать тестирование мобильных приложений в контейнерах (Appium + Docker)?

---

## Практические задания

### Задание 1: Настройка Selenoid
Развернуть Selenoid локально:
- Docker Compose с Selenoid и Selenoid UI
- `browsers.json` с Chrome и Firefox (по 2 версии)
- Запуск простого Selenide-теста через Selenoid
- Проверка видеозаписи

### Задание 2: TestContainers
Написать интеграционный тест с использованием TestContainers:
- PostgreSQL-контейнер с инициализацией схемы
- CRUD-операции через DAO/Repository
- Убедиться, что тест проходит в CI (GitHub Actions или GitLab CI)

### Задание 3: Full-Stack окружение
Создать Docker Compose для полного тестового окружения:
- Backend (Spring Boot приложение)
- Frontend (React/Angular)
- PostgreSQL + Redis
- Selenoid для UI-тестов
- Health checks для всех сервисов
- Скрипт для запуска E2E-тестов

### Задание 4: Масштабирование
Спроектировать тестовую инфраструктуру для команды из 20 QA-инженеров:
- Selenoid + GGR для распределённых UI-тестов
- Общий Allure Server для хранения отчётов
- Изолированные окружения для каждой feature-ветки
- Мониторинг загрузки инфраструктуры

### Задание 5: CI/CD Pipeline с инфраструктурой
Написать полный CI/CD pipeline (GitLab CI или GitHub Actions), который:
- Поднимает приложение через Docker Compose
- Ожидает готовности всех сервисов (health check)
- Запускает E2E-тесты через Selenoid
- Собирает артефакты (Allure, видео, скриншоты)
- Останавливает и очищает окружение

---

## Дополнительные ресурсы

- [Selenoid Documentation](https://aerokube.com/selenoid/latest/) — официальная документация Selenoid
- [Selenoid UI](https://aerokube.com/selenoid-ui/latest/) — Selenoid UI
- [GGR Documentation](https://aerokube.com/ggr/latest/) — Go Grid Router
- [TestContainers for Java](https://java.testcontainers.org/) — официальная документация TestContainers
- [Selenium Grid 4](https://www.selenium.dev/documentation/grid/) — документация Selenium Grid
- [Docker Compose Documentation](https://docs.docker.com/compose/) — Docker Compose
- [Moon (Cloud Selenoid)](https://aerokube.com/moon/latest/) — Kubernetes-native альтернатива Selenoid
