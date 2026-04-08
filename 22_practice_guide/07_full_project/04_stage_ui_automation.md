# Этап 3: UI-автоматизация Conduit

## Обзор

На предыдущем этапе мы покрыли все API-эндпоинты Conduit тестами на REST Assured. Теперь
переходим к автоматизации пользовательского интерфейса. Selenide позволяет писать лаконичные
и стабильные UI-тесты с встроенными ожиданиями, скриншотами при падении и удобной интеграцией
с Allure. Мы реализуем полноценный Page Object Model для всех ключевых страниц приложения.

> **Цель:** Создать стабильный набор UI-тестов для ключевых пользовательских сценариев Conduit,
> используя Selenide, Page Object Model и Allure-аннотации.

---

## Предварительные требования

Перед началом убедитесь, что:
- Conduit запущен локально (UI доступен на `http://localhost:4100`)
- API-тесты из Этапа 2 проходят
- Установлен Chrome или Chromium
- В `pom.xml` подключены Selenide и Allure

---

## Часть 1: Структура проекта для UI-тестов

### 1.1 Целевая структура пакетов

```
src/test/java/
├── ui/
│   ├── pages/                     # Page Objects
│   │   ├── BasePage.java          # Базовый класс с общими методами
│   │   ├── HomePage.java          # Главная страница (лента статей)
│   │   ├── LoginPage.java         # Страница логина
│   │   ├── RegisterPage.java      # Страница регистрации
│   │   ├── ArticlePage.java       # Страница просмотра статьи
│   │   ├── EditorPage.java        # Редактор статей (создание/редактирование)
│   │   ├── ProfilePage.java       # Профиль пользователя
│   │   └── SettingsPage.java      # Настройки профиля
│   ├── components/                # Переиспользуемые UI-компоненты
│   │   ├── NavigationBar.java     # Навигационная панель
│   │   └── ArticlePreview.java    # Превью статьи в ленте
│   └── tests/                     # Тестовые классы
│       ├── BaseUiTest.java        # Базовый тестовый класс
│       ├── LoginUiTest.java       # Тесты логина
│       ├── RegisterUiTest.java    # Тесты регистрации
│       ├── ArticleUiTest.java     # Тесты статей (CRUD)
│       ├── CommentUiTest.java     # Тесты комментариев
│       ├── ProfileUiTest.java     # Тесты профиля и подписок
│       └── FavoriteUiTest.java    # Тесты избранного
├── config/
│   └── ConfigReader.java          # (из Этапа 2)
└── utils/
    ├── DataGenerator.java         # (из Этапа 2)
    └── UiHelper.java              # Вспомогательные методы для UI
```

### 1.2 Дополнительные зависимости в pom.xml

```xml
<dependencies>
    <!-- Selenide: UI-тестирование с встроенными ожиданиями -->
    <dependency>
        <groupId>com.codeborne</groupId>
        <artifactId>selenide</artifactId>
        <version>7.2.3</version>
        <scope>test</scope>
    </dependency>

    <!-- Allure Selenide: интеграция скриншотов и логов с Allure -->
    <dependency>
        <groupId>io.qameta.allure</groupId>
        <artifactId>allure-selenide</artifactId>
        <version>2.25.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

## Часть 2: Конфигурация Selenide

### 2.1 Расширение config.properties

```properties
# Базовый URL для UI
app.base.url=http://localhost:4100

# Настройки браузера
browser=chrome
browser.size=1920x1080
headless=false
timeout=8000
page.load.timeout=15000
```

### 2.2 Базовый тестовый класс BaseUiTest

```java
package ui.tests;

import api.client.AuthApi;
import api.client.TokenManager;
import api.models.request.RegisterRequest;
import api.models.response.UserResponse;
import com.codeborne.selenide.Configuration;
import com.codeborne.selenide.Selenide;
import com.codeborne.selenide.logevents.SelenideLogger;
import config.ConfigReader;
import io.qameta.allure.selenide.AllureSelenide;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Tag;
import utils.DataGenerator;

/**
 * Базовый класс для всех UI-тестов.
 * Настраивает Selenide, Allure и управляет жизненным циклом браузера.
 */
@Tag("ui")
public abstract class BaseUiTest {

    @BeforeAll
    static void setUpAll() {
        // Настройка Selenide
        Configuration.baseUrl = ConfigReader.get("app.base.url");
        Configuration.browser = ConfigReader.get("browser");
        Configuration.browserSize = ConfigReader.get("browser.size");
        Configuration.timeout = Long.parseLong(
                ConfigReader.get("timeout") != null ? ConfigReader.get("timeout") : "8000");
        Configuration.pageLoadTimeout = Long.parseLong(
                ConfigReader.get("page.load.timeout") != null
                        ? ConfigReader.get("page.load.timeout") : "15000");

        // Headless-режим для CI
        String headless = ConfigReader.get("headless");
        if ("true".equals(headless)) {
            Configuration.headless = true;
        }

        // Allure Selenide listener: автоматические скриншоты при падении,
        // логирование шагов Selenide, сохранение page source
        SelenideLogger.addListener("AllureSelenide",
                new AllureSelenide()
                        .screenshots(true)           // Скриншот при падении
                        .savePageSource(true)         // Сохранение HTML при падении
                        .includeSelenideSteps(true)); // Логирование каждого действия Selenide
    }

    @AfterEach
    void tearDown() {
        // Закрываем браузер после каждого теста для изоляции
        Selenide.closeWebDriver();
    }

    /**
     * Вспомогательный метод: авторизация через UI.
     * Устанавливает JWT-токен в localStorage, что быстрее, чем заполнять форму логина.
     * Используется в тестах, где логин -- не предмет тестирования.
     */
    protected void loginViaApi() {
        String username = DataGenerator.randomUsername();
        String email = DataGenerator.randomEmail();
        String password = DataGenerator.randomPassword();

        RegisterRequest request = new RegisterRequest(username, email, password);
        UserResponse response = AuthApi.register(request)
                .then()
                .statusCode(200)
                .extract()
                .as(UserResponse.class);

        String token = response.getUser().getToken();

        // Открываем пустую страницу и устанавливаем токен в localStorage
        Selenide.open("/");
        Selenide.executeJavaScript(
                "window.localStorage.setItem('jwtToken', arguments[0]);", token);
        // Перезагружаем страницу, чтобы приложение прочитало токен
        Selenide.refresh();
    }

