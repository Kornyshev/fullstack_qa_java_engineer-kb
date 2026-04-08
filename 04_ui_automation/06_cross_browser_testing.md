# Кросс-браузерное тестирование

## Обзор

Кросс-браузерное тестирование — это процесс проверки корректной работы веб-приложения в различных браузерах,
их версиях и на разных операционных системах. Даже при использовании веб-стандартов браузеры могут по-разному
интерпретировать HTML, CSS и JavaScript, что приводит к визуальным и функциональным расхождениям.

Для QA-инженера критически важно уметь настраивать инфраструктуру для параллельного запуска тестов в нескольких
браузерах, работать с Selenium Grid, Selenoid и облачными сервисами, а также интегрировать матричное
тестирование в CI/CD-пайплайн.

---

## Зачем нужно кросс-браузерное тестирование

### Ключевые причины

1. **Различия в рендеринге** — CSS Grid, Flexbox, шрифты могут отображаться по-разному
2. **Различия в JavaScript-движках** — V8 (Chrome), SpiderMonkey (Firefox), JavaScriptCore (Safari)
3. **Разная поддержка API** — Web APIs, Clipboard API, Payment Request API
4. **Пользовательская база** — клиенты используют разные браузеры, нельзя игнорировать ни один
5. **Регрессия при обновлениях** — новая версия браузера может сломать работающий функционал
6. **Требования заказчика/бизнеса** — SLA часто включает поддержку конкретных браузеров

### Типичные проблемы между браузерами

| Категория | Пример |
|---|---|
| CSS | Отображение `gap` в `flexbox` в старых Safari |
| JavaScript | Различия в `Date.parse()` между браузерами |
| Формы | Автозаполнение, валидация, стили `input[type=date]` |
| Медиа | Поддержка кодеков видео/аудио |
| Шрифты | Рендеринг шрифтов, сглаживание, `font-variant` |
| Скроллинг | `scroll-behavior: smooth`, инерция скролла |

---

## Selenium Grid

### Архитектура

Selenium Grid позволяет распределять выполнение тестов по нескольким машинам (нодам) с разными браузерами
и ОС. Начиная с Selenium 4, архитектура включает несколько компонентов.

**Компоненты Selenium Grid 4:**

| Компонент | Роль |
|---|---|
| **Router** | Точка входа, маршрутизирует запросы |
| **Distributor** | Распределяет запросы на создание сессий по нодам |
| **Session Map** | Хранит соответствие сессий и нодов |
| **Session Queue** | Очередь запросов, ожидающих свободную ноду |
| **Node** | Машина с установленным браузером, выполняет тесты |
| **Event Bus** | Внутренняя коммуникация между компонентами |

### Режимы запуска

```bash
# Standalone — все компоненты в одном процессе (для локальной разработки)
java -jar selenium-server-4.x.jar standalone

# Hub + Node — классическая схема (Hub = Router + Distributor + Session Map + Queue)
java -jar selenium-server-4.x.jar hub
java -jar selenium-server-4.x.jar node --hub http://hub-host:4444

# Полностью распределённый режим (каждый компонент отдельно)
java -jar selenium-server-4.x.jar sessions
java -jar selenium-server-4.x.jar distributor
java -jar selenium-server-4.x.jar router
java -jar selenium-server-4.x.jar node
```

### Подключение тестов к Grid

```java
// Подключение к удалённому Selenium Grid
ChromeOptions options = new ChromeOptions();
options.addArguments("--headless");

WebDriver driver = new RemoteWebDriver(
    new URL("http://grid-host:4444/wd/hub"),
    options
);
```

### Ограничения Selenium Grid

- Требует ручного управления инфраструктурой (установка браузеров, драйверов)
- Масштабирование требует добавления физических/виртуальных машин
- Нет изоляции между тестами (общий браузер на ноде)
- Сложная настройка для множества версий браузеров

---

## Selenoid

Selenoid — это легковесная альтернатива Selenium Grid, запускающая каждый браузер в изолированном
Docker-контейнере. Разработан командой Aerokube.

### Преимущества перед Selenium Grid

| Аспект | Selenium Grid | Selenoid |
|---|---|---|
| Изоляция | Общий процесс на ноде | Docker-контейнер на каждую сессию |
| Масштабирование | Ручное добавление нодов | Автоматическое создание контейнеров |
| Управление браузерами | Ручная установка | Docker-образы |
| Ресурсы | Браузеры постоянно запущены | Контейнеры создаются по запросу |
| Запись видео | Требует дополнительной настройки | Встроенная поддержка |
| VNC-доступ | Нет из коробки | Встроенный (для отладки) |
| UI-мониторинг | Grid Console | Selenoid UI |

