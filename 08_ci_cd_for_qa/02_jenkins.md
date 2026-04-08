# Jenkins для QA

## Обзор

Jenkins — это open-source сервер автоматизации, который является одним из самых распространённых инструментов
для построения CI/CD pipeline. Для QA-инженера Jenkins — ключевой инструмент, через который запускаются
автотесты, генерируются отчёты, настраиваются расписания регрессий и формируются уведомления о результатах.
Несмотря на появление более современных систем (GitLab CI, GitHub Actions), Jenkins остаётся стандартом
в Enterprise-компаниях благодаря гибкости, огромной экосистеме плагинов и возможности self-hosted развёртывания.

---

## Архитектура Jenkins

### Master / Agent (Controller / Agent)

Jenkins работает по архитектуре Master-Agent (в новой терминологии — Controller-Agent).

```
┌─────────────────────────────────────────────────────┐
│                  JENKINS MASTER                      │
│              (Controller Node)                       │
│                                                      │
│  ┌──────────────┐  ┌──────────┐  ┌──────────────┐  │
│  │  Web UI      │  │ Scheduler│  │  Plugin Mgr  │  │
│  │  (Dashboard) │  │ (Cron)   │  │  (Allure,    │  │
│  │              │  │          │  │   Slack...)   │  │
│  └──────────────┘  └──────────┘  └──────────────┘  │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │           Job Configurations / Pipelines      │   │
│  └──────────────────────────────────────────────┘   │
└──────────┬──────────────┬──────────────┬────────────┘
           │              │              │
     ┌─────▼─────┐  ┌────▼──────┐  ┌───▼───────┐
     │  Agent 1  │  │  Agent 2  │  │  Agent 3  │
     │  (Linux)  │  │  (Windows)│  │  (Docker) │
     │           │  │           │  │           │
     │ Java 17   │  │ Java 11   │  │ Selenoid  │
     │ Maven     │  │ Gradle    │  │ Chrome    │
     │ Chrome    │  │ Edge      │  │ Firefox   │
     └───────────┘  └───────────┘  └───────────┘
```

**Master (Controller):**
- Управляет конфигурациями jobs и pipeline
- Планирует выполнение (scheduler)
- Хранит результаты сборок и артефакты
- Предоставляет Web UI
- Управляет плагинами

**Agent (Node):**
- Выполняет задачи, назначенные master
- Может быть физической машиной, VM или Docker-контейнером
- Помечается labels для маршрутизации задач (например, `linux`, `docker`, `selenium`)
- Может быть постоянным или запускаться по требованию (cloud agents)

---

## Jenkinsfile — Declarative Pipeline

Jenkinsfile — это файл конфигурации pipeline, который хранится в репозитории вместе с кодом (Pipeline as Code).
Существуют два синтаксиса: Declarative (рекомендуемый) и Scripted (Groovy). Для QA-проектов обычно достаточно
Declarative.

### Базовая структура Declarative Pipeline

```groovy
pipeline {
    // Указываем, на каком агенте запускать pipeline
    agent any

    // Инструменты, которые должны быть доступны
    tools {
        maven 'Maven-3.9'
        jdk 'JDK-17'
    }

    // Переменные окружения
    environment {
        BASE_URL = 'https://staging.example.com'
        BROWSER = 'chrome'
    }

    // Настройки pipeline
    options {
        timeout(time: 60, unit: 'MINUTES')    // Таймаут для всего pipeline
        timestamps()                           // Добавить временные метки в логи
        buildDiscarder(logRotator(            // Хранить только 10 последних сборок
            numToKeepStr: '10'
        ))
    }

    // Этапы pipeline
    stages {
        stage('Checkout') {
            steps {
                // Получение кода из репозитория
                checkout scm
            }
        }

        stage('Build') {
            steps {
                // Сборка проекта без запуска тестов
                sh 'mvn clean compile -DskipTests'
            }
        }

        stage('Unit Tests') {
            steps {
                sh 'mvn test -Dtest.suite=unit'
            }
        }

        stage('Integration Tests') {
            steps {
                sh 'mvn test -Dtest.suite=integration -Dbase.url=${BASE_URL}'
            }
        }
    }

    // Пост-действия: выполняются после всех stages
    post {
        always {
            // Генерация отчёта Allure (выполняется всегда)
            allure includeProperties: false,
                   results: [[path: 'target/allure-results']]

            // Публикация JUnit-отчётов
            junit 'target/surefire-reports/*.xml'
        }
        failure {
            // Уведомление при провале
            echo 'Pipeline FAILED!'
        }
        success {
            echo 'Pipeline PASSED!'
        }
    }
}
```

