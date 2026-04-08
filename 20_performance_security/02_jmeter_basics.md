# JMeter -- основы

## Обзор

Apache JMeter -- это open-source инструмент для нагрузочного и функционального тестирования,
написанный на Java. Изначально разработан для тестирования веб-приложений, но сейчас
поддерживает множество протоколов: HTTP/HTTPS, JDBC, LDAP, JMS, FTP, SOAP, REST и другие.

JMeter -- самый распространённый инструмент нагрузочного тестирования в индустрии.
Для QA-инженера владение JMeter -- это обязательный навык при работе с performance testing.
Даже если в команде используют Gatling или k6, знание JMeter остаётся стандартом на собеседованиях.

JMeter работает на уровне протокола (protocol level) -- он не рендерит страницы, не выполняет
JavaScript. Это означает, что он не измеряет клиентскую производительность (browser rendering),
а фокусируется на серверной стороне.

---

## Установка и запуск

### Требования

- Java 8+ (рекомендуется Java 11+)
- Минимум 2 GB RAM (для генерации нагрузки -- больше)

### Установка

```bash
# Скачать и распаковать (или через brew на macOS)
brew install jmeter

# Или скачать с сайта
# https://jmeter.apache.org/download_jmeter.cgi

# Проверить версию
jmeter --version
```

### Запуск

```bash
# GUI-режим (для разработки скриптов)
jmeter

# CLI-режим (для генерации нагрузки -- ОБЯЗАТЕЛЬНО!)
jmeter -n -t test_plan.jmx -l results.jtl -e -o report_folder

# Параметры CLI:
# -n  -- non-GUI режим
# -t  -- путь к тест-плану (.jmx файл)
# -l  -- файл для записи результатов (.jtl)
# -e  -- генерация HTML-отчёта после теста
# -o  -- папка для HTML-отчёта
```

> **Важно:** Никогда не запускайте реальную нагрузку в GUI-режиме. GUI потребляет значительные
> ресурсы и искажает результаты. GUI используется только для разработки и отладки скриптов.

---

## Структура Test Plan

Test Plan -- это корневой элемент JMeter, содержащий все компоненты теста.
Файл сохраняется в формате `.jmx` (XML).

### Иерархия элементов

```
Test Plan
├── Thread Group                    ← Группа виртуальных пользователей
│   ├── Config Elements             ← Настройки (HTTP Defaults, CSV Data Set)
│   ├── Samplers                    ← Запросы (HTTP Request, JDBC Request)
│   │   ├── Assertions              ← Проверки ответа
│   │   └── Post Processors        ← Извлечение данных из ответа
│   ├── Timers                      ← Паузы между запросами
│   ├── Logic Controllers           ← Управление потоком (if, loop, random)
│   └── Listeners                   ← Сбор и отображение результатов
└── Thread Group 2 (опционально)
```

---

## Thread Group -- группа потоков

Thread Group -- центральный элемент, определяющий профиль нагрузки.
Каждый thread (поток) имитирует одного виртуального пользователя.

### Основные параметры

| Параметр | Описание | Пример |
|----------|----------|--------|
| **Number of Threads** | Количество виртуальных пользователей | 100 |
| **Ramp-Up Period** | Время (сек) для запуска всех потоков | 60 (1 пользователь в сек) |
| **Loop Count** | Количество итераций на поток | 10 или Infinite |
| **Duration** | Максимальная длительность теста | 300 сек |
| **Startup Delay** | Задержка перед стартом группы | 0 |

### Пример расчёта

```
Number of Threads = 100
Ramp-Up Period = 50 сек
→ Каждую секунду стартует 100/50 = 2 пользователя
→ Через 50 секунд все 100 пользователей будут активны
```

> **Совет:** Ramp-Up должен быть достаточно длинным, чтобы не создать spike-нагрузку
> при старте. Хорошее правило: ramp-up = количество потоков / 2..10.

---

## Samplers -- сэмплеры (запросы)

Samplers отправляют запросы к тестируемой системе. Самый используемый -- **HTTP Request**.

### HTTP Request Sampler

Настройки HTTP Request:

| Поле | Описание | Пример |
|------|----------|--------|
| **Protocol** | HTTP или HTTPS | https |
| **Server Name** | Хост | api.example.com |
| **Port** | Порт | 443 |
| **Method** | HTTP-метод | GET, POST, PUT, DELETE |
| **Path** | Путь | /api/v1/products |
| **Body Data** | Тело запроса (для POST/PUT) | JSON, XML |
| **Content Encoding** | Кодировка | UTF-8 |

### Пример: GET-запрос

```
HTTP Request:
  Method: GET
  Path: /api/v1/products?category=electronics&page=1
  Headers:
    Content-Type: application/json
    Authorization: Bearer ${token}
```

### Пример: POST-запрос с JSON-телом

```json
{
  "username": "${username}",
  "email": "${email}",
  "password": "TestPass123!"
}
```

> Переменные `${username}`, `${email}` -- это JMeter-переменные, которые подставляются
> динамически (из CSV, из предыдущих ответов, из User Defined Variables).

---

## Config Elements -- элементы конфигурации

### HTTP Request Defaults

Задаёт значения по умолчанию для всех HTTP Request в рамках Thread Group.
Если все запросы идут на один сервер -- укажите его здесь один раз.

```
Server Name: api.example.com
Port: 443
Protocol: https
Content Encoding: UTF-8
```

### HTTP Header Manager

Добавляет заголовки ко всем запросам:
- `Content-Type: application/json`
- `Accept: application/json`
- `Authorization: Bearer ${token}`

### HTTP Cookie Manager

Автоматически управляет cookies (как браузер). Необходим для тестирования
веб-приложений с сессиями.

### CSV Data Set Config

Один из важнейших элементов -- позволяет параметризовать тесты данными из CSV-файла.

**Файл `users.csv`:**
```csv
username,password,email
user1,pass1,user1@example.com
user2,pass2,user2@example.com
user3,pass3,user3@example.com
```

**Настройки CSV Data Set Config:**

| Параметр | Значение | Описание |
|----------|----------|----------|
| **Filename** | users.csv | Путь к файлу |
| **Variable Names** | username,password,email | Имена переменных |
| **Delimiter** | , | Разделитель |
| **Recycle on EOF** | True | Начать сначала при достижении конца файла |
| **Stop thread on EOF** | False | Не останавливать поток при конце файла |
| **Sharing mode** | All threads | Общий для всех потоков |

После настройки переменные `${username}`, `${password}`, `${email}` доступны в сэмплерах.

---

## Assertions -- проверки

Assertions проверяют ответ сервера. Если проверка не пройдена -- запрос считается неуспешным (FAIL).

### Response Assertion

Проверяет текст, код ответа, заголовки.

| Проверка | Пример |
|----------|--------|
| Response Code | Equals 200 |
| Response Message | Contains "OK" |
| Response Body | Contains "success" |
| Response Headers | Contains "application/json" |

### JSON Assertion

Проверяет структуру и значения в JSON-ответе с помощью JSONPath.

| JSONPath | Ожидание | Описание |
|----------|----------|----------|
| `$.status` | `success` | Значение поля status |
| `$.data.items` | (exists) | Поле существует |
| `$.data.items[0].id` | `1` | Значение первого элемента |
| `$.data.total` | > 0 | Проверка числового значения |

### Duration Assertion

Проверяет, что время ответа не превышает заданный порог.

```
Duration (ms): 2000  → Ответ должен прийти быстрее 2 секунд
```

### Size Assertion

Проверяет размер ответа (полезно для выявления пустых или аномально больших ответов).

---

## Listeners -- слушатели

Listeners собирают и отображают результаты тестирования.

### View Results Tree

Показывает детали каждого запроса и ответа. Незаменим при **отладке** скриптов.
Содержит:
- Request (заголовки, тело)
- Response (заголовки, тело, код)
- Sampler Result (время, размер)

> **Внимание:** View Results Tree хранит все данные в памяти. При нагрузочном тесте
> его использование приведёт к OutOfMemoryError. Используйте только при отладке.

### Summary Report

Таблица с агрегированными результатами:

| Метрика | Описание |
|---------|----------|
| Samples | Количество запросов |
| Average | Среднее время отклика (мс) |
| Min | Минимальное время |
| Max | Максимальное время |
| Std. Dev. | Стандартное отклонение |
| Error % | Процент ошибок |
| Throughput | Запросов в секунду |
| KB/sec | Пропускная способность |

### Aggregate Report

Расширенная версия Summary Report, включает:
- Медиану (Median)
- 90% Line (p90)
- 95% Line (p95)
- 99% Line (p99)

### Backend Listener

Отправляет результаты в реальном времени во внешние системы:
- InfluxDB + Grafana -- для визуализации в реальном времени
- Elasticsearch -- для анализа и поиска

---

## Timers -- таймеры

Timers добавляют задержки между запросами, имитируя реальное поведение пользователей.

### Constant Timer

Фиксированная пауза:
```
Thread Delay (ms): 1000  → Пауза 1 секунда между запросами
```

### Gaussian Random Timer

Случайная пауза с нормальным распределением:
```
Constant Delay Offset (ms): 1000
Deviation (ms): 500
→ Пауза от 500 мс до 1500 мс (в среднем 1000 мс)
```

### Uniform Random Timer

Равномерно распределённая случайная пауза:
```
Random Delay Maximum (ms): 3000
Constant Delay Offset (ms): 1000
→ Пауза от 1000 мс до 4000 мс
```

> **Отсутствие timers -- типичная ошибка новичков.** Без пауз скрипт генерирует
> нереалистично высокую нагрузку. Реальные пользователи делают паузы между действиями
> (think time), обычно 1-5 секунд.

---

## Recording Proxy -- запись сценариев

JMeter может записывать HTTP-трафик браузера через встроенный HTTP(S) Test Script Recorder.

### Настройка записи

1. Добавить **HTTP(S) Test Script Recorder** в Test Plan
2. Настроить порт прокси (по умолчанию 8888)
3. Добавить **Recording Controller** в Thread Group
4. Настроить браузер на использование прокси `localhost:8888`
5. Установить JMeter CA-сертификат для записи HTTPS
6. Нажать Start и выполнить действия в браузере
7. Записанные запросы появятся в Recording Controller

### Фильтрация при записи

Настройте исключения для статических ресурсов, чтобы не засорять скрипт:
```
Exclude patterns:
  .*\.(css|js|png|jpg|gif|ico|svg|woff|woff2|ttf).*
```

---

## Запуск из командной строки

### Базовый запуск

```bash
# Запуск теста с генерацией HTML-отчёта
jmeter -n -t /path/to/test_plan.jmx \
       -l /path/to/results.jtl \
       -e -o /path/to/html_report

# С переопределением параметров через properties
jmeter -n -t test_plan.jmx \
       -l results.jtl \
       -Jthreads=200 \        # Переопределить количество потоков
       -Jrampup=60 \          # Переопределить ramp-up
       -Jduration=300          # Переопределить длительность
```

### Использование properties в скрипте

В Thread Group вместо фиксированных значений:
```
Number of Threads: ${__P(threads,100)}    → По умолчанию 100
Ramp-Up Period:    ${__P(rampup,30)}      → По умолчанию 30
Duration:          ${__P(duration,180)}   → По умолчанию 180
```

### Генерация HTML-отчёта из существующего .jtl

```bash
# Если тест уже прошёл и есть .jtl файл
jmeter -g results.jtl -o html_report_folder
```

---

## JMeter + CI/CD интеграция

### Jenkins Pipeline (пример)

```groovy
pipeline {
    agent any
    stages {
        stage('Performance Test') {
            steps {
                // Запуск JMeter-теста
                sh '''
                    jmeter -n \
                        -t tests/performance/api_load_test.jmx \
                        -l results/results.jtl \
                        -Jthreads=${THREADS} \
                        -Jrampup=${RAMPUP} \
                        -e -o results/html_report
                '''
            }
        }
        stage('Check Results') {
            steps {
                // Проверка порога ошибок
                script {
                    // Парсинг результатов и fail если error rate > 1%
                    def errorRate = sh(
                        script: "awk -F',' 'NR>1{if(\$8==\"false\")e++; t++} END{print (e/t)*100}' results/results.jtl",
                        returnStdout: true
                    ).trim().toFloat()

                    if (errorRate > 1.0) {
                        error "Error rate ${errorRate}% exceeds threshold 1%"
                    }
                }
            }
        }
    }
    post {
        always {
            // Публикация HTML-отчёта
            publishHTML([
                reportDir: 'results/html_report',
                reportFiles: 'index.html',
                reportName: 'JMeter Report'
            ])
        }
    }
}
```