    /**
     * Вспомогательный метод: авторизация с возвратом имени пользователя.
     */
    protected String loginViaApiAndGetUsername() {
        String username = DataGenerator.randomUsername();
        String email = DataGenerator.randomEmail();
        String password = DataGenerator.randomPassword();

        RegisterRequest request = new RegisterRequest(username, email, password);
        UserResponse response = AuthApi.register(request)
                .then()
                .statusCode(200)
                .extract()
                .as(UserResponse.class);

        String token = response.getUser().getToken();

        Selenide.open("/");
        Selenide.executeJavaScript(
                "window.localStorage.setItem('jwtToken', arguments[0]);", token);
        Selenide.refresh();

        return username;
    }
}
```

---

## Часть 3: Page Objects

### 3.1 BasePage -- базовый класс страниц

```java
package ui.pages;

import com.codeborne.selenide.Condition;
import com.codeborne.selenide.SelenideElement;
import io.qameta.allure.Step;

import static com.codeborne.selenide.Selenide.$;

/**
 * Базовый класс для всех Page Objects.
 * Содержит общие элементы (навигация) и вспомогательные методы.
 */
public abstract class BasePage {

    // Навигационная панель -- общая для всех страниц
    protected final SelenideElement navHome = $("a.navbar-brand");
    protected final SelenideElement navNewArticle = $("a[href='#/editor']");
    protected final SelenideElement navSettings = $("a[href='#/settings']");
    protected final SelenideElement navSignIn = $("a[href='#/login']");
    protected final SelenideElement navSignUp = $("a[href='#/register']");

    @Step("Переход на главную страницу через логотип")
    public HomePage goToHome() {
        navHome.click();
        return new HomePage();
    }

    @Step("Переход к созданию новой статьи")
    public EditorPage goToNewArticle() {
        navNewArticle.click();
        return new EditorPage();
    }

    @Step("Переход в настройки профиля")
    public SettingsPage goToSettings() {
        navSettings.click();
        return new SettingsPage();
    }

    @Step("Переход на страницу входа")
    public LoginPage goToSignIn() {
        navSignIn.click();
        return new LoginPage();
    }

    @Step("Переход на страницу регистрации")
    public RegisterPage goToSignUp() {
        navSignUp.click();
        return new RegisterPage();
    }

    @Step("Проверка: пользователь авторизован (видна ссылка Settings)")
    public void shouldBeLoggedIn() {
        navSettings.shouldBe(Condition.visible);
    }

    @Step("Проверка: пользователь не авторизован (видна ссылка Sign in)")
    public void shouldBeLoggedOut() {
        navSignIn.shouldBe(Condition.visible);
    }

    /**
     * Возвращает имя пользователя из навигационной панели.
     * Ссылка на профиль содержит username.
     */
    @Step("Получение имени пользователя из навигации")
    public String getLoggedInUsername() {
        return $("nav a[href^='#/@']").getText().trim();
    }
}
```

### 3.2 LoginPage

```java
package ui.pages;

import com.codeborne.selenide.Condition;
import com.codeborne.selenide.SelenideElement;
import io.qameta.allure.Step;

import static com.codeborne.selenide.Selenide.$;
import static com.codeborne.selenide.Selenide.open;

/**
 * Page Object для страницы авторизации (/login).
 */
public class LoginPage extends BasePage {

    private final SelenideElement emailInput = $("input[placeholder='Email']");
    private final SelenideElement passwordInput = $("input[placeholder='Password']");
    private final SelenideElement signInButton = $("button[type='submit']");
    private final SelenideElement errorMessages = $(".error-messages");
    private final SelenideElement signUpLink = $("a[href='#/register']");

    @Step("Открытие страницы логина")
    public LoginPage openPage() {
        open("/#/login");
        signInButton.shouldBe(Condition.visible);
        return this;
    }

    @Step("Ввод email: {email}")
    public LoginPage typeEmail(String email) {
        emailInput.setValue(email);
        return this;
    }

    @Step("Ввод пароля")
    public LoginPage typePassword(String password) {
        passwordInput.setValue(password);
        return this;
    }

    @Step("Нажатие кнопки Sign in")
    public LoginPage clickSignIn() {
        signInButton.click();
        return this;
    }

    /**
     * Полный сценарий логина: заполнение полей и нажатие кнопки.
     */
    @Step("Авторизация: email={email}")
    public HomePage loginAs(String email, String password) {
        typeEmail(email);
        typePassword(password);
        clickSignIn();
        return new HomePage();
    }

    @Step("Проверка отображения ошибки авторизации")
    public LoginPage shouldHaveError() {
        errorMessages.shouldBe(Condition.visible);
        return this;
    }

    @Step("Проверка текста ошибки: {expectedText}")
    public LoginPage shouldHaveErrorText(String expectedText) {
        errorMessages.shouldHave(Condition.text(expectedText));
        return this;
    }

    @Step("Переход на страницу регистрации по ссылке")
    public RegisterPage clickSignUpLink() {
        signUpLink.click();
        return new RegisterPage();
    }
}
```

### 3.3 RegisterPage

```java
package ui.pages;

import com.codeborne.selenide.Condition;
import com.codeborne.selenide.SelenideElement;
import io.qameta.allure.Step;

import static com.codeborne.selenide.Selenide.$;
import static com.codeborne.selenide.Selenide.open;

/**
 * Page Object для страницы регистрации (/register).
 */
public class RegisterPage extends BasePage {

    private final SelenideElement usernameInput = $("input[placeholder='Username']");
    private final SelenideElement emailInput = $("input[placeholder='Email']");
    private final SelenideElement passwordInput = $("input[placeholder='Password']");
    private final SelenideElement signUpButton = $("button[type='submit']");
    private final SelenideElement errorMessages = $(".error-messages");
    private final SelenideElement signInLink = $("a[href='#/login']");

