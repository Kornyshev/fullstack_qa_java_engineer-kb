# Структура проекта

## Обзор

Правильная структура проекта — это первый шаг к масштабируемому тестовому фреймворку. Она определяет, как быстро
новые участники включатся в работу, насколько легко искать и модифицировать код, и как эффективно команда будет
поддерживать автотесты. В этом разделе мы разберём типичную структуру Maven/Gradle-проекта для тестовой автоматизации,
соглашения об именовании, организацию пакетов и multi-module подход.

---

## Стандартная структура Maven-проекта

Maven и Gradle используют convention-over-configuration подход. Стандартная структура:

```
project-root/
├── pom.xml                          ← Конфигурация проекта и зависимости
├── README.md                        ← Документация для быстрого старта
├── .gitignore                       ← Исключения для Git
├── .env.example                     ← Шаблон переменных окружения
│
├── src/
│   ├── main/java/                   ← (обычно пустой в тестовом проекте)
│   └── test/
│       ├── java/
│       │   └── com/company/project/
│       │       ├── tests/           ← Тестовые классы
│       │       ├── pages/           ← Page Objects для UI
│       │       ├── api/             ← API-клиенты и модели
│       │       ├── steps/           ← Шаги бизнес-логики
│       │       ├── config/          ← Конфигурационные интерфейсы и классы
│       │       ├── data/            ← Фабрики тестовых данных
│       │       ├── db/              ← Работа с базами данных
│       │       ├── utils/           ← Утилитарные классы
│       │       ├── models/          ← POJO / DTO модели
│       │       ├── listeners/       ← Слушатели (Allure, custom)
│       │       └── base/            ← Базовые классы
│       │
│       └── resources/
│           ├── config/              ← Конфигурационные файлы
│           │   ├── application.properties
│           │   ├── dev.properties
│           │   ├── staging.properties
│           │   └── prod.properties
│           ├── testdata/            ← Тестовые данные
│           │   ├── users.json
│           │   └── products.csv
│           ├── allure.properties     ← Настройки Allure
│           ├── junit-platform.properties ← Настройки JUnit 5
│           └── logback-test.xml     ← Настройки логирования
│
├── build/                           ← (Генерируется при сборке)
├── allure-results/                  ← (Генерируется при запуске тестов)
└── allure-report/                   ← (Генерируется Allure)
```

---

## Детальный разбор каждого пакета

### `tests/` — Тестовые классы

Здесь живут **только тесты**. Никакой вспомогательной логики.

```
tests/
├── ui/
│   ├── LoginTest.java
│   ├── CatalogTest.java
│   ├── CartTest.java
│   └── CheckoutTest.java
├── api/
│   ├── UserApiTest.java
│   ├── ProductApiTest.java
│   └── OrderApiTest.java
└── db/
    └── UserRepositoryTest.java
```

Пример тестового класса:

```java
package com.company.project.tests.ui;

import com.company.project.base.BaseUiTest;
import com.company.project.steps.LoginSteps;
import com.company.project.steps.CatalogSteps;
import io.qameta.allure.Feature;
import io.qameta.allure.Story;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

@Feature("Каталог")
@Tag("ui")
public class CatalogTest extends BaseUiTest {

    private final LoginSteps loginSteps = new LoginSteps();
    private final CatalogSteps catalogSteps = new CatalogSteps();

    @Test
    @Story("Поиск товаров")
    @DisplayName("Поиск товара по названию возвращает релевантные результаты")
    void searchByNameReturnsRelevantResults() {
        loginSteps.loginAsDefaultUser();
        catalogSteps.searchProduct("iPhone");
        catalogSteps.verifySearchResultsContain("iPhone");
    }
}
```

### `pages/` — Page Objects

Каждый класс соответствует **одной странице** или **компоненту** UI:

```
pages/
├── LoginPage.java
├── CatalogPage.java
├── CartPage.java
├── components/
│   ├── HeaderComponent.java
│   ├── FooterComponent.java
│   └── ProductCardComponent.java
└── modals/
    ├── ConfirmationModal.java
    └── FilterModal.java
```

