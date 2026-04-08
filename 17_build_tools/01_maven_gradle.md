# Maven и Gradle для QA

## Обзор

Maven и Gradle — два основных инструмента сборки в экосистеме Java. Для QA Automation Engineer они важны не меньше, чем для разработчика: именно через build-инструмент управляются зависимости тестового фреймворка (Selenium, RestAssured, TestNG, Allure), запускаются тесты из командной строки и CI/CD, настраиваются профили для разных окружений. Знание Maven — обязательный минимум; знание Gradle — сильное конкурентное преимущество.

---

## Maven

### Что такое Maven

Maven — декларативный инструмент сборки и управления зависимостями. Конфигурация описывается в XML-файле `pom.xml` (Project Object Model). Maven следует принципу «Convention over Configuration» — он предлагает стандартную структуру проекта.

### Стандартная структура Maven-проекта

```
test-framework/
├── pom.xml                          # Конфигурация проекта
├── src/
│   ├── main/
│   │   ├── java/                    # Код приложения (для QA — утилиты, page objects)
│   │   └── resources/               # Ресурсы (конфигурации, данные)
│   └── test/
│       ├── java/                    # Тестовые классы
│       └── resources/               # Тестовые ресурсы (test data, properties)
└── target/                          # Результат сборки (генерируется автоматически)
```

### Структура pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <!-- Координаты проекта (GAV) — уникальный идентификатор -->
    <groupId>com.company.qa</groupId>         <!-- Организация / команда -->
    <artifactId>test-framework</artifactId>    <!-- Имя проекта -->
    <version>1.0.0-SNAPSHOT</version>          <!-- Версия -->
    <packaging>jar</packaging>                  <!-- Тип упаковки -->

    <!-- Свойства проекта — переменные для переиспользования -->
    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <selenium.version>4.18.1</selenium.version>
        <testng.version>7.9.0</testng.version>
        <allure.version>2.25.0</allure.version>
        <rest-assured.version>5.4.0</rest-assured.version>
    </properties>

    <!-- Зависимости проекта -->
    <dependencies>
        <!-- Selenium WebDriver — для UI-автотестов -->
        <dependency>
            <groupId>org.seleniumhq.selenium</groupId>
            <artifactId>selenium-java</artifactId>
            <version>${selenium.version}</version>
        </dependency>

        <!-- TestNG — тестовый фреймворк -->
        <dependency>
            <groupId>org.testng</groupId>
            <artifactId>testng</artifactId>
            <version>${testng.version}</version>
            <scope>test</scope>
        </dependency>

        <!-- Rest Assured — для API-тестов -->
        <dependency>
            <groupId>io.rest-assured</groupId>
            <artifactId>rest-assured</artifactId>
            <version>${rest-assured.version}</version>
            <scope>test</scope>
        </dependency>

        <!-- Allure — отчёты -->
        <dependency>
            <groupId>io.qameta.allure</groupId>
            <artifactId>allure-testng</artifactId>
            <version>${allure.version}</version>
        </dependency>
    </dependencies>

    <!-- Плагины сборки -->
    <build>
        <plugins>
            <!-- Плагин для запуска тестов -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.2.5</version>
                <configuration>
                    <suiteXmlFiles>
                        <suiteXmlFile>testng.xml</suiteXmlFile>
                    </suiteXmlFiles>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

### Ключевые элементы GAV

| Элемент | Описание | Пример |
|---------|----------|--------|
| `groupId` | Идентификатор организации (обратный домен) | `com.company.qa` |
| `artifactId` | Имя проекта | `test-framework` |
| `version` | Версия проекта | `1.0.0-SNAPSHOT` |

**SNAPSHOT** означает, что версия находится в разработке и может меняться. Без SNAPSHOT — это релизная версия.

### Scope зависимостей

| Scope | Описание | Пример использования |
|-------|----------|---------------------|
| `compile` | По умолчанию, доступна везде | Selenium, page objects |
| `test` | Только для тестов (src/test) | JUnit, TestNG, AssertJ |
| `provided` | Предоставляется контейнером | Servlet API |
| `runtime` | Нужна только при выполнении | JDBC-драйверы |

---

### Maven Lifecycle (жизненный цикл)

Maven имеет три встроенных жизненных цикла. Основной — **default lifecycle**:

```
validate → compile → test → package → verify → install → deploy
```