---

## Этапы Pipeline для тестового проекта

### Полный пример: Checkout → Build → Test → Report

```groovy
pipeline {
    agent { label 'linux && docker' }  // Агент с Docker

    tools {
        maven 'Maven-3.9'
        jdk 'JDK-17'
    }

    environment {
        BASE_URL     = "${params.BASE_URL ?: 'https://staging.example.com'}"
        BROWSER      = "${params.BROWSER ?: 'chrome'}"
        ALLURE_RESULTS = 'target/allure-results'
    }

    parameters {
        // Параметры сборки — можно менять при ручном запуске
        string(name: 'BASE_URL',
               defaultValue: 'https://staging.example.com',
               description: 'URL тестируемого приложения')
        choice(name: 'BROWSER',
               choices: ['chrome', 'firefox', 'edge'],
               description: 'Браузер для UI-тестов')
        choice(name: 'TEST_SUITE',
               choices: ['smoke', 'regression', 'full'],
               description: 'Набор тестов для запуска')
        booleanParam(name: 'RUN_PERFORMANCE',
                     defaultValue: false,
                     description: 'Запустить performance-тесты')
    }

    options {
        timeout(time: 90, unit: 'MINUTES')
        timestamps()
        ansiColor('xterm')                  // Цветной вывод в консоли
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    stages {

        stage('Checkout') {
            steps {
                // Клонирование репозитория
                checkout scm
                // Вывод информации о сборке
                sh '''
                    echo "=== Информация о сборке ==="
                    echo "Ветка: ${GIT_BRANCH}"
                    echo "Коммит: ${GIT_COMMIT}"
                    echo "URL: ${BASE_URL}"
                    echo "Браузер: ${BROWSER}"
                    echo "Набор тестов: ${TEST_SUITE}"
                '''
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean compile -DskipTests -q'
            }
        }

        stage('Unit Tests') {
            steps {
                sh 'mvn test -Dgroups=unit -Dmaven.test.failure.ignore=true'
            }
            post {
                always {
                    junit testResults: 'target/surefire-reports/*.xml',
                          allowEmptyResults: true
                }
            }
        }

        stage('API Tests') {
            steps {
                sh """
                    mvn test \\
                        -Dgroups=api \\
                        -Dbase.url=${BASE_URL} \\
                        -Dtest.suite=${params.TEST_SUITE} \\
                        -Dmaven.test.failure.ignore=true
                """
            }
        }

        stage('UI Tests') {
            steps {
                sh """
                    mvn test \\
                        -Dgroups=ui \\
                        -Dbase.url=${BASE_URL} \\
                        -Dbrowser=${BROWSER} \\
                        -Dselenide.remote=http://selenoid:4444/wd/hub \\
                        -Dtest.suite=${params.TEST_SUITE} \\
                        -Dmaven.test.failure.ignore=true
                """
            }
        }

        stage('Performance Tests') {
            when {
                // Запускаем только если выбран соответствующий параметр
                expression { params.RUN_PERFORMANCE == true }
            }
            steps {
                sh 'mvn gatling:test -Dbase.url=${BASE_URL}'
            }
        }
    }

    post {
        always {
            // Allure Report — всегда генерируем отчёт
            allure([
                includeProperties: false,
                jdk: '',
                results: [[path: "${ALLURE_RESULTS}"]]
            ])

            // Архивируем скриншоты и видео
            archiveArtifacts artifacts: 'target/screenshots/**',
                             allowEmptyArchive: true
            archiveArtifacts artifacts: 'target/video/**',
                             allowEmptyArchive: true

            // Очистка workspace
            cleanWs()
        }
        failure {
            // Уведомление в Slack при провале
            slackSend(
                channel: '#qa-alerts',
                color: 'danger',
                message: "❌ FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}\n" +
                         "URL: ${env.BUILD_URL}\n" +
                         "Allure: ${env.BUILD_URL}allure/"
            )
        }
        success {
            slackSend(
                channel: '#qa-results',
                color: 'good',
                message: "✅ PASSED: ${env.JOB_NAME} #${env.BUILD_NUMBER}\n" +
                         "Allure: ${env.BUILD_URL}allure/"
            )
        }
    }
}
```