    @Step("Открытие страницы регистрации")
    public RegisterPage openPage() {
        open("/#/register");
        signUpButton.shouldBe(Condition.visible);
        return this;
    }

    @Step("Ввод username: {username}")
    public RegisterPage typeUsername(String username) {
        usernameInput.setValue(username);
        return this;
    }

    @Step("Ввод email: {email}")
    public RegisterPage typeEmail(String email) {
        emailInput.setValue(email);
        return this;
    }

    @Step("Ввод пароля")
    public RegisterPage typePassword(String password) {
        passwordInput.setValue(password);
        return this;
    }

    @Step("Нажатие кнопки Sign up")
    public RegisterPage clickSignUp() {
        signUpButton.click();
        return this;
    }

    /**
     * Полный сценарий регистрации.
     */
    @Step("Регистрация: username={username}, email={email}")
    public HomePage registerAs(String username, String email, String password) {
        typeUsername(username);
        typeEmail(email);
        typePassword(password);
        clickSignUp();
        return new HomePage();
    }

    @Step("Проверка отображения ошибки регистрации")
    public RegisterPage shouldHaveError() {
        errorMessages.shouldBe(Condition.visible);
        return this;
    }

    @Step("Проверка текста ошибки содержит: {expectedText}")
    public RegisterPage shouldHaveErrorContaining(String expectedText) {
        errorMessages.shouldHave(Condition.text(expectedText));
        return this;
    }
}
```

### 3.4 HomePage

```java
package ui.pages;

import com.codeborne.selenide.CollectionCondition;
import com.codeborne.selenide.Condition;
import com.codeborne.selenide.ElementsCollection;
import com.codeborne.selenide.SelenideElement;
import io.qameta.allure.Step;

import static com.codeborne.selenide.Selenide.*;

/**
 * Page Object для главной страницы с лентой статей.
 */
public class HomePage extends BasePage {

    private final SelenideElement globalFeedTab = $("a[data-test='global-feed']");
    private final SelenideElement yourFeedTab = $("a[data-test='your-feed']");
    private final ElementsCollection articlePreviews = $$(".article-preview");
    private final ElementsCollection sidebarTags = $$(".sidebar .tag-list a");

    @Step("Открытие главной страницы")
    public HomePage openPage() {
        open("/");
        return this;
    }

    @Step("Переключение на вкладку Global Feed")
    public HomePage clickGlobalFeed() {
        // На некоторых реализациях Conduit селектор может отличаться
        $(".feed-toggle a", 0).click();
        return this;
    }

    @Step("Переключение на вкладку Your Feed")
    public HomePage clickYourFeed() {
        $(".feed-toggle a", 0).click();
        return this;
    }

    @Step("Проверка: лента содержит статьи")
    public HomePage shouldHaveArticles() {
        articlePreviews.shouldHave(CollectionCondition.sizeGreaterThan(0));
        return this;
    }

    @Step("Проверка: лента содержит минимум {count} статей")
    public HomePage shouldHaveAtLeastArticles(int count) {
        articlePreviews.shouldHave(CollectionCondition.sizeGreaterThanOrEqual(count));
        return this;
    }

    @Step("Клик на первую статью в ленте")
    public ArticlePage clickFirstArticle() {
        articlePreviews.first().$("a.preview-link").click();
        return new ArticlePage();
    }

    @Step("Клик на тег в sidebar: {tagName}")
    public HomePage clickSidebarTag(String tagName) {
        sidebarTags.findBy(Condition.text(tagName)).click();
        return this;
    }

    @Step("Проверка: sidebar содержит тег {tagName}")
    public HomePage shouldHaveTag(String tagName) {
        sidebarTags.findBy(Condition.text(tagName)).shouldBe(Condition.visible);
        return this;
    }

    @Step("Клик на кнопку лайка первой статьи")
    public HomePage clickFavoriteOnFirstArticle() {
        articlePreviews.first().$("button.btn-outline-primary").click();
        return this;
    }

    @Step("Получение счётчика лайков первой статьи")
    public String getFirstArticleFavoritesCount() {
        return articlePreviews.first().$("button.btn-outline-primary").getText().trim();
    }
}
```

### 3.5 EditorPage

```java
package ui.pages;

import com.codeborne.selenide.Condition;
import com.codeborne.selenide.SelenideElement;
import io.qameta.allure.Step;

import static com.codeborne.selenide.Selenide.$;
import static com.codeborne.selenide.Selenide.open;

/**
 * Page Object для редактора статей (/editor).
 * Используется как для создания, так и для редактирования.
 */
public class EditorPage extends BasePage {

    private final SelenideElement titleInput =
            $("input[placeholder='Article Title']");
    private final SelenideElement descriptionInput =
            $("input[placeholder=\"What's this article about?\"]");
    private final SelenideElement bodyTextarea =
            $("textarea[placeholder='Write your article (in markdown)']");
    private final SelenideElement tagsInput =
            $("input[placeholder='Enter tags']");
    private final SelenideElement publishButton =
            $("button[type='button']");
    private final SelenideElement errorMessages =
            $(".error-messages");

    @Step("Открытие страницы создания статьи")
    public EditorPage openPage() {
        open("/#/editor");
        titleInput.shouldBe(Condition.visible);
        return this;
    }

    @Step("Ввод заголовка статьи: {title}")
    public EditorPage typeTitle(String title) {
        titleInput.setValue(title);
        return this;
    }

    @Step("Ввод описания статьи: {description}")
    public EditorPage typeDescription(String description) {
        descriptionInput.setValue(description);
        return this;
    }

    @Step("Ввод тела статьи")
    public EditorPage typeBody(String body) {
        bodyTextarea.setValue(body);
        return this;
    }

    @Step("Добавление тега: {tag}")
    public EditorPage addTag(String tag) {
        tagsInput.setValue(tag).pressEnter();
        return this;
    }