### Конфигурация browsers.json

Файл `browsers.json` определяет доступные браузеры и их версии:

```json
{
  "chrome": {
    "default": "125.0",
    "versions": {
      "125.0": {
        "image": "selenoid/chrome:125.0",
        "port": "4444",
        "path": "/"
      },
      "124.0": {
        "image": "selenoid/chrome:124.0",
        "port": "4444",
        "path": "/"
      }
    }
  },
  "firefox": {
    "default": "126.0",
    "versions": {
      "126.0": {
        "image": "selenoid/firefox:126.0",
        "port": "4444",
        "path": "/wd/hub"
      }
    }
  },
  "opera": {
    "default": "109.0",
    "versions": {
      "109.0": {
        "image": "selenoid/opera:109.0",
        "port": "4444",
        "path": "/"
      }
    }
  }
}
```

### Настройка Selenoid с docker-compose

```yaml
# docker-compose.yml
version: "3"

services:
  selenoid:
    image: aerokube/selenoid:latest
    container_name: selenoid
    ports:
      - "4444:4444"
    volumes:
      # Конфигурация браузеров
      - "./config/browsers.json:/etc/selenoid/browsers.json"
      # Docker socket — для создания контейнеров с браузерами
      - "/var/run/docker.sock:/var/run/docker.sock"
      # Директория для записи видео
      - "./video:/opt/selenoid/video"
      # Директория для логов
      - "./logs:/opt/selenoid/logs"
    environment:
      - OVERRIDE_VIDEO_OUTPUT_DIR=${PWD}/video
    command:
      - "-conf"
      - "/etc/selenoid/browsers.json"
      - "-video-output-dir"
      - "/opt/selenoid/video"
      - "-log-output-dir"
      - "/opt/selenoid/logs"
      - "-limit"            # максимальное количество одновременных сессий
      - "10"
      - "-timeout"          # таймаут сессии
      - "3m"

  selenoid-ui:
    image: aerokube/selenoid-ui:latest
    container_name: selenoid-ui
    ports:
      - "8080:8080"
    depends_on:
      - selenoid
    command:
      - "--selenoid-uri"
      - "http://selenoid:4444"
```

### Запуск

```bash
# Предварительно загрузить образы браузеров
docker pull selenoid/chrome:125.0
docker pull selenoid/firefox:126.0
docker pull selenoid/opera:109.0

# Загрузить образ для записи видео
docker pull selenoid/video-recorder:latest-release

# Запустить Selenoid и UI
docker-compose up -d

# Проверить статус
curl http://localhost:4444/status
```

### Selenoid UI

Selenoid UI (доступен на порту 8080) предоставляет:

- **Статистику** — количество активных сессий, использование ресурсов
- **Live Preview** — просмотр экрана браузера в реальном времени через VNC
- **Логи** — просмотр логов текущих сессий
- **Видео** — доступ к записанным видео
- **Capabilities** — генератор JSON capabilities для тестов

### Подключение тестов к Selenoid

```java
public class SelenoidDriverFactory {

    private static final String SELENOID_URL = "http://localhost:4444/wd/hub";

    /**
     * Создание драйвера для Chrome с записью видео и VNC-доступом.
     */
    public static WebDriver createChromeDriver() {
        ChromeOptions options = new ChromeOptions();
        options.setCapability("selenoid:options", Map.of(
            "enableVideo", true,          // запись видео сессии
            "enableVNC", true,            // VNC-доступ для отладки
            "videoName", "test-video.mp4",
            "name", "My Test Session",    // имя сессии в Selenoid UI
            "sessionTimeout", "3m"        // таймаут неактивной сессии
        ));

        try {
            return new RemoteWebDriver(new URL(SELENOID_URL), options);
        } catch (MalformedURLException e) {
            throw new RuntimeException("Некорректный URL Selenoid", e);
        }
    }

    /**
     * Создание драйвера для Firefox.
     */
    public static WebDriver createFirefoxDriver() {
        FirefoxOptions options = new FirefoxOptions();
        options.setCapability("selenoid:options", Map.of(
            "enableVideo", true,
            "enableVNC", true
        ));

        try {
            return new RemoteWebDriver(new URL(SELENOID_URL), options);
        } catch (MalformedURLException e) {
            throw new RuntimeException("Некорректный URL Selenoid", e);
        }
    }
}
```