---

## Parallel Stages — параллельные этапы

Параллельное выполнение позволяет значительно сократить время pipeline, запуская независимые тесты одновременно.

```groovy
stage('Parallel Tests') {
    parallel {
        stage('Chrome Tests') {
            agent { label 'docker' }
            steps {
                sh """
                    mvn test -Dgroups=ui \\
                        -Dbrowser=chrome \\
                        -Dselenide.remote=http://selenoid:4444/wd/hub
                """
            }
            post {
                always {
                    // Сохранение результатов для каждого параллельного потока
                    stash name: 'chrome-results',
                          includes: 'target/allure-results/**'
                }
            }
        }

        stage('Firefox Tests') {
            agent { label 'docker' }
            steps {
                sh """
                    mvn test -Dgroups=ui \\
                        -Dbrowser=firefox \\
                        -Dselenide.remote=http://selenoid:4444/wd/hub
                """
            }
            post {
                always {
                    stash name: 'firefox-results',
                          includes: 'target/allure-results/**'
                }
            }
        }

        stage('API Tests') {
            agent { label 'linux' }
            steps {
                sh 'mvn test -Dgroups=api -Dbase.url=${BASE_URL}'
            }
            post {
                always {
                    stash name: 'api-results',
                          includes: 'target/allure-results/**'
                }
            }
        }
    }
}

// Объединение результатов после параллельных этапов
stage('Merge Results') {
    steps {
        unstash 'chrome-results'
        unstash 'firefox-results'
        unstash 'api-results'
    }
    post {
        always {
            allure results: [[path: 'target/allure-results']]
        }
    }
}
```

---

## Credentials Management — управление секретами

Jenkins хранит секреты (пароли, токены, SSH-ключи) в зашифрованном виде. QA-инженер использует credentials
для доступа к тестовым окружениям, API-ключам, базам данных.

```groovy
environment {
    // Привязка credentials к переменным окружения
    DB_CREDS = credentials('staging-db-credentials')    // username:password
    API_TOKEN = credentials('api-auth-token')           // секретный текст
}

stages {
    stage('Test with Credentials') {
        steps {
            // DB_CREDS_USR — логин, DB_CREDS_PSW — пароль (автоматическое разделение)
            sh """
                mvn test \\
                    -Ddb.username=${DB_CREDS_USR} \\
                    -Ddb.password=${DB_CREDS_PSW} \\
                    -Dapi.token=${API_TOKEN}
            """
        }
    }
}
```

**Типы Credentials в Jenkins:**
- **Username with Password** — логин/пароль (например, для БД)
- **Secret Text** — токен или ключ API
- **Secret File** — файл (например, сертификат)
- **SSH Username with Private Key** — SSH-доступ к серверам
- **Certificate** — PKCS#12-сертификат

---

## Allure Jenkins Plugin

Allure — стандартный инструмент для отчётности в тестировании. Плагин Jenkins позволяет генерировать
и публиковать Allure Report прямо в интерфейсе Jenkins.

### Настройка