    @Step("Нажатие кнопки Publish Article")
    public ArticlePage clickPublish() {
        publishButton.click();
        return new ArticlePage();
    }

    /**
     * Полный сценарий создания статьи.
     */
    @Step("Создание статьи: title={title}")
    public ArticlePage createArticle(String title, String description,
                                     String body, String... tags) {
        typeTitle(title);
        typeDescription(description);
        typeBody(body);
        for (String tag : tags) {
            addTag(tag);
        }
        return clickPublish();
    }

    @Step("Очистка и ввод нового заголовка: {newTitle}")
    public EditorPage clearAndTypeTitle(String newTitle) {
        titleInput.clear();
        titleInput.setValue(newTitle);
        return this;
    }

    @Step("Очистка и ввод нового тела статьи")
    public EditorPage clearAndTypeBody(String newBody) {
        bodyTextarea.clear();
        bodyTextarea.setValue(newBody);
        return this;
    }

    @Step("Проверка отображения ошибки в редакторе")
    public EditorPage shouldHaveError() {
        errorMessages.shouldBe(Condition.visible);
        return this;
    }
}
```

### 3.6 ArticlePage

```java
package ui.pages;

import com.codeborne.selenide.Condition;
import com.codeborne.selenide.ElementsCollection;
import com.codeborne.selenide.SelenideElement;
import io.qameta.allure.Step;

import static com.codeborne.selenide.Selenide.$;
import static com.codeborne.selenide.Selenide.$$;

/**
 * Page Object для страницы просмотра статьи (/article/:slug).
 */
public class ArticlePage extends BasePage {

    private final SelenideElement articleTitle = $("h1");
    private final SelenideElement articleBody = $(".article-content");
    private final SelenideElement authorName = $(".article-meta a[href^='#/@']");
    private final SelenideElement editButton = $("a[href*='/editor/']");
    private final SelenideElement deleteButton = $("button.btn-outline-danger");
    private final SelenideElement followButton = $("button.btn-outline-secondary");
    private final SelenideElement favoriteButton = $("button.btn-outline-primary");

    // Комментарии
    private final SelenideElement commentTextarea = $("textarea[placeholder='Write a comment...']");
    private final SelenideElement postCommentButton = $("button[type='submit']");
    private final ElementsCollection commentCards = $$(".card:not(.comment-form)");
    private final ElementsCollection tagList = $$(".tag-list .tag-default");

    @Step("Проверка заголовка статьи: {expectedTitle}")
    public ArticlePage shouldHaveTitle(String expectedTitle) {
        articleTitle.shouldHave(Condition.text(expectedTitle));
        return this;
    }

    @Step("Проверка тела статьи содержит: {expectedText}")
    public ArticlePage shouldHaveBodyContaining(String expectedText) {
        articleBody.shouldHave(Condition.text(expectedText));
        return this;
    }

    @Step("Проверка имени автора: {expectedAuthor}")
    public ArticlePage shouldHaveAuthor(String expectedAuthor) {
        authorName.shouldHave(Condition.text(expectedAuthor));
        return this;
    }

    @Step("Проверка наличия тега: {tagName}")
    public ArticlePage shouldHaveTag(String tagName) {
        tagList.findBy(Condition.text(tagName)).shouldBe(Condition.visible);
        return this;
    }

    @Step("Нажатие кнопки Edit Article")
    public EditorPage clickEdit() {
        editButton.click();
        return new EditorPage();
    }

    @Step("Нажатие кнопки Delete Article")
    public HomePage clickDelete() {
        deleteButton.click();
        return new HomePage();
    }

    @Step("Проверка: кнопка Edit Article не видна")
    public ArticlePage shouldNotHaveEditButton() {
        editButton.shouldNot(Condition.exist);
        return this;
    }

    @Step("Проверка: кнопка Delete Article не видна")
    public ArticlePage shouldNotHaveDeleteButton() {
        deleteButton.shouldNot(Condition.exist);
        return this;
    }

    // ========== Комментарии ==========

    @Step("Ввод текста комментария: {commentText}")
    public ArticlePage typeComment(String commentText) {
        commentTextarea.setValue(commentText);
        return this;
    }

    @Step("Нажатие кнопки Post Comment")
    public ArticlePage clickPostComment() {
        postCommentButton.click();
        return this;
    }

    @Step("Добавление комментария: {commentText}")
    public ArticlePage addComment(String commentText) {
        typeComment(commentText);
        clickPostComment();
        return this;
    }

    @Step("Проверка: комментарий отображается с текстом {expectedText}")
    public ArticlePage shouldHaveComment(String expectedText) {
        commentCards.findBy(Condition.text(expectedText)).shouldBe(Condition.visible);
        return this;
    }

    @Step("Удаление первого комментария")
    public ArticlePage deleteFirstComment() {
        commentCards.first().$("i.ion-trash-a").click();
        return this;
    }

    @Step("Проверка: количество комментариев = {count}")
    public ArticlePage shouldHaveCommentsCount(int count) {
        if (count == 0) {
            commentCards.shouldHave(com.codeborne.selenide.CollectionCondition.size(0));
        } else {
            commentCards.shouldHave(
                    com.codeborne.selenide.CollectionCondition.size(count));
        }
        return this;
    }

    // ========== Подписки и лайки ==========

    @Step("Нажатие кнопки Follow/Unfollow")
    public ArticlePage clickFollow() {
        followButton.click();
        return this;
    }

    @Step("Проверка: кнопка показывает Unfollow")
    public ArticlePage shouldShowUnfollow() {
        followButton.shouldHave(Condition.text("Unfollow"));
        return this;
    }

    @Step("Проверка: кнопка показывает Follow")
    public ArticlePage shouldShowFollow() {
        followButton.shouldHave(Condition.text("Follow"));
        return this;
    }

    @Step("Нажатие кнопки Favorite")
    public ArticlePage clickFavorite() {
        favoriteButton.click();
        return this;
    }