| Фаза | Что делает | Важность для QA |
|------|-----------|----------------|
| `clean` | Удаляет директорию `target/` | Чистый прогон без артефактов |
| `compile` | Компилирует исходный код | Проверка, что код компилируется |
| `test` | Запускает unit-тесты (surefire) | Основная фаза для QA |
| `package` | Создаёт JAR/WAR | Упаковка тестового фреймворка |
| `verify` | Запускает integration-тесты (failsafe) | Интеграционные тесты |
| `install` | Устанавливает артефакт в локальный репозиторий | Для использования как зависимость |
| `deploy` | Публикует артефакт в удалённый репозиторий | Для shared-библиотек |

**Важно:** выполнение фазы запускает все предыдущие. `mvn test` выполнит: `validate → compile → test`.

### Запуск из командной строки

```bash
# Очистка и запуск тестов (самая частая команда для QA)
mvn clean test

# Запуск с указанием конкретного тестового класса
mvn clean test -Dtest=LoginTest

# Запуск конкретного метода
mvn clean test -Dtest=LoginTest#testValidLogin

# Запуск тестов с определённым тегом/группой (TestNG)
mvn clean test -Dgroups=smoke

# Пропустить тесты (при сборке)
mvn clean package -DskipTests

# Запуск с профилем
mvn clean test -Pstaging
```

---

### Surefire Plugin

Maven Surefire Plugin — плагин для запуска unit-тестов (фаза `test`).

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.5</version>
    <configuration>
        <!-- Файл конфигурации TestNG -->
        <suiteXmlFiles>
            <suiteXmlFile>testng.xml</suiteXmlFile>
        </suiteXmlFiles>

        <!-- Или паттерн для поиска тестовых классов -->
        <includes>
            <include>**/*Test.java</include>
            <include>**/*Tests.java</include>
        </includes>

        <!-- Параллельный запуск -->
        <parallel>methods</parallel>
        <threadCount>4</threadCount>

        <!-- Системные свойства (передаются в тесты) -->
        <systemPropertyVariables>
            <browser>chrome</browser>
            <env>staging</env>
        </systemPropertyVariables>
    </configuration>
</plugin>
```

### Failsafe Plugin

Maven Failsafe Plugin — для запуска integration-тестов (фаза `verify`). Главное отличие от Surefire: Failsafe **не останавливает сборку** при падении тестов (сначала завершает все тесты, потом проверяет результаты).

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-failsafe-plugin</artifactId>
    <version>3.2.5</version>
    <configuration>
        <includes>
            <include>**/*IT.java</include>  <!-- Naming convention: IntegrationTest -->
        </includes>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>integration-test</goal>
                <goal>verify</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

| Аспект | Surefire | Failsafe |
|--------|----------|----------|
| Фаза | `test` | `integration-test` + `verify` |
| Паттерн имён | `*Test.java`, `Test*.java` | `*IT.java`, `IT*.java` |
| При падении теста | Сборка прерывается | Сборка продолжается до verify |
| Назначение | Unit-тесты | Интеграционные тесты |

---

### Maven Profiles

Профили позволяют настраивать разные конфигурации для разных окружений.

```xml
<profiles>
    <!-- Профиль для smoke-тестов -->
    <profile>
        <id>smoke</id>
        <properties>
            <testng.suite>testng-smoke.xml</testng.suite>
        </properties>
    </profile>

    <!-- Профиль для regression-тестов -->
    <profile>
        <id>regression</id>
        <properties>
            <testng.suite>testng-regression.xml</testng.suite>
        </properties>
    </profile>

    <!-- Профиль для staging-окружения -->
    <profile>
        <id>staging</id>
        <properties>
            <base.url>https://staging.example.com</base.url>
            <db.host>staging-db.example.com</db.host>
        </properties>
    </profile>

    <!-- Профиль для production (только read-only тесты) -->
    <profile>
        <id>production</id>
        <properties>
            <base.url>https://www.example.com</base.url>
            <testng.suite>testng-prod-smoke.xml</testng.suite>
        </properties>
    </profile>
</profiles>
```

```bash
# Запуск smoke-тестов на staging
mvn clean test -Psmoke,staging