1. Установить плагин `Allure Jenkins Plugin` через Manage Jenkins → Plugins
2. Настроить Allure Commandline: Manage Jenkins → Tools → Allure Commandline
3. Использовать в Jenkinsfile:

```groovy
post {
    always {
        allure([
            includeProperties: false,
            jdk: '',
            reportBuildPolicy: 'ALWAYS',      // Генерировать отчёт всегда
            results: [[path: 'target/allure-results']]
        ])
    }
}
```

После сборки в интерфейсе Jenkins появится ссылка **Allure Report** с интерактивным отчётом,
включающим графики, историю запусков, прикреплённые скриншоты и логи.

---

## Scheduled Builds — запуск по расписанию

Для ночных регрессий и периодических проверок используется cron-синтаксис.

```groovy
pipeline {
    triggers {
        // Каждый будний день в 2:00 ночи
        cron('0 2 * * 1-5')

        // Каждый час по будням (для мониторинга)
        // cron('0 * * * 1-5')

        // Запуск при изменении в SCM (polling каждые 5 минут)
        // pollSCM('H/5 * * * *')
    }

    // ... остальная конфигурация
}
```

**Cron-синтаксис Jenkins:**
```
МИНУТЫ ЧАСЫ ДЕНЬ_МЕСЯЦА МЕСЯЦ ДЕНЬ_НЕДЕЛИ
  0      2      *          *       1-5

# H — хэш-символ Jenkins для распределения нагрузки
# H(0-30) — случайное значение от 0 до 30 (но стабильное для конкретного job)
```

---

## Уведомления

### Slack

```groovy
post {
    failure {
        slackSend(
            channel: '#qa-alerts',
            color: 'danger',
            message: """
                *FAILED:* ${env.JOB_NAME} #${env.BUILD_NUMBER}
                *Branch:* ${env.GIT_BRANCH}
                *Build:* ${env.BUILD_URL}
                *Allure:* ${env.BUILD_URL}allure/
            """.stripIndent()
        )
    }
}
```

### Email

```groovy
post {
    failure {
        emailext(
            subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            body: """
                <h2>Сборка провалена</h2>
                <p>Job: ${env.JOB_NAME}</p>
                <p>Build: <a href="${env.BUILD_URL}">${env.BUILD_NUMBER}</a></p>
                <p>Allure: <a href="${env.BUILD_URL}allure/">Отчёт</a></p>
            """,
            to: 'qa-team@company.com',
            mimeType: 'text/html'
        )
    }
}
```

---

## Связь с тестированием

Jenkins для QA — это не просто "кнопка запуска тестов". Это платформа, которая:

- **Обеспечивает повторяемость:** одинаковые условия запуска тестов при каждой сборке
- **Централизует отчётность:** Allure Report доступен всей команде через Web UI
- **Автоматизирует рутину:** ночные регрессии, кросс-браузерные прогоны запускаются без участия человека
- **Предоставляет историю:** можно отследить, когда конкретный тест начал падать (trend-графики)
- **Интегрирует инструменты:** Selenoid, Docker, Allure, Slack, Jira — всё объединено в единый pipeline
- **Масштабируется:** через agents можно запускать сотни тестов параллельно

---

## Типичные ошибки

1. **Запуск тестов на Master-ноде.** Master должен только управлять, а не выполнять тяжёлые задачи.
   Тесты следует запускать на Agent-нодах. В противном случае — замедление UI Jenkins и риск падения.

2. **Отсутствие `maven.test.failure.ignore=true`.** Без этого флага Maven прекращает сборку при первом
   упавшем тесте, и post-секция с Allure Report не выполняется. Результаты теряются.

3. **Не очищается workspace.** Артефакты предыдущих сборок накапливаются и могут приводить к ложным
   результатам или переполнению диска. Решение: `cleanWs()` в `post { always }`.

4. **Секреты в открытом виде в Jenkinsfile.** Пароли и токены никогда не должны быть в коде.
   Используйте `credentials()` и Jenkins Credentials Store.