```java
package com.company.project.pages;

import com.codeborne.selenide.SelenideElement;
import com.company.project.base.BasePage;
import io.qameta.allure.Step;

import static com.codeborne.selenide.Selenide.$;

// Страница авторизации
public class LoginPage extends BasePage {

    // Локаторы — приватные элементы страницы
    private final SelenideElement usernameInput = $("#username");
    private final SelenideElement passwordInput = $("#password");
    private final SelenideElement loginButton = $("[data-testid='login-btn']");
    private final SelenideElement errorMessage = $(".error-message");

    @Step("Ввести логин '{username}'")
    public LoginPage enterUsername(String username) {
        usernameInput.setValue(username);
        return this;
    }

    @Step("Ввести пароль")
    public LoginPage enterPassword(String password) {
        passwordInput.setValue(password);
        return this;
    }

    @Step("Нажать кнопку 'Войти'")
    public void clickLogin() {
        loginButton.click();
    }

    @Step("Получить текст ошибки")
    public String getErrorText() {
        return errorMessage.getText();
    }
}
```

### `api/` — API-клиенты

Классы для взаимодействия с REST API:

```
api/
├── clients/
│   ├── UserApiClient.java
│   ├── ProductApiClient.java
│   └── OrderApiClient.java
├── models/
│   ├── request/
│   │   ├── CreateUserRequest.java
│   │   └── CreateOrderRequest.java
│   └── response/
│       ├── UserResponse.java
│       └── OrderResponse.java
└── specs/
    ├── RequestSpecs.java           ← Спецификации запросов
    └── ResponseSpecs.java          ← Спецификации ответов
```

### `steps/` — Шаги бизнес-логики

Промежуточный слой между тестами и Page Objects / API-клиентами:

```java
package com.company.project.steps;

import com.company.project.pages.LoginPage;
import com.company.project.config.ProjectConfig;
import io.qameta.allure.Step;
import org.aeonbits.owner.ConfigFactory;

// Шаги авторизации, переиспользуемые в разных тестах
public class LoginSteps {

    private final LoginPage loginPage = new LoginPage();
    private final ProjectConfig config = ConfigFactory.create(ProjectConfig.class);

    @Step("Авторизоваться как пользователь по умолчанию")
    public void loginAsDefaultUser() {
        loginPage.enterUsername(config.defaultUsername())
                 .enterPassword(config.defaultPassword())
                 .clickLogin();
    }

    @Step("Авторизоваться как '{username}'")
    public void loginAs(String username, String password) {
        loginPage.enterUsername(username)
                 .enterPassword(password)
                 .clickLogin();
    }
}
```

### `config/` — Конфигурация

```
config/
├── ProjectConfig.java              ← Owner-интерфейс
├── ConfigProvider.java             ← Провайдер конфигурации (Singleton)
└── Environment.java                ← Enum окружений
```

### `data/` — Фабрики тестовых данных

```
data/
├── UserFactory.java                ← Генерация пользователей
├── ProductFactory.java             ← Генерация товаров
└── TestDataReader.java             ← Чтение данных из файлов
```

```java
package com.company.project.data;

import com.company.project.models.User;
import com.github.javafaker.Faker;

// Фабрика создания тестовых пользователей
public class UserFactory {

    private static final Faker faker = new Faker();

    public static User randomUser() {
        return User.builder()
                .username(faker.name().username())
                .email(faker.internet().emailAddress())
                .password(faker.internet().password(8, 20))
                .firstName(faker.name().firstName())
                .lastName(faker.name().lastName())
                .build();
    }

    public static User adminUser() {
        return User.builder()
                .username("admin")
                .email("admin@test.com")
                .password("admin123")
                .firstName("Admin")
                .lastName("User")
                .build();
    }
}
```

### `base/` — Базовые классы

```
base/
├── BasePage.java                   ← Общие методы для всех страниц
├── BaseUiTest.java                 ← Настройка/очистка для UI-тестов
├── BaseApiTest.java                ← Настройка для API-тестов
└── BaseDbTest.java                 ← Подключение к БД
```

```java
package com.company.project.base;

import com.codeborne.selenide.Configuration;
import com.codeborne.selenide.logevents.SelenideLogger;
import com.company.project.config.ConfigProvider;
import io.qameta.allure.selenide.AllureSelenide;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.BeforeEach;

import static com.codeborne.selenide.Selenide.open;

// Базовый класс для всех UI-тестов
public abstract class BaseUiTest {

    @BeforeAll
    static void setupAllure() {
        SelenideLogger.addListener("allure", new AllureSelenide());
    }

    @BeforeEach
    void setupBrowser() {
        Configuration.baseUrl = ConfigProvider.getConfig().baseUrl();
        Configuration.browser = ConfigProvider.getConfig().browser();
        Configuration.browserSize = ConfigProvider.getConfig().browserSize();
        open(Configuration.baseUrl);
    }
}
```