---

## Docker-контейнеры для браузеров

### Как это работает

1. Каждый браузер упакован в Docker-образ вместе с WebDriver и VNC-сервером
2. При запросе новой сессии Selenoid создаёт контейнер из соответствующего образа
3. Тест выполняется внутри контейнера — полная изоляция
4. После завершения сессии контейнер удаляется

### Преимущества контейнеризации

- **Изоляция** — каждый тест получает чистый браузер без кэша и cookies
- **Воспроизводимость** — одинаковое окружение на любой машине
- **Параллельность** — десятки контейнеров на одной машине
- **Версионирование** — точная версия браузера зафиксирована в образе
- **Простота очистки** — нет "мусора" после тестов

### Управление ресурсами

```yaml
# Ограничение ресурсов через Docker
services:
  selenoid:
    # ...
    command:
      - "-limit"
      - "10"            # не более 10 одновременных сессий
      - "-mem"
      - "512m"          # ограничение памяти на контейнер
      - "-cpu"
      - "1.0"           # ограничение CPU на контейнер
```

---

## Облачные сервисы

Для команд, не желающих поддерживать собственную инфраструктуру, существуют облачные платформы.

### Сравнение основных платформ

| Критерий | BrowserStack | SauceLabs | LambdaTest |
|---|---|---|---|
| Браузеры и ОС | 3000+ комбинаций | 2000+ комбинаций | 3000+ комбинаций |
| Мобильные устройства | Реальные устройства | Эмуляторы + реальные | Эмуляторы + реальные |
| Интеграция CI | Jenkins, GitHub Actions, и др. | Jenkins, GitHub Actions, и др. | Jenkins, GitHub Actions, и др. |
| Запись видео | Да | Да | Да |
| Live-тестирование | Да | Да | Да |
| Параллельные сессии | Зависит от тарифа | Зависит от тарифа | Зависит от тарифа |
| Геолокация | Да | Да | Да |
| Стоимость | От $29/мес. | От $49/мес. | От $15/мес. |

### Подключение к BrowserStack

```java
public class BrowserStackDriverFactory {

    /**
     * Создание драйвера для запуска тестов в BrowserStack.
     * Credentials хранятся в переменных окружения (не в коде!).
     */
    public static WebDriver createDriver() {
        String username = System.getenv("BROWSERSTACK_USERNAME");
        String accessKey = System.getenv("BROWSERSTACK_ACCESS_KEY");

        ChromeOptions options = new ChromeOptions();

        // BrowserStack-специфичные capabilities
        Map<String, Object> bstackOptions = new HashMap<>();
        bstackOptions.put("os", "Windows");
        bstackOptions.put("osVersion", "11");
        bstackOptions.put("browserVersion", "latest");
        bstackOptions.put("projectName", "My Project");
        bstackOptions.put("buildName", "Build #" + System.getenv("BUILD_NUMBER"));
        bstackOptions.put("sessionName", "Cross-browser Test");

        options.setCapability("bstack:options", bstackOptions);

        try {
            String url = String.format(
                "https://%s:%s@hub-cloud.browserstack.com/wd/hub",
                username, accessKey
            );
            return new RemoteWebDriver(new URL(url), options);
        } catch (MalformedURLException e) {
            throw new RuntimeException("Ошибка подключения к BrowserStack", e);
        }
    }
}
```

### Подключение к SauceLabs

```java
public class SauceLabsDriverFactory {

    public static WebDriver createDriver() {
        String username = System.getenv("SAUCE_USERNAME");
        String accessKey = System.getenv("SAUCE_ACCESS_KEY");

        ChromeOptions options = new ChromeOptions();

        Map<String, Object> sauceOptions = new HashMap<>();
        sauceOptions.put("build", "Build #" + System.getenv("BUILD_NUMBER"));
        sauceOptions.put("name", "Cross-browser Test");
        sauceOptions.put("screenResolution", "1920x1080");

        options.setCapability("sauce:options", sauceOptions);
        options.setPlatformName("Windows 11");
        options.setBrowserVersion("latest");

        try {
            String url = String.format(
                "https://%s:%s@ondemand.eu-central-1.saucelabs.com/wd/hub",
                username, accessKey
            );
            return new RemoteWebDriver(new URL(url), options);
        } catch (MalformedURLException e) {
            throw new RuntimeException("Ошибка подключения к SauceLabs", e);
        }
    }
}
```