    /**
     * Возвращает текущий заголовок статьи.
     */
    public String getTitle() {
        return articleTitle.getText();
    }
}
```

### 3.7 ProfilePage

```java
package ui.pages;

import com.codeborne.selenide.CollectionCondition;
import com.codeborne.selenide.Condition;
import com.codeborne.selenide.ElementsCollection;
import com.codeborne.selenide.SelenideElement;
import io.qameta.allure.Step;

import static com.codeborne.selenide.Selenide.*;

/**
 * Page Object для профиля пользователя (/@username).
 */
public class ProfilePage extends BasePage {

    private final SelenideElement profileUsername = $("h4");
    private final SelenideElement profileBio = $(".user-info p");
    private final SelenideElement editProfileButton = $("a[href='#/settings']");
    private final SelenideElement followButton = $("button.action-btn");
    private final SelenideElement myArticlesTab = $(".articles-toggle a:first-child");
    private final SelenideElement favoritedArticlesTab = $(".articles-toggle a:last-child");
    private final ElementsCollection articlePreviews = $$(".article-preview");

    @Step("Открытие профиля пользователя: {username}")
    public ProfilePage openProfile(String username) {
        open("/#/@" + username);
        profileUsername.shouldBe(Condition.visible);
        return this;
    }

    @Step("Проверка имени пользователя в профиле: {expectedUsername}")
    public ProfilePage shouldHaveUsername(String expectedUsername) {
        profileUsername.shouldHave(Condition.text(expectedUsername));
        return this;
    }

    @Step("Проверка bio в профиле: {expectedBio}")
    public ProfilePage shouldHaveBio(String expectedBio) {
        profileBio.shouldHave(Condition.text(expectedBio));
        return this;
    }

    @Step("Переход на вкладку My Articles")
    public ProfilePage clickMyArticles() {
        myArticlesTab.click();
        return this;
    }

    @Step("Переход на вкладку Favorited Articles")
    public ProfilePage clickFavoritedArticles() {
        favoritedArticlesTab.click();
        return this;
    }

    @Step("Проверка: список статей не пуст")
    public ProfilePage shouldHaveArticles() {
        articlePreviews.shouldHave(CollectionCondition.sizeGreaterThan(0));
        return this;
    }

    @Step("Проверка: статья с заголовком {title} видна в списке")
    public ProfilePage shouldHaveArticleWithTitle(String title) {
        articlePreviews.findBy(Condition.text(title)).shouldBe(Condition.visible);
        return this;
    }

    @Step("Нажатие кнопки Follow/Unfollow в профиле")
    public ProfilePage clickFollow() {
        followButton.click();
        return this;
    }

    @Step("Нажатие кнопки Edit Profile Settings")
    public SettingsPage clickEditProfile() {
        editProfileButton.click();
        return new SettingsPage();
    }
}
```

### 3.8 SettingsPage

```java
package ui.pages;

import com.codeborne.selenide.Condition;
import com.codeborne.selenide.SelenideElement;
import io.qameta.allure.Step;

import static com.codeborne.selenide.Selenide.$;
import static com.codeborne.selenide.Selenide.open;

/**
 * Page Object для страницы настроек профиля (/settings).
 */
public class SettingsPage extends BasePage {

    private final SelenideElement imageUrlInput =
            $("input[placeholder='URL of profile picture']");
    private final SelenideElement usernameInput =
            $("input[placeholder='Username']");
    private final SelenideElement bioTextarea =
            $("textarea[placeholder='Short bio about you']");
    private final SelenideElement emailInput =
            $("input[placeholder='Email']");
    private final SelenideElement passwordInput =
            $("input[placeholder='New Password']");
    private final SelenideElement updateButton =
            $("button[type='submit']");
    private final SelenideElement logoutButton =
            $("button.btn-outline-danger");

    @Step("Открытие страницы настроек")
    public SettingsPage openPage() {
        open("/#/settings");
        updateButton.shouldBe(Condition.visible);
        return this;
    }

    @Step("Ввод URL аватара: {imageUrl}")
    public SettingsPage typeImageUrl(String imageUrl) {
        imageUrlInput.clear();
        imageUrlInput.setValue(imageUrl);
        return this;
    }

    @Step("Ввод нового username: {username}")
    public SettingsPage typeUsername(String username) {
        usernameInput.clear();
        usernameInput.setValue(username);
        return this;
    }

    @Step("Ввод bio: {bio}")
    public SettingsPage typeBio(String bio) {
        bioTextarea.clear();
        bioTextarea.setValue(bio);
        return this;
    }

    @Step("Ввод нового email: {email}")
    public SettingsPage typeEmail(String email) {
        emailInput.clear();
        emailInput.setValue(email);
        return this;
    }

    @Step("Ввод нового пароля")
    public SettingsPage typePassword(String password) {
        passwordInput.setValue(password);
        return this;
    }

    @Step("Нажатие кнопки Update Settings")
    public SettingsPage clickUpdate() {
        updateButton.click();
        return this;
    }

    @Step("Нажатие кнопки Logout")
    public HomePage clickLogout() {
        logoutButton.click();
        return new HomePage();
    }

    @Step("Обновление bio и проверка")
    public SettingsPage updateBio(String newBio) {
        typeBio(newBio);
        clickUpdate();
        return this;
    }
}
```

---

## Часть 4: Тесты регистрации и логина

### 4.1 RegisterUiTest

```java
package ui.tests;

import io.qameta.allure.*;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import ui.pages.HomePage;
import ui.pages.RegisterPage;
import utils.DataGenerator;

@Epic("Conduit Blog Platform")
@Feature("Registration")
@DisplayName("UI: Регистрация")
public class RegisterUiTest extends BaseUiTest {

    @Test
    @Story("Successful Registration")
    @Severity(SeverityLevel.BLOCKER)
    @DisplayName("Успешная регистрация нового пользователя")
    void shouldRegisterNewUser() {
        String username = DataGenerator.randomUsername();
        String email = DataGenerator.randomEmail();
        String password = DataGenerator.randomPassword();

        HomePage homePage = new RegisterPage()
                .openPage()
                .registerAs(username, email, password);

        // Проверяем, что пользователь авторизован
        homePage.shouldBeLoggedIn();
        // Проверяем, что имя пользователя в навигации
        org.assertj.core.api.Assertions.assertThat(homePage.getLoggedInUsername())
                .isEqualTo(username);
    }