5. **Нет таймаутов.** Зависший тест может блокировать agent на часы. Всегда устанавливайте
   `timeout()` на уровне pipeline и отдельных stages.

6. **Игнорирование `post`-секции.** Без `post { always }` артефакты не собираются при провале pipeline,
   что делает анализ невозможным.

7. **Monolithic pipeline.** Все тесты (unit, API, UI) в одном последовательном pipeline без
   параллелизации. Решение: использовать `parallel` stages.

8. **Отсутствие параметризации.** Для каждого окружения создаётся отдельный job вместо одного
   параметризованного. Решение: `parameters {}` блок.

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Что такое Jenkins? Какова его основная роль в CI/CD?
2. Объясните разницу между Master (Controller) и Agent в Jenkins.
3. Что такое Jenkinsfile? Где он хранится?
4. Чем отличается Declarative pipeline от Scripted pipeline?
5. Как посмотреть результаты выполнения тестов в Jenkins?

### 🟡 Средний уровень
6. Как настроить параллельный запуск тестов в Jenkinsfile?
7. Как использовать credentials в Jenkins для доступа к тестовым окружениям?
8. Как настроить запуск тестов по расписанию (cron)?
9. Для чего нужна `post`-секция? Перечислите возможные условия (always, failure, success и т.д.).
10. Как настроить Allure Report в Jenkins?
11. Как параметризовать pipeline для запуска с разными конфигурациями?

### 🔴 Продвинутый уровень
12. Как организовать распределённый запуск тестов на нескольких agents с объединением результатов?
13. Как реализовать dynamic pipeline — генерацию stages на основе данных (например, списка сервисов)?
14. Как настроить Jenkins Shared Libraries для переиспользования тестовых pipeline-конфигураций?
15. Как интегрировать Jenkins с Selenoid для масштабируемого UI-тестирования?
16. Как мониторить производительность самого Jenkins и его agents?

---

## Практические задания

### Задание 1: Написание Jenkinsfile
Напишите Jenkinsfile для тестового проекта со следующими требованиями:
- Параметры: URL окружения, браузер, набор тестов
- Этапы: checkout → build → unit tests → API tests → UI tests → report
- Allure Report в post-секции
- Уведомление в Slack при провале
- Таймаут: 60 минут

### Задание 2: Параллельный Pipeline
Модифицируйте Jenkinsfile из задания 1, добавив параллельные этапы:
- Chrome UI-тесты и Firefox UI-тесты запускаются одновременно
- API-тесты запускаются параллельно с UI-тестами
- Результаты объединяются в один Allure Report

### Задание 3: Ночная Регрессия
Настройте scheduled pipeline для ночной регрессии:
- Запуск в 2:00 по будням
- Полный набор тестов
- Уведомление о результатах в Slack и по email
- Сохранение артефактов (скриншоты, видео, логи) на 14 дней

### Задание 4: Анализ Проблемы
Ваш pipeline регулярно падает с ошибкой: тесты проходят локально, но не в Jenkins. Перечислите
возможные причины и шаги для диагностики. (Подсказка: окружение, зависимости, timing, ресурсы.)

---

## Дополнительные ресурсы

- [Jenkins Official Documentation](https://www.jenkins.io/doc/) — официальная документация
- [Jenkins Pipeline Syntax](https://www.jenkins.io/doc/book/pipeline/syntax/) — синтаксис pipeline
- [Allure Jenkins Plugin](https://plugins.jenkins.io/allure-jenkins-plugin/) — плагин Allure
- [Jenkins Best Practices](https://www.jenkins.io/doc/book/pipeline/pipeline-best-practices/) — лучшие практики
- [Blue Ocean](https://www.jenkins.io/projects/blueocean/) — современный UI для Jenkins
- [Jenkins Shared Libraries](https://www.jenkins.io/doc/book/pipeline/shared-libraries/) — общие библиотеки