---

## WebDriver Capabilities

Capabilities — это набор параметров, описывающих желаемую конфигурацию браузера.

### Основные capabilities

```java
// Chrome
ChromeOptions chromeOptions = new ChromeOptions();
chromeOptions.addArguments("--headless=new");        // безголовый режим
chromeOptions.addArguments("--window-size=1920,1080");
chromeOptions.addArguments("--disable-gpu");
chromeOptions.addArguments("--no-sandbox");
chromeOptions.addArguments("--disable-dev-shm-usage"); // важно для Docker
chromeOptions.addArguments("--incognito");

// Firefox
FirefoxOptions firefoxOptions = new FirefoxOptions();
firefoxOptions.addArguments("-headless");
firefoxOptions.addPreference("browser.download.folderList", 2);
firefoxOptions.addPreference("browser.download.dir", "/tmp/downloads");

// Edge
EdgeOptions edgeOptions = new EdgeOptions();
edgeOptions.addArguments("--headless=new");
edgeOptions.addArguments("--window-size=1920,1080");
```

### Фабрика драйверов для кросс-браузерного тестирования

```java
/**
 * Фабрика для создания WebDriver на основе параметра браузера.
 * Значение браузера передаётся через системное свойство или переменную окружения.
 */
public class DriverFactory {

    public static WebDriver createDriver(String browser) {
        return switch (browser.toLowerCase()) {
            case "chrome" -> createChromeDriver();
            case "firefox" -> createFirefoxDriver();
            case "edge" -> createEdgeDriver();
            default -> throw new IllegalArgumentException(
                "Неподдерживаемый браузер: " + browser
            );
        };
    }

    private static WebDriver createChromeDriver() {
        ChromeOptions options = new ChromeOptions();
        options.addArguments("--window-size=1920,1080");
        return new ChromeDriver(options);
    }

    private static WebDriver createFirefoxDriver() {
        FirefoxOptions options = new FirefoxOptions();
        return new FirefoxDriver(options);
    }

    private static WebDriver createEdgeDriver() {
        EdgeOptions options = new EdgeOptions();
        options.addArguments("--window-size=1920,1080");
        return new EdgeDriver(options);
    }
}
```

---

## Матричное тестирование в CI

### GitHub Actions — матрица браузеров

```yaml
# .github/workflows/cross-browser.yml
name: Cross-Browser Tests

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest

    strategy:
      fail-fast: false               # не останавливать остальные при провале одного
      matrix:
        browser: [ chrome, firefox, edge ]

    services:
      selenoid:
        image: aerokube/selenoid:latest
        ports:
          - 4444:4444
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Run tests on ${{ matrix.browser }}
        run: |
          mvn test -Dbrowser=${{ matrix.browser }} -Dselenoid.url=http://localhost:4444/wd/hub

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results-${{ matrix.browser }}
          path: target/surefire-reports/
```

### Jenkins Pipeline — параллельный запуск

```groovy
pipeline {
    agent any

    stages {
        stage('Cross-Browser Tests') {
            parallel {
                stage('Chrome') {
                    steps {
                        sh 'mvn test -Dbrowser=chrome -Dselenoid.url=http://selenoid:4444/wd/hub'
                    }
                }
                stage('Firefox') {
                    steps {
                        sh 'mvn test -Dbrowser=firefox -Dselenoid.url=http://selenoid:4444/wd/hub'
                    }
                }
                stage('Edge') {
                    steps {
                        sh 'mvn test -Dbrowser=edge -Dselenoid.url=http://selenoid:4444/wd/hub'
                    }
                }
            }
        }
    }

    post {
        always {
            // Сбор результатов из всех параллельных веток
            junit '**/target/surefire-reports/*.xml'
            archiveArtifacts artifacts: 'video/**', allowEmptyArchive: true
        }
    }
}
```

---

## Связь с тестированием

- **Тест-план** — кросс-браузерное тестирование должно быть частью тест-стратегии проекта
- **Приоритизация браузеров** — анализ аналитики (Google Analytics и т.д.) для выбора браузеров
- **Регрессионное тестирование** — кросс-браузерные тесты включаются в регрессионный набор
- **Визуальное тестирование** — скриншот-сравнение между браузерами (Percy, Applitools)
- **Performance** — время рендеринга может различаться между браузерами
- **Отчётность** — результаты тестов должны группироваться по браузерам для удобного анализа
- **Flaky-тесты** — нестабильные тесты часто проявляются только в конкретном браузере