    @Test
    @Story("Duplicate Registration")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("Регистрация с дублирующим email показывает ошибку")
    void shouldShowErrorForDuplicateEmail() {
        String email = DataGenerator.randomEmail();
        String password = DataGenerator.randomPassword();

        // Регистрируем первого пользователя через API (быстрее)
        api.client.AuthApi.register(
                new api.models.request.RegisterRequest(
                        DataGenerator.randomUsername(), email, password))
                .then().statusCode(200);

        // Пытаемся зарегистрировать второго с тем же email через UI
        new RegisterPage()
                .openPage()
                .typeUsername(DataGenerator.randomUsername())
                .typeEmail(email)
                .typePassword(password)
                .clickSignUp()
                .shouldHaveError();
    }

    @Test
    @Story("Empty Fields")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("Регистрация с пустыми полями показывает ошибки")
    void shouldShowErrorForEmptyFields() {
        new RegisterPage()
                .openPage()
                .clickSignUp()
                .shouldHaveError();
    }
}
```

### 4.2 LoginUiTest

```java
package ui.tests;

import api.client.AuthApi;
import api.models.request.RegisterRequest;
import io.qameta.allure.*;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import ui.pages.HomePage;
import ui.pages.LoginPage;
import utils.DataGenerator;

@Epic("Conduit Blog Platform")
@Feature("Authentication")
@DisplayName("UI: Авторизация")
public class LoginUiTest extends BaseUiTest {

    @Test
    @Story("Successful Login")
    @Severity(SeverityLevel.BLOCKER)
    @DisplayName("Успешный логин зарегистрированного пользователя")
    void shouldLoginWithValidCredentials() {
        // Подготовка: регистрируем пользователя через API
        String username = DataGenerator.randomUsername();
        String email = DataGenerator.randomEmail();
        String password = DataGenerator.randomPassword();
        AuthApi.register(new RegisterRequest(username, email, password))
                .then().statusCode(200);

        // Логинимся через UI
        HomePage homePage = new LoginPage()
                .openPage()
                .loginAs(email, password);

        homePage.shouldBeLoggedIn();
        org.assertj.core.api.Assertions.assertThat(homePage.getLoggedInUsername())
                .isEqualTo(username);
    }

    @Test
    @Story("Failed Login")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("Логин с неверным паролем показывает ошибку")
    void shouldShowErrorForWrongPassword() {
        new LoginPage()
                .openPage()
                .typeEmail(DataGenerator.randomEmail())
                .typePassword("wrongpassword")
                .clickSignIn()
                .shouldHaveError();
    }

    @Test
    @Story("Failed Login")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("Логин с пустыми полями показывает ошибку")
    void shouldShowErrorForEmptyFields() {
        new LoginPage()
                .openPage()
                .clickSignIn()
                .shouldHaveError();
    }

    @Test
    @Story("Navigation")
    @Severity(SeverityLevel.MINOR)
    @DisplayName("Ссылка Need an account ведёт на страницу регистрации")
    void shouldNavigateToRegisterPage() {
        new LoginPage()
                .openPage()
                .clickSignUpLink();
        // Проверяем, что мы на странице регистрации
        com.codeborne.selenide.Selenide.$("input[placeholder='Username']")
                .shouldBe(com.codeborne.selenide.Condition.visible);
    }
}
```

---

## Часть 5: Тесты статей

```java
package ui.tests;

import io.qameta.allure.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import ui.pages.ArticlePage;
import ui.pages.EditorPage;
import ui.pages.HomePage;
import utils.DataGenerator;

@Epic("Conduit Blog Platform")
@Feature("Articles")
@DisplayName("UI: Статьи")
public class ArticleUiTest extends BaseUiTest {

    @BeforeEach
    void setUp() {
        // Логинимся через API для ускорения
        loginViaApi();
    }

    @Test
    @Story("Create Article")
    @Severity(SeverityLevel.BLOCKER)
    @DisplayName("Создание статьи со всеми полями")
    void shouldCreateArticle() {
        String title = DataGenerator.randomTitle();
        String description = DataGenerator.randomDescription();
        String body = DataGenerator.randomBody();
        String tag = DataGenerator.randomTag();

        ArticlePage articlePage = new EditorPage()
                .openPage()
                .createArticle(title, description, body, tag);

        articlePage
                .shouldHaveTitle(title)
                .shouldHaveBodyContaining(body)
                .shouldHaveTag(tag);
    }

    @Test
    @Story("Edit Article")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("Редактирование заголовка и тела статьи")
    void shouldEditArticle() {
        // Создаём статью
        String originalTitle = DataGenerator.randomTitle();
        ArticlePage articlePage = new EditorPage()
                .openPage()
                .createArticle(originalTitle, DataGenerator.randomDescription(),
                        DataGenerator.randomBody());

        // Редактируем
        String newTitle = "Updated: " + DataGenerator.randomTitle();
        String newBody = "Updated body: " + DataGenerator.randomBody();

        EditorPage editorPage = articlePage.clickEdit();
        ArticlePage updatedPage = editorPage
                .clearAndTypeTitle(newTitle)
                .clearAndTypeBody(newBody)
                .clickPublish();

        updatedPage.shouldHaveTitle(newTitle);
        updatedPage.shouldHaveBodyContaining(newBody);
    }

    @Test
    @Story("Delete Article")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("Удаление своей статьи")
    void shouldDeleteArticle() {
        // Создаём статью
        ArticlePage articlePage = new EditorPage()
                .openPage()
                .createArticle(DataGenerator.randomTitle(),
                        DataGenerator.randomDescription(),
                        DataGenerator.randomBody());

        // Удаляем — должны оказаться на главной странице
        HomePage homePage = articlePage.clickDelete();
        homePage.shouldBeLoggedIn();
    }