# Запуск regression на production
mvn clean test -Pregression,production
```

### Dependency Management и репозитории

```xml
<!-- Управление версиями зависимостей (в parent pom) -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.qameta.allure</groupId>
            <artifactId>allure-bom</artifactId>
            <version>${allure.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- Репозитории для скачивания зависимостей -->
<repositories>
    <repository>
        <id>company-nexus</id>
        <url>https://nexus.company.com/repository/maven-public/</url>
    </repository>
</repositories>
```

**Репозитории Maven:**
- **Локальный** (`~/.m2/repository`) — кэш скачанных зависимостей
- **Центральный** (Maven Central) — публичный репозиторий
- **Корпоративный** (Nexus / Artifactory) — внутренний репозиторий компании

---

## Gradle

### Что такое Gradle

Gradle — современный инструмент сборки, использующий Groovy или Kotlin DSL вместо XML. Gradle быстрее Maven благодаря инкрементальной сборке и кэшированию.

### Структура build.gradle (Groovy DSL)

```groovy
plugins {
    id 'java'
    id 'io.qameta.allure' version '2.11.2'  // Плагин Allure
}

group = 'com.company.qa'
version = '1.0.0-SNAPSHOT'

// Версия Java
java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}

// Репозитории для скачивания зависимостей
repositories {
    mavenCentral()
    maven { url 'https://nexus.company.com/repository/maven-public/' }
}

// Зависимости проекта
dependencies {
    // Код фреймворка (page objects, утилиты)
    implementation 'org.seleniumhq.selenium:selenium-java:4.18.1'

    // Только для тестов
    testImplementation 'org.testng:testng:7.9.0'
    testImplementation 'io.rest-assured:rest-assured:5.4.0'
    testImplementation 'io.qameta.allure:allure-testng:2.25.0'
    testImplementation 'org.assertj:assertj-core:3.25.3'
}

// Конфигурация запуска тестов
test {
    useTestNG {
        suites 'src/test/resources/testng.xml'
    }

    // Системные свойства (доступны в тестах)
    systemProperty 'browser', System.getProperty('browser', 'chrome')
    systemProperty 'env', System.getProperty('env', 'staging')

    // Параллельный запуск
    maxParallelForks = 4

    // Логирование
    testLogging {
        events "passed", "skipped", "failed"
        showStandardStreams = true
    }
}
```

### Gradle: build.gradle.kts (Kotlin DSL)

```kotlin
plugins {
    java
    id("io.qameta.allure") version "2.11.2"
}

group = "com.company.qa"
version = "1.0.0-SNAPSHOT"

java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.seleniumhq.selenium:selenium-java:4.18.1")
    testImplementation("org.testng:testng:7.9.0")
    testImplementation("io.rest-assured:rest-assured:5.4.0")
}

tasks.test {
    useTestNG {
        suites("src/test/resources/testng.xml")
    }
}
```

### Запуск тестов через Gradle

```bash
# Запуск всех тестов
./gradlew clean test

# Запуск конкретного тестового класса
./gradlew test --tests "com.company.qa.LoginTest"

# Запуск конкретного метода
./gradlew test --tests "com.company.qa.LoginTest.testValidLogin"

# Передача системного свойства
./gradlew test -Dbrowser=firefox -Denv=staging