---

## Типичные ошибки

1. **Тестирование только в Chrome** — самая распространённая ошибка, упускаются проблемы в Firefox/Safari
2. **Хардкод URL Selenoid/Grid** — URL инфраструктуры должен быть конфигурируемым
3. **Игнорирование `--no-sandbox` и `--disable-dev-shm-usage` в Docker** — тесты падают из-за нехватки ресурсов
4. **Отсутствие `fail-fast: false` в матрице CI** — падение в одном браузере останавливает все остальные
5. **Хранение credentials облачных сервисов в коде** — используйте переменные окружения и секреты CI
6. **Не указание размера окна** — тесты зависят от дефолтного размера, который отличается между средами
7. **Игнорирование таймаутов сессий** — "зависшие" сессии в Selenoid занимают ресурсы
8. **Отсутствие записи видео** — без видео сложно воспроизвести кросс-браузерную проблему

---

## Вопросы на интервью

### Уровень Junior

- Зачем нужно кросс-браузерное тестирование?
- Какие браузеры вы тестировали? Какие различия встречали?
- Что такое Selenium Grid? Для чего он используется?
- Что такое `RemoteWebDriver`? Чем отличается от `ChromeDriver`?
- Что такое capabilities? Приведите примеры.

### Уровень Middle

- Сравните Selenium Grid и Selenoid. В чём преимущества Selenoid?
- Как настроить Selenoid с помощью docker-compose? Опишите ключевые моменты.
- Что такое `browsers.json` в Selenoid? Какие параметры он содержит?
- Как организовать параллельный запуск тестов в нескольких браузерах в CI?
- Как обеспечить изоляцию тестов при параллельном запуске?
- Какие облачные сервисы для кросс-браузерного тестирования вы знаете?

### Уровень Senior

- Как построить инфраструктуру кросс-браузерного тестирования для команды из 20 QA?
- Как определить, какие браузеры включать в тест-план? Какие метрики использовать?
- Как масштабировать Selenoid для сотен параллельных сессий (Selenoid + GGR)?
- Как организовать визуальное кросс-браузерное тестирование?
- Как оптимизировать время прогона кросс-браузерных тестов?
- Как обрабатывать browser-specific баги: пропускать тесты или маркировать как известные?

---

## Практические задания

### Задание 1: Настройка Selenoid

1. Создайте `docker-compose.yml` для Selenoid и Selenoid UI
2. Настройте `browsers.json` с Chrome (2 версии) и Firefox (1 версия)
3. Запустите инфраструктуру и проверьте через `/status` endpoint
4. Откройте Selenoid UI и убедитесь, что все браузеры видны

### Задание 2: Фабрика драйверов

Реализуйте `DriverFactory`, поддерживающую:
- Локальный и удалённый (Selenoid) режимы
- Chrome, Firefox, Edge
- Конфигурацию через системные свойства (`-Dbrowser=chrome`, `-Dremote=true`)
- Включение записи видео и VNC в удалённом режиме

### Задание 3: Матрица в CI

Настройте GitHub Actions workflow, который:
- Запускает тесты параллельно в Chrome и Firefox
- Использует Selenoid как service container
- Собирает артефакты (отчёты, видео) для каждого браузера
- Не останавливает весь пайплайн при падении в одном браузере

### Задание 4: Кросс-браузерный баг-репорт

Найдите реальное различие в поведении одного и того же веб-приложения в Chrome и Firefox
(например, на сайте https://the-internet.herokuapp.com). Оформите баг-репорт с указанием:
- Окружение (браузер, версия, ОС)
- Шаги воспроизведения
- Ожидаемый/фактический результат
- Скриншоты из обоих браузеров

---

## Дополнительные ресурсы

- [Selenium Grid Documentation](https://www.selenium.dev/documentation/grid/)
- [Selenoid — Official Documentation](https://aerokube.com/selenoid/latest/)
- [Selenoid UI — GitHub](https://github.com/aerokube/selenoid-ui)
- [BrowserStack — Documentation](https://www.browserstack.com/docs)
- [SauceLabs — Documentation](https://docs.saucelabs.com/)
- [Can I Use](https://caniuse.com/) — проверка совместимости Web API с браузерами
- [Docker Hub — Selenoid Browser Images](https://hub.docker.com/u/selenoid)
- [GGR (Go Grid Router)](https://aerokube.com/ggr/latest/) — балансировщик нагрузки для Selenoid