    @Test
    @Story("Create Article")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("Создание статьи с несколькими тегами")
    void shouldCreateArticleWithMultipleTags() {
        String title = DataGenerator.randomTitle();
        String tag1 = "java";
        String tag2 = "testing";

        ArticlePage articlePage = new EditorPage()
                .openPage()
                .createArticle(title, DataGenerator.randomDescription(),
                        DataGenerator.randomBody(), tag1, tag2);

        articlePage
                .shouldHaveTitle(title)
                .shouldHaveTag(tag1)
                .shouldHaveTag(tag2);
    }
}
```

---

## Часть 6: Тесты комментариев, подписок и профиля

### 6.1 CommentUiTest

```java
package ui.tests;

import io.qameta.allure.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import ui.pages.ArticlePage;
import ui.pages.EditorPage;
import utils.DataGenerator;

@Epic("Conduit Blog Platform")
@Feature("Comments")
@DisplayName("UI: Комментарии")
public class CommentUiTest extends BaseUiTest {

    private ArticlePage articlePage;

    @BeforeEach
    void setUp() {
        loginViaApi();

        // Создаём статью для комментирования
        articlePage = new EditorPage()
                .openPage()
                .createArticle(DataGenerator.randomTitle(),
                        DataGenerator.randomDescription(),
                        DataGenerator.randomBody());
    }

    @Test
    @Story("Add Comment")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("Добавление комментария к статье")
    void shouldAddComment() {
        String commentText = DataGenerator.randomComment();

        articlePage
                .addComment(commentText)
                .shouldHaveComment(commentText);
    }

    @Test
    @Story("Delete Comment")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("Удаление своего комментария")
    void shouldDeleteComment() {
        String commentText = DataGenerator.randomComment();

        articlePage
                .addComment(commentText)
                .shouldHaveComment(commentText)
                .deleteFirstComment()
                .shouldHaveCommentsCount(0);
    }

    @Test
    @Story("Add Comment")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("Добавление нескольких комментариев к статье")
    void shouldAddMultipleComments() {
        String comment1 = DataGenerator.randomComment();
        String comment2 = DataGenerator.randomComment();

        articlePage
                .addComment(comment1)
                .addComment(comment2)
                .shouldHaveCommentsCount(2);
    }
}
```

### 6.2 ProfileUiTest

```java
package ui.tests;

import io.qameta.allure.*;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import ui.pages.EditorPage;
import ui.pages.ProfilePage;
import ui.pages.SettingsPage;
import utils.DataGenerator;

@Epic("Conduit Blog Platform")
@Feature("Profile")
@DisplayName("UI: Профиль пользователя")
public class ProfileUiTest extends BaseUiTest {

    @Test
    @Story("View Profile")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("Просмотр своего профиля с созданными статьями")
    void shouldViewOwnProfile() {
        String username = loginViaApiAndGetUsername();

        // Создаём статью
        String title = DataGenerator.randomTitle();
        new EditorPage()
                .openPage()
                .createArticle(title, DataGenerator.randomDescription(),
                        DataGenerator.randomBody());

        // Открываем профиль
        new ProfilePage()
                .openProfile(username)
                .shouldHaveUsername(username)
                .clickMyArticles()
                .shouldHaveArticleWithTitle(title);
    }

    @Test
    @Story("Edit Profile")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("Обновление bio в настройках")
    void shouldUpdateBio() {
        String username = loginViaApiAndGetUsername();
        String newBio = "QA Engineer with " + System.currentTimeMillis();

        new SettingsPage()
                .openPage()
                .updateBio(newBio);

        // Проверяем в профиле
        new ProfilePage()
                .openProfile(username)
                .shouldHaveBio(newBio);
    }

    @Test
    @Story("Logout")
    @Severity(SeverityLevel.CRITICAL)
    @DisplayName("Выход из аккаунта через настройки")
    void shouldLogout() {
        loginViaApi();

        new SettingsPage()
                .openPage()
                .clickLogout()
                .shouldBeLoggedOut();
    }

    @Test
    @Story("Follow User")
    @Severity(SeverityLevel.NORMAL)
    @DisplayName("Подписка и отписка от другого пользователя")
    void shouldFollowAndUnfollowUser() {
        loginViaApi();

        // Создаём второго пользователя через API
        String targetUsername = DataGenerator.randomUsername();
        api.client.AuthApi.registerAndGetToken(
                targetUsername, DataGenerator.randomEmail(),
                DataGenerator.randomPassword());

        // Подписываемся через профиль
        ProfilePage profilePage = new ProfilePage()
                .openProfile(targetUsername);

        profilePage.clickFollow();

        // Отписываемся
        profilePage.clickFollow();
    }
}
```

---

## Часть 7: Параллельный запуск тестов

### 7.1 Конфигурация JUnit 5 для параллельного запуска

Создайте файл `src/test/resources/junit-platform.properties`:

```properties
# Включаем параллельный запуск тестов
junit.jupiter.execution.parallel.enabled=true

# Стратегия: CONCURRENT — тесты внутри одного класса запускаются параллельно
# SAME_THREAD — тесты внутри класса запускаются последовательно
junit.jupiter.execution.parallel.mode.default=same_thread

# Тестовые классы запускаются параллельно между собой
junit.jupiter.execution.parallel.mode.classes.default=concurrent

# Фиксированное количество потоков (= количество ядер процессора)
junit.jupiter.execution.parallel.config.strategy=fixed
junit.jupiter.execution.parallel.config.fixed.parallelism=4
```

### 7.2 Потокобезопасность Page Objects

Selenide по умолчанию потокобезопасен: каждый поток получает свой экземпляр WebDriver.
Однако нужно убедиться, что:

1. **Тестовые данные уникальны** -- `DataGenerator` генерирует уникальные значения с timestamp
2. **Нет общего состояния** -- каждый тест создаёт своего пользователя
3. **Браузер закрывается** -- `@AfterEach` вызывает `Selenide.closeWebDriver()`

### 7.3 Запуск параллельных тестов

```bash
# Запуск UI-тестов параллельно (4 потока по умолчанию)
mvn clean test -Dgroups=ui