### Maven Plugin (jmeter-maven-plugin)

```xml
<plugin>
    <groupId>com.lazerycode.jmeter</groupId>
    <artifactId>jmeter-maven-plugin</artifactId>
    <version>3.7.0</version>
    <executions>
        <execution>
            <id>performance-test</id>
            <goals>
                <goal>jmeter</goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <testFilesDirectory>${project.basedir}/src/test/jmeter</testFilesDirectory>
        <resultsDirectory>${project.build.directory}/jmeter/results</resultsDirectory>
    </configuration>
</plugin>
```

---

## Пример: базовый нагрузочный тест REST API

### Сценарий

Тестируемый API: интернет-магазин. Сценарий:
1. Авторизация (POST /api/auth/login)
2. Получение списка товаров (GET /api/products)
3. Просмотр товара (GET /api/products/{id})
4. Добавление в корзину (POST /api/cart)

### Структура Test Plan

```
Test Plan
├── HTTP Request Defaults
│     Server: api.example.com, Port: 443, Protocol: https
├── HTTP Header Manager
│     Content-Type: application/json
├── CSV Data Set Config
│     File: users.csv (username, password)
├── Thread Group (100 users, ramp-up 60s, duration 300s)
│   ├── HTTP Request: Login (POST /api/auth/login)
│   │   ├── Body: {"username":"${username}","password":"${password}"}
│   │   ├── JSON Extractor: token = $.accessToken
│   │   └── Response Assertion: Response Code = 200
│   ├── HTTP Header Manager
│   │     Authorization: Bearer ${token}
│   ├── Gaussian Random Timer (1000ms ± 500ms)
│   ├── HTTP Request: Get Products (GET /api/products)
│   │   ├── JSON Assertion: $.data.items exists
│   │   └── JSON Extractor: productId = $.data.items[0].id
│   ├── Gaussian Random Timer (2000ms ± 1000ms)
│   ├── HTTP Request: View Product (GET /api/products/${productId})
│   │   └── Response Assertion: Response Code = 200
│   ├── Gaussian Random Timer (1500ms ± 500ms)
│   └── HTTP Request: Add to Cart (POST /api/cart)
│       ├── Body: {"productId":"${productId}","quantity":1}
│       └── Response Assertion: Response Code = 201
├── Aggregate Report
└── Backend Listener (InfluxDB)
```

---

## Связь с тестированием

- JMeter -- основной инструмент, о котором спрашивают на собеседованиях по performance testing
- Знание JMeter позволяет QA-инженеру самостоятельно создавать и запускать нагрузочные тесты
- Интеграция JMeter с CI/CD обеспечивает раннее обнаружение проблем производительности
- JMeter может использоваться и для функционального тестирования API (как альтернатива Postman для автоматизации)
- Навык чтения JMeter-отчётов необходим для анализа результатов, даже если вы не пишете скрипты лично

---

## Типичные ошибки

1. **Запуск нагрузки в GUI-режиме** -- GUI потребляет до 50% ресурсов машины, искажая результаты.
   Всегда используйте CLI: `jmeter -n -t test.jmx -l results.jtl`.

2. **Отсутствие think time** -- без таймеров один поток генерирует сотни запросов в секунду,
   что не соответствует реальному поведению пользователя.

3. **Все потоки используют одинаковые данные** -- без CSV Data Set Config все пользователи
   отправляют одинаковые запросы, что не выявляет реальных проблем (кэширование, конкуренция).

4. **Не извлекаются динамические данные** -- hardcoded session tokens и IDs приводят к ошибкам.
   Используйте JSON Extractor, Regular Expression Extractor.