# Запуск с фильтрацией по паттерну
./gradlew test --tests "*Smoke*"
```

---

## Сравнение Maven и Gradle

| Критерий | Maven | Gradle |
|----------|-------|--------|
| Формат конфигурации | XML (`pom.xml`) | Groovy/Kotlin DSL (`build.gradle`) |
| Скорость сборки | Медленнее | Быстрее (инкрементальная сборка, кэш) |
| Кривая обучения | Проще для начинающих | Сложнее, но гибче |
| Гибкость | Менее гибкий (Convention over Configuration) | Очень гибкий (code-based) |
| Популярность в QA | Доминирует в тестовых проектах | Растёт, особенно в новых проектах |
| Документация | Обширная, давно существует | Хорошая, но менее объёмная |
| Управление зависимостями | `<dependency>` в XML | Одна строка в `dependencies {}` |
| Плагины | Огромная экосистема | Растущая экосистема |
| CI/CD интеграция | Поддерживается везде | Поддерживается везде |
| Wrapper | `mvnw` (с Maven 3.7+) | `gradlew` (стандартная практика) |

---

## Связь с тестированием

| Аспект | Как build tool помогает QA |
|--------|--------------------------|
| Управление зависимостями | Все библиотеки (Selenium, RestAssured, Allure) подключаются через build tool |
| Запуск тестов | Основной способ запуска тестов в CI/CD — через Maven/Gradle |
| Профили | Разные конфигурации для smoke, regression, разных окружений |
| Отчёты | Surefire reports, Allure reports генерируются build tool |
| Параллелизация | Настройка параллельного запуска через плагины |
| Фильтрация тестов | Запуск конкретных тестов, групп, классов из CLI |

---

## Типичные ошибки

1. **Не указывать scope для тестовых зависимостей** — TestNG или JUnit без `<scope>test</scope>` попадут в production-артефакт. Всегда указывайте `test` scope для тестовых библиотек.

2. **Хардкод версий в каждой зависимости** — вместо переменных в `<properties>`. При обновлении придётся менять версию в нескольких местах. Используйте properties для версий.

3. **Игнорирование `mvn clean`** — запуск `mvn test` без `clean` может использовать старые скомпилированные классы. Всегда используйте `mvn clean test`.

4. **Непонимание разницы Surefire vs Failsafe** — использование Surefire для интеграционных тестов. Результат: сборка прерывается при первом падении, не давая завершить все тесты.

5. **Отсутствие Maven Wrapper** — проект зависит от версии Maven, установленной на машине. Используйте `mvnw` для гарантии одинаковой версии.

6. **Конфликт зависимостей** — разные библиотеки требуют разные версии одного артефакта. Используйте `mvn dependency:tree` для диагностики.

7. **Коммит `target/` или `.gradle/`** — артефакты сборки не должны попадать в Git. Добавляйте в `.gitignore`.

8. **Неиспользование профилей** — одинаковая конфигурация для всех окружений. Используйте profiles для staging / production / local.

---

## Вопросы на интервью

### 🟢 Базовый уровень

1. Что такое Maven и зачем он нужен в тестовом проекте?
2. Из чего состоит координата GAV?
3. Что такое `pom.xml`? Какие основные секции он содержит?
4. Как запустить тесты через Maven из командной строки?
5. Что делает команда `mvn clean test`?
6. Что такое `scope` зависимости? Какие бывают?

### 🟡 Средний уровень

7. Опишите жизненный цикл Maven. Какие фазы важны для QA?
8. В чём разница между Surefire и Failsafe плагинами?
9. Что такое Maven profiles и как их используют в тестировании?
10. Как запустить конкретный тестовый класс или метод из CLI?
11. Сравните Maven и Gradle. Когда лучше использовать каждый?
12. Как передать параметры (браузер, URL окружения) в тесты через Maven?

### 🔴 Продвинутый уровень

13. Как разрешить конфликт зависимостей в Maven? Как найти проблему?
14. Как настроить параллельный запуск тестов через Surefire?
15. Как организовать multi-module Maven-проект для тестового фреймворка?
16. Как настроить Gradle для запуска тестов с Allure-отчётами?
17. Как работает dependency resolution в Gradle? Чем отличается от Maven?

---

## Практические задания

### Задание 1: Создание Maven-проекта

1. Создайте Maven-проект с `pom.xml`
2. Добавьте зависимости: Selenium, TestNG, RestAssured, Allure
3. Напишите простой тест
4. Запустите через `mvn clean test` из командной строки

### Задание 2: Профили для окружений

1. В `pom.xml` создайте профили `local`, `staging`, `production`
2. Каждый профиль должен устанавливать свой `base.url`
3. В тесте считайте `System.getProperty("base.url")`
4. Запустите тесты с разными профилями и убедитесь, что URL меняется

### Задание 3: Миграция на Gradle

1. Возьмите существующий Maven-проект (из задания 1)
2. Создайте эквивалентный `build.gradle`
3. Перенесите все зависимости и конфигурацию тестов
4. Запустите тесты через `./gradlew clean test`
5. Сравните время сборки между Maven и Gradle

### Задание 4: Диагностика зависимостей

1. Выполните `mvn dependency:tree` для вашего проекта
2. Найдите все транзитивные зависимости
3. Проверьте наличие конфликтов версий
4. При наличии конфликта — разрешите его через `<exclusion>` или `<dependencyManagement>`

---

## Дополнительные ресурсы

- [Maven Official Documentation](https://maven.apache.org/guides/) — официальная документация Maven
- [Gradle User Manual](https://docs.gradle.org/current/userguide/userguide.html) — официальное руководство Gradle
- [Maven Repository (mvnrepository.com)](https://mvnrepository.com/) — поиск зависимостей
- [Baeldung: Maven Tutorial](https://www.baeldung.com/maven) — подробные руководства с примерами
- [Baeldung: Gradle Tutorial](https://www.baeldung.com/gradle) — руководства по Gradle
- [Maven Surefire Plugin Docs](https://maven.apache.org/surefire/maven-surefire-plugin/) — документация по Surefire