# С указанием количества потоков
mvn clean test -Dgroups=ui \
    -Djunit.jupiter.execution.parallel.config.fixed.parallelism=6

# Headless-режим для CI
mvn clean test -Dgroups=ui -Dheadless=true
```

---

## Часть 8: Скриншоты при падении

### 8.1 Встроенные скриншоты Selenide + Allure

Мы уже настроили AllureSelenide listener в `BaseUiTest.setUpAll()`:

```java
SelenideLogger.addListener("AllureSelenide",
        new AllureSelenide()
                .screenshots(true)           // Скриншот при падении
                .savePageSource(true)         // HTML-код страницы
                .includeSelenideSteps(true)); // Все действия Selenide как шаги Allure
```

Этого достаточно: при падении теста Allure автоматически получит скриншот и page source.

### 8.2 Ручное прикрепление скриншота (опционально)

Если нужно сделать скриншот в определённый момент (не только при падении):

```java
package utils;

import com.codeborne.selenide.Selenide;
import io.qameta.allure.Allure;
import io.qameta.allure.Step;
import org.openqa.selenium.OutputType;

import java.io.ByteArrayInputStream;

/**
 * Утилиты для UI-тестов: скриншоты, ожидания, отладка.
 */
public class UiHelper {

    @Step("Делаем скриншот: {name}")
    public static void takeScreenshot(String name) {
        byte[] screenshot = Selenide.screenshot(OutputType.BYTES);
        if (screenshot != null) {
            Allure.addAttachment(name, "image/png",
                    new ByteArrayInputStream(screenshot), ".png");
        }
    }

    @Step("Сохраняем page source: {name}")
    public static void savePageSource(String name) {
        String pageSource = Selenide.webdriver().driver().source();
        if (pageSource != null) {
            Allure.addAttachment(name, "text/html", pageSource, ".html");
        }
    }
}
```

---

## Часть 9: Запуск и анализ

### 9.1 Команды запуска

```bash
# Запуск всех UI-тестов
mvn clean test -Dgroups=ui

# Запуск конкретного тестового класса
mvn clean test -Dtest=ArticleUiTest

# Запуск в headless-режиме
mvn clean test -Dgroups=ui -Dheadless=true -Dbrowser.size=1920x1080

# Запуск с другим базовым URL
mvn clean test -Dgroups=ui -Dapp.base.url=https://demo.realworld.io

# Генерация Allure-отчёта
mvn allure:serve
```

### 9.2 Ожидаемое покрытие

| Сценарий | Тестовый класс | Количество тестов |
|----------|----------------|-------------------|
| Регистрация (успешная, дубли, пустые поля) | RegisterUiTest | 3 |
| Логин (успешный, неверный пароль, пустые, навигация) | LoginUiTest | 4 |
| Статьи (создание, редактирование, удаление, теги) | ArticleUiTest | 4 |
| Комментарии (добавление, удаление, несколько) | CommentUiTest | 3 |
| Профиль (просмотр, bio, логаут, подписка) | ProfileUiTest | 4 |
| **Итого** | | **18+** |

---

## Практическое задание

### Задание 1: Базовая инфраструктура

1. Создайте пакеты `ui/pages`, `ui/components`, `ui/tests`
2. Реализуйте `BaseUiTest` с настройкой Selenide и Allure
3. Реализуйте `BasePage` с общими элементами навигации
4. Проверьте, что Selenide открывает браузер: напишите тест, который открывает главную страницу

### Задание 2: Регистрация и логин

1. Реализуйте `LoginPage` и `RegisterPage`
2. Напишите тесты: успешная регистрация, логин, негативные сценарии
3. Реализуйте метод `loginViaApi()` для ускорения подготовки тестов
4. Убедитесь, что тесты проходят

### Задание 3: Статьи и комментарии

1. Реализуйте `EditorPage`, `ArticlePage`, `HomePage`
2. Напишите тесты: создание, редактирование, удаление статьи
3. Напишите тесты: добавление и удаление комментария
4. Используйте `loginViaApi()` в `@BeforeEach`

### Задание 4: Профиль и настройки

1. Реализуйте `ProfilePage` и `SettingsPage`
2. Напишите тесты: просмотр профиля, обновление bio, логаут
3. Напишите тест: подписка/отписка через страницу профиля
4. Общее количество UI-тестов должно быть не менее 15

### Задание 5: Параллельный запуск и стабильность

1. Настройте `junit-platform.properties` для параллельного запуска
2. Запустите все UI-тесты параллельно (4 потока)
3. Убедитесь, что нет конфликтов между потоками
4. Запустите в headless-режиме и проверьте стабильность

---

## Чек-лист самопроверки

- [ ] Структура проекта содержит `ui/pages`, `ui/tests`, `ui/components`
- [ ] `BaseUiTest` настраивает Selenide, AllureSelenide listener и закрывает браузер
- [ ] `BasePage` содержит общие элементы навигации и методы навигации
- [ ] Реализовано 7+ Page Objects: Login, Register, Home, Editor, Article, Profile, Settings
- [ ] Каждый Page Object использует `@Step` для всех публичных методов
- [ ] Page Objects возвращают соответствующую страницу (fluent API / method chaining)
- [ ] `loginViaApi()` используется для быстрой авторизации без UI
- [ ] Написано 18+ UI-тестов, покрывающих все ключевые сценарии
- [ ] Все тесты помечены `@Epic`, `@Feature`, `@Story`, `@Severity`
- [ ] Скриншоты автоматически прикрепляются к Allure при падении
- [ ] Параллельный запуск настроен через `junit-platform.properties`
- [ ] Тесты проходят при запуске `mvn clean test -Dgroups=ui`
- [ ] Тесты стабильны в headless-режиме
- [ ] Allure-отчёт генерируется и показывает иерархию тестов с скриншотами
- [ ] Готов перейти к этапу настройки Allure-репортинга