### `utils/` — Утилитарные классы

```
utils/
├── DateUtils.java                  ← Работа с датами
├── JsonUtils.java                  ← Сериализация/десериализация
├── FileUtils.java                  ← Чтение файлов из resources
├── WaitUtils.java                  ← Кастомные ожидания
└── AllureUtils.java                ← Прикрепление скриншотов и логов
```

---

## Ресурсы (src/test/resources)

### config/ — Конфигурационные файлы

```properties
# application.properties — общие настройки
browser=chrome
browser.size=1920x1080

# dev.properties — настройки для dev-окружения
base.url=https://dev.myapp.com
api.base.url=https://api.dev.myapp.com

# staging.properties — настройки для staging
base.url=https://staging.myapp.com
api.base.url=https://api.staging.myapp.com
```

### testdata/ — Тестовые данные

```json
// users.json
[
  {
    "username": "testuser1",
    "password": "pass123",
    "role": "USER"
  },
  {
    "username": "testadmin",
    "password": "admin123",
    "role": "ADMIN"
  }
]
```

### Специальные файлы ресурсов

```properties
# junit-platform.properties — настройки параллельного запуска JUnit 5
junit.jupiter.execution.parallel.enabled=true
junit.jupiter.execution.parallel.mode.default=concurrent
junit.jupiter.execution.parallel.config.strategy=fixed
junit.jupiter.execution.parallel.config.fixed.parallelism=4
```

```properties
# allure.properties
allure.results.directory=allure-results
allure.link.tms.pattern=https://jira.company.com/browse/{}
allure.link.issue.pattern=https://jira.company.com/browse/{}
```

---

## Соглашения об именовании

### Пакеты

- Всегда lowercase: `com.company.project.tests.ui`
- Используйте обратный доменный формат: `com.company.project`
- Группируйте по назначению, не по типу файла

### Классы

| Тип | Суффикс | Пример |
|-----|---------|--------|
| Тестовый класс | `Test` | `LoginTest`, `UserApiTest` |
| Page Object | `Page` | `LoginPage`, `CatalogPage` |
| Компонент UI | `Component` | `HeaderComponent` |
| API-клиент | `ApiClient` | `UserApiClient` |
| Steps-класс | `Steps` | `LoginSteps` |
| Модель данных | без суффикса | `User`, `Product` |
| Утилитарный класс | `Utils` | `DateUtils`, `JsonUtils` |
| Базовый класс | `Base` + тип | `BaseUiTest`, `BasePage` |
| Конфигурация | `Config` | `ProjectConfig` |
| Фабрика данных | `Factory` | `UserFactory` |

### Тестовые методы

```java
// Хорошо — описывают поведение
void userCanLoginWithValidCredentials() { }
void searchReturnsEmptyResultsForNonExistingProduct() { }
void adminCanDeleteOtherUserAccount() { }

// Плохо — не информативно
void test1() { }
void loginTest() { }
void testSearch() { }
```

---

## Multi-module проекты

Для крупных проектов с микросервисной архитектурой имеет смысл разделить фреймворк на модули:

```
automation-framework/
├── pom.xml                          ← Родительский POM
├── common/                          ← Общие утилиты и конфигурация
│   ├── pom.xml
│   └── src/test/java/
│       └── com/company/common/
│           ├── config/
│           ├── utils/
│           └── base/
├── ui-tests/                        ← UI-тесты
│   ├── pom.xml
│   └── src/test/java/
│       └── com/company/ui/
│           ├── tests/
│           └── pages/
├── api-tests/                       ← API-тесты
│   ├── pom.xml
│   └── src/test/java/
│       └── com/company/api/
│           ├── tests/
│           └── clients/
└── performance-tests/               ← Нагрузочные тесты
    ├── pom.xml
    └── src/test/java/
```

Родительский `pom.xml`:

```xml
<project>
    <groupId>com.company</groupId>
    <artifactId>automation-framework</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>

    <modules>
        <module>common</module>
        <module>ui-tests</module>
        <module>api-tests</module>
        <module>performance-tests</module>
    </modules>

    <!-- Общие зависимости и версии для всех модулей -->
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>com.codeborne</groupId>
                <artifactId>selenide</artifactId>
                <version>7.2.3</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

---

## Связь с тестированием

Структура проекта напрямую влияет на эффективность тестирования:

- **Чёткое разделение** тестов по типам (UI/API/DB) позволяет запускать только нужные наборы.
- **Tag-based execution** — через `@Tag("smoke")` можно запускать только smoke-тесты.
- **Отдельные ресурсы для окружений** — переключение между dev/staging/prod без изменения кода.
- **Параллельный запуск** — правильная структура упрощает распараллеливание.
- **CI/CD интеграция** — каждый модуль может запускаться независимо в отдельном pipeline stage.

---

## Типичные ошибки

1. **Все классы в одном пакете** — при 100+ файлах навигация превращается в кошмар.
2. **Тестовая логика в `src/main/java`** — тесты должны жить в `src/test/java`.
3. **Хардкод путей к ресурсам** — используйте `getClass().getResource()` или classpath.
4. **Нет базовых классов** — дублирование `@BeforeEach` в каждом тестовом классе.
5. **Смешение Page Objects и тестов** — в одном пакете лежат и страницы, и тесты.
6. **Отсутствие `.gitignore`** — в репозитории оказываются `allure-results/`, `build/`, IDE-файлы.
7. **Отсутствие README** — никто не знает, как запустить тесты.
8. **Неконсистентное именование** — `LoginPage`, `catalogPO`, `Cart_page` в одном проекте.

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Опишите типичную структуру Maven-проекта для автотестов.
2. Для чего используется `src/test/resources`?
3. Какие соглашения об именовании тестовых классов вы знаете?
4. Что такое Page Object и в каком пакете он обычно находится?
5. Чем отличается `src/main/java` от `src/test/java`?

### 🟡 Средний уровень
6. Как организовать пакеты для UI-тестов и API-тестов в одном проекте?
7. Зачем нужны базовые классы (`BaseUiTest`, `BaseApiTest`)? Что в них находится?
8. Как вы организуете тестовые данные в проекте?
9. Что такое multi-module проект? Когда его стоит использовать?
10. Как обеспечить, чтобы конфигурация не попала в Git-репозиторий?

### 🔴 Продвинутый уровень
11. Как бы вы организовали структуру фреймворка для 5 микросервисов с общим UI?
12. Как разделить общие утилиты между несколькими тестовыми модулями?
13. Опишите стратегию версионирования для внутреннего тестового фреймворка.
14. Как организовать проект, чтобы разные команды могли работать независимо?
15. Как настроить selective test execution в CI/CD на основе изменённых файлов?

---

## Практические задания

### Задание 1: Создание структуры
Создайте с нуля Maven-проект для тестирования интернет-магазина. Определите пакеты, создайте базовые классы,
настройте `pom.xml` с основными зависимостями. Проект должен компилироваться и содержать хотя бы один тест.

### Задание 2: Рефакторинг структуры
Возьмите проект, где все файлы в одном пакете (tests, pages, utils — всё вместе).
Проведите рефакторинг: распределите по пакетам, добавьте базовые классы, вынесите конфигурацию в ресурсы.

### Задание 3: Multi-module
Разделите существующий monolith-проект на модули: `common`, `ui-tests`, `api-tests`.
Настройте зависимости между модулями. Убедитесь, что каждый модуль можно собрать и запустить отдельно.

### Задание 4: Code Review
Проведите code review структуры open-source тестового проекта на GitHub. Составьте список
рекомендаций по улучшению организации пакетов и именования.

---

## Дополнительные ресурсы

- [Maven Standard Directory Layout](https://maven.apache.org/guides/introduction/introduction-to-the-standard-directory-layout.html)
- [Gradle Project Structure](https://docs.gradle.org/current/userguide/organizing_gradle_projects.html)
- [Google Java Style Guide](https://google.github.io/styleguide/javaguide.html)
- [Baeldung — Guide to JUnit 5 Project Structure](https://www.baeldung.com/junit-5)
- [Test Automation University — Java courses](https://testautomationu.applitools.com/)