5. **View Results Tree в нагрузочном тесте** -- приводит к OutOfMemoryError.
   При нагрузке используйте только Summary/Aggregate Report или Backend Listener.

6. **Недостаточный ramp-up** -- мгновенный старт 1000 потоков вызывает connection errors,
   не имеющие отношения к производительности приложения.

7. **Игнорирование корреляции** -- многие API возвращают токены, session ID, CSRF-токены,
   которые нужно извлекать и передавать в следующие запросы.

8. **Тестирование без baseline** -- сначала проведите базовый тест с малой нагрузкой,
   затем увеличивайте. Без baseline невозможно оценить деградацию.

---

## Вопросы на интервью

### 🟢 Базовый уровень (Junior)

1. Что такое JMeter? Для чего он используется?
2. Из каких основных элементов состоит Test Plan?
3. Что такое Thread Group и какие параметры у него есть?
4. Зачем нужны Timers в JMeter?
5. Как добавить проверку HTTP-кода ответа?
6. Почему нельзя запускать нагрузку в GUI-режиме?

### 🟡 Средний уровень (Middle)

1. Как параметризовать тест с помощью CSV Data Set Config?
2. Объясните разницу между Summary Report и Aggregate Report.
3. Как извлечь значение из JSON-ответа и использовать его в следующем запросе?
4. Что такое корреляция (correlation) в JMeter и зачем она нужна?
5. Как запустить JMeter из командной строки с переопределением параметров?
6. Как организовать тестирование REST API с авторизацией через Bearer Token?

### 🔴 Продвинутый уровень (Senior)

1. Как интегрировать JMeter в CI/CD pipeline? Приведите пример.
2. Как организовать распределённое тестирование (distributed testing) в JMeter?
3. Как настроить мониторинг результатов в реальном времени (InfluxDB + Grafana)?
4. Как написать кастомный sampler с помощью JSR223 / BeanShell?
5. Как оптимизировать JMeter для генерации максимальной нагрузки с одной машины?
6. Сравните JMeter с Gatling и k6. В каких случаях вы бы выбрали каждый инструмент?

---

## Практические задания

### Задание 1: Создание базового теста
Создайте Test Plan для тестирования открытого API (например, https://jsonplaceholder.typicode.com):
1. Thread Group: 10 пользователей, ramp-up 10 сек, 5 итераций
2. GET /posts -- получить все посты
3. GET /posts/1 -- получить конкретный пост
4. POST /posts -- создать пост (с JSON-телом)
5. Добавьте Response Assertion (код 200/201) и Aggregate Report

### Задание 2: Параметризация
К заданию 1 добавьте:
1. CSV-файл с данными (userId, title, body)
2. CSV Data Set Config для параметризации POST-запроса
3. Gaussian Random Timer (1-3 сек)
4. Запустите из CLI и сгенерируйте HTML-отчёт

### Задание 3: Корреляция
Создайте сценарий:
1. POST /posts -- создать пост
2. Извлечь `id` из ответа с помощью JSON Extractor
3. GET /posts/${id} -- получить созданный пост
4. Добавить JSON Assertion для проверки возвращённых данных

### Задание 4: CI/CD
Напишите Jenkins Pipeline (или GitHub Actions workflow), который:
1. Запускает JMeter-тест из CLI
2. Генерирует HTML-отчёт
3. Проверяет, что error rate < 1%
4. Публикует отчёт как артефакт

---

## Дополнительные ресурсы

- [Apache JMeter -- официальная документация](https://jmeter.apache.org/usermanual/index.html)
- [JMeter Best Practices](https://jmeter.apache.org/usermanual/best-practices.html)
- [BlazeMeter University -- JMeter курсы](https://www.blazemeter.com/university)
- [JMeter Plugins Manager](https://jmeter-plugins.org/)
- [Книга: "Master Apache JMeter - From Load Testing to DevOps" -- Antonio Gomes Rodrigues](https://leanpub.com/master-jmeter-from-load-test-to-devops)
- [InfluxDB + Grafana + JMeter -- мониторинг в реальном времени](https://grafana.com/grafana/dashboards/5496-apache-jmeter-dashboard-by-ubikloadpack/)
