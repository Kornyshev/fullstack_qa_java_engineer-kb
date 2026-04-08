# Сайты для практики UI-автоматизации

## Содержание

1. [Обзор ресурсов](#обзор-ресурсов)
2. [SauceDemo — E-commerce flow](#saucedemo--e-commerce-flow)
3. [The Internet — Изолированные элементы](#the-internet--изолированные-элементы)
4. [DemoQA — Формы и виджеты](#demoqa--формы-и-виджеты)
5. [Practice Automation — Page Object Model](#practice-automation--page-object-model)
6. [UI Testing Playground — Waits и динамические элементы](#ui-testing-playground--waits-и-динамические-элементы)
7. [Conduit — E2E Single Page Application](#conduit--e2e-single-page-application)
8. [Сводная таблица навыков](#сводная-таблица-навыков)
9. [Рекомендуемый план обучения](#рекомендуемый-план-обучения)

---

## Обзор ресурсов

| Сайт | URL | Ключевые навыки | Сложность |
|------|-----|-----------------|:---------:|
| SauceDemo | `https://www.saucedemo.com` | E2E, авторизация, корзина | Начальный |
| The Internet | `https://the-internet.herokuapp.com` | Отдельные элементы, alerts, frames | Средний |
| DemoQA | `https://demoqa.com` | Формы, виджеты, interactions | Средний |
| Practice Automation | `https://practice-automation.com` | POM, смешанные сценарии | Средний |
| UI Testing Playground | `http://uitestingplayground.com` | Waits, dynamic ID, AJAX | Продвинутый |
| Conduit | `https://demo.realworld.io` | SPA, REST + UI, регистрация | Продвинутый |

---

## SauceDemo — E-commerce flow

**URL:** [https://www.saucedemo.com](https://www.saucedemo.com)

**Описание:** Тренировочный интернет-магазин от Sauce Labs. Идеально подходит для первого проекта — простая структура, предсказуемое поведение, несколько типов пользователей.

### Учётные данные

| Логин | Пароль | Поведение |
|-------|--------|-----------|
| `standard_user` | `secret_sauce` | Нормальная работа |
| `locked_out_user` | `secret_sauce` | Заблокированный пользователь |
| `problem_user` | `secret_sauce` | Сломанные изображения, глюки |
| `performance_glitch_user` | `secret_sauce` | Медленная загрузка страниц |
| `error_user` | `secret_sauce` | Ошибки при действиях |
| `visual_user` | `secret_sauce` | Визуальные дефекты |

### Что автоматизировать

#### Авторизация
- Успешный логин с `standard_user`
- Неуспешный логин с пустыми полями
- Неуспешный логин с неверным паролем
- Попытка логина с `locked_out_user` (сообщение об ошибке)
- Логин/логаут цикл

#### Каталог товаров
- Проверка отображения 6 товаров на странице
- Сортировка: A-Z, Z-A, цена по возрастанию, цена по убыванию
- Проверка соответствия сортировки (названия, цены)
- Открытие карточки товара и проверка деталей

#### Корзина
- Добавление одного товара в корзину
- Добавление нескольких товаров
- Удаление товара из корзины
- Проверка badge-счётчика на иконке корзины
- Проверка цен в корзине

#### Оформление заказа
- Полный checkout flow: корзина → информация → подтверждение → завершение
- Валидация обязательных полей на странице Checkout
- Проверка итоговой суммы (tax + items)
- Возврат к покупкам после завершения заказа

### Конкретные задания

1. **Задание 1:** Напишите тест, который логинится, добавляет "Sauce Labs Backpack" в корзину, переходит в корзину и проверяет название и цену товара ($29.99)
2. **Задание 2:** Автоматизируйте полный checkout: логин → добавить 2 товара → checkout → заполнить данные (First Name, Last Name, Zip Code) → finish → проверить "Thank you for your order!"
3. **Задание 3:** Проверьте все виды сортировки. Для Price (low to high) убедитесь, что каждая следующая цена >= предыдущей

---

## The Internet — Изолированные элементы

**URL:** [https://the-internet.herokuapp.com](https://the-internet.herokuapp.com)

**Описание:** Коллекция страниц, каждая из которых демонстрирует отдельный элемент или паттерн. Создана Дэйвом Хэффнером (Dave Haeffner), автором Selenium. Отлично подходит для изучения работы с конкретными типами элементов.

### Ключевые страницы и задания

| Страница | URL | Что практиковать |
|----------|-----|------------------|
| A/B Testing | `/abtest` | Проверка текста на странице |
| Add/Remove Elements | `/add_remove_elements/` | Динамическое создание/удаление кнопок |
| Basic Auth | `/basic_auth` | HTTP Basic Authentication |
| Checkboxes | `/checkboxes` | Работа с чекбоксами |
| Context Menu | `/context_menu` | Правый клик, Alert |
| Disappearing Elements | `/disappearing_elements` | Элементы, появляющиеся/исчезающие |
| Drag and Drop | `/drag_and_drop` | Перетаскивание элементов |
| Dropdown | `/dropdown` | Выпадающие списки |
| Dynamic Content | `/dynamic_content` | Контент, меняющийся при перезагрузке |
| Dynamic Controls | `/dynamic_controls` | Асинхронные элементы (ожидания) |
| Dynamic Loading | `/dynamic_loading` | Lazy loading, explicit waits |
| File Download | `/download` | Скачивание файлов |
| File Upload | `/upload` | Загрузка файлов |
| Floating Menu | `/floating_menu` | Прокрутка, фиксированные элементы |
| Frames | `/frames` | iframes, nested frames |
| Hovers | `/hovers` | Наведение мыши (hover) |
| Infinite Scroll | `/infinite_scroll` | Бесконечная прокрутка |
| Inputs | `/inputs` | Поле ввода числа |
| JavaScript Alerts | `/javascript_alerts` | Alert, Confirm, Prompt |
| Key Presses | `/key_presses` | Клавиатурные события |
| Multiple Windows | `/windows` | Работа с несколькими окнами/вкладками |
| Notification Messages | `/notification_message_rendered` | Flash-сообщения |
| Secure File Download | `/download_secure` | Скачивание с Basic Auth |
| Shadow DOM | `/shadowdom` | Работа с Shadow DOM |
| Tables | `/tables` | Сортировка и парсинг таблиц |

### Конкретные задания

1. **Задание 1 — Checkboxes:** Откройте `/checkboxes`. Установите оба чекбокса. Проверьте, что оба `isSelected()` = true
2. **Задание 2 — Dropdown:** Откройте `/dropdown`. Выберите "Option 2". Проверьте, что выбранный текст = "Option 2"
3. **Задание 3 — JavaScript Alerts:** Откройте `/javascript_alerts`. Нажмите "Click for JS Prompt", введите "Test text", подтвердите. Проверьте сообщение "You entered: Test text"
4. **Задание 4 — Dynamic Loading:** Откройте `/dynamic_loading/1`. Нажмите "Start". Дождитесь появления текста "Hello World!" с помощью explicit wait
5. **Задание 5 — File Upload:** Загрузите файл через `/upload`. Проверьте, что отображается имя загруженного файла
6. **Задание 6 — Hovers:** Наведите курсор на каждый из 3 аватаров на `/hovers`. Проверьте, что для каждого появляется имя пользователя
7. **Задание 7 — Frames:** Откройте `/iframe`. Переключитесь в iframe с TinyMCE-редактором. Очистите содержимое, введите свой текст. Проверьте, что текст записан
8. **Задание 8 — Tables:** Откройте `/tables`. Прочитайте данные из таблицы. Отсортируйте по столбцу "Last Name" и проверьте порядок

---

## DemoQA — Формы и виджеты

**URL:** [https://demoqa.com](https://demoqa.com)

**Описание:** Большой тренировочный сайт с разнообразными элементами UI. Организован по категориям: Elements, Forms, Alerts/Windows, Widgets, Interactions, Book Store.

### Ключевые разделы

| Раздел | URL | Что практиковать |
|--------|-----|------------------|
| Text Box | `/text-box` | Заполнение текстовых полей |
| Check Box | `/checkbox` | Древовидный чекбокс |
| Radio Button | `/radio-button` | Радио-кнопки |
| Web Tables | `/webtables` | CRUD для таблицы |
| Buttons | `/buttons` | Одинарный, двойной, правый клик |
| Links | `/links` | Новая вкладка, API-ссылки |
| Upload/Download | `/upload-download` | Файловые операции |
| Practice Form | `/automation-practice-form` | Комплексная форма |
| Browser Windows | `/browser-windows` | Новые окна и вкладки |
| Alerts | `/alerts` | Различные типы alerts |
| Modal Dialogs | `/modal-dialogs` | Модальные окна |
| Accordian | `/accordian` | Раскрывающиеся панели |
| Auto Complete | `/auto-complete` | Автодополнение |
| Date Picker | `/date-picker` | Выбор даты |
| Slider | `/slider` | Ползунок |
| Progress Bar | `/progress-bar` | Индикатор прогресса |
| Tabs | `/tabs` | Вкладки |
| Tool Tips | `/tool-tips` | Всплывающие подсказки |
| Menu | `/menu` | Многоуровневое меню |
| Select Menu | `/select-menu` | Различные виды select |
| Sortable | `/sortable` | Сортировка drag & drop |
| Resizable | `/resizable` | Изменение размера |
| Droppable | `/droppable` | Drag and drop |
| Draggable | `/dragabble` | Перетаскивание |
| Book Store | `/books` | Регистрация, поиск, профиль |

### Конкретные задания

1. **Задание 1 — Text Box:** Заполните все поля (Full Name, Email, Current Address, Permanent Address). Нажмите Submit. Проверьте, что данные отображаются в нижнем блоке
2. **Задание 2 — Practice Form:** Заполните полную регистрационную форму:
   - First Name, Last Name
   - Email
   - Gender (Radio Button)
   - Mobile (10 цифр)
   - Date of Birth (Date Picker)
   - Subjects (Auto Complete)
   - Hobbies (Checkboxes)
   - Picture (Upload)
   - Current Address
   - State & City (Select)
   - Submit и проверьте модальное окно с данными
3. **Задание 3 — Web Tables:** Добавьте 3 записи в таблицу. Отредактируйте одну. Удалите другую. Проверьте количество строк после каждой операции
4. **Задание 4 — Buttons:** Выполните: обычный клик, двойной клик, правый клик. Проверьте появление соответствующих сообщений
5. **Задание 5 — Progress Bar:** Запустите Progress Bar. Дождитесь 100%. Нажмите Reset. Проверьте, что значение вернулось к 0%

---

## Practice Automation — Page Object Model

**URL:** [https://practice-automation.com](https://practice-automation.com)

**Описание:** Относительно новый сайт, специально спроектированный для практики автоматизации с использованием паттерна Page Object Model. Чистый дизайн, предсказуемое поведение.

### Доступные страницы

| Страница | URL | Что практиковать |
|----------|-----|------------------|
| JavaScript Delays | `/javascript-delays/` | Ожидание AJAX-ответов |
| Form Fields | `/form-fields/` | Все типы полей формы |
| Popups | `/popups/` | Модальные окна, tooltips |
| iframes | `/iframes/` | Работа с фреймами |
| Tables | `/tables/` | Парсинг и валидация таблиц |
| Calendars | `/calendars/` | Выбор даты |
| File Upload | `/file-upload/` | Загрузка файлов |
| Sliders | `/slider/` | Ползунки |
| Hover | `/hover/` | Наведение курсора |
| Ads | `/ads/` | Закрытие рекламных баннеров |

### Конкретные задания

1. **Задание 1 — Form Fields:** Создайте Page Object для страницы Form Fields. Методы: `fillName()`, `selectWater()`, `selectFavoriteColor()`, `selectAutomationTool()`, `fillEmail()`, `fillMessage()`, `submit()`. Напишите тест, заполняющий все поля
2. **Задание 2 — POM-архитектура:** Создайте BasePage с общими методами (`waitForElement()`, `clickElement()`, `setText()`). Наследуйте каждую страницу от BasePage
3. **Задание 3 — JavaScript Delays:** Нажмите "Start". Дождитесь появления текста "Liftoff!" с помощью WebDriverWait. Не используйте `Thread.sleep()`
4. **Задание 4 — Popups:** Обработайте все типы: Alert, Confirm, Prompt, Tooltip, модальное окно. Для каждого напишите отдельный тест-метод

---

## UI Testing Playground — Waits и динамические элементы

**URL:** [http://uitestingplayground.com](http://uitestingplayground.com)

**Описание:** Сайт, созданный Николаем Адволодкиным. Каждая страница демонстрирует конкретную проблему, с которой сталкиваются автоматизаторы. Акцент на ожиданиях, динамических атрибутах и антипаттернах.

### Ключевые страницы

| Страница | URL | Проблема |
|----------|-----|----------|
| Dynamic ID | `/dynamicid` | ID элемента меняется при каждой загрузке |
| Class Attribute | `/classattr` | Несколько CSS-классов, порядок меняется |
| Hidden Layers | `/hiddenlayers` | Элемент перекрыт невидимым слоем |
| Load Delay | `/loaddelay` | Долгая загрузка страницы |
| AJAX Data | `/ajax` | Данные загружаются после AJAX-запроса |
| Client Side Delay | `/clientdelay` | Задержка на стороне клиента |
| Click | `/click` | Кнопка, которую тяжело нажать (overlapping) |
| Text Input | `/textinput` | Ввод текста меняет кнопку |
| Scrollbars | `/scrollbars` | Прокрутка до элемента |
| Dynamic Table | `/dynamictable` | Таблица с меняющимися данными |
| Verify Text | `/verifytext` | Проверка текста с пробелами |
| Progress Bar | `/progressbar` | Остановка прогресс-бара на 75% |
| Visibility | `/visibility` | Элементы, скрытые разными способами |
| Sample App | `/sampleapp` | Простое приложение с логином |
| Mouse Over | `/mouseover` | Двойной клик при наведении |
| Non-Breaking Space | `/nbsp` | Поиск элемента с неразрывным пробелом |
| Overlapped Element | `/overlapped` | Элемент частично перекрыт |
| Shadow DOM | `/shadowdom` | Элементы внутри Shadow DOM |

### Конкретные задания

1. **Задание 1 — Dynamic ID:** Нажмите кнопку "Button with Dynamic ID". Найдите её без использования ID (по тексту, CSS-классу). Убедитесь, что при перезагрузке тест не ломается
2. **Задание 2 — AJAX Data:** Нажмите "Button Triggering AJAX Request". Дождитесь появления зелёного сообщения "Data loaded with AJAX get request." Используйте explicit wait с таймаутом 20 секунд
3. **Задание 3 — Client Side Delay:** Нажмите "Button Triggering Client Side Logic". Дождитесь сообщения. Измерьте реальное время ожидания
4. **Задание 4 — Progress Bar:** Нажмите "Start". Остановите прогресс-бар как можно ближе к 75%. Проверьте значение. Это упражнение на точность ожиданий
5. **Задание 5 — Visibility:** Нажмите "Hide". Проверьте каждую кнопку: какие стали `display:none`, какие `visibility:hidden`, какие `opacity:0`, какие удалены из DOM. Это важно для понимания различий
6. **Задание 6 — Sample App:** Логин с `"username" / "pwd"`. Проверьте сообщение "Welcome, username!". Логаут. Проверьте "User logged out."
7. **Задание 7 — Dynamic Table:** Найдите значение CPU для процесса "Chrome". Это значение меняется при каждой загрузке — используйте парсинг таблицы, а не hardcoded-значение

---

## Conduit — E2E Single Page Application

**URL:** [https://demo.realworld.io](https://demo.realworld.io)

**Описание:** Полноценный клон Medium, реализованный как Single Page Application. Отличная практика для E2E-сценариев: регистрация, авторизация, создание статей, комментарии, подписки. Работает через REST API.

### Функциональность

| Функция | Описание |
|---------|----------|
| Регистрация | Sign Up с уникальным username, email, password |
| Авторизация | Sign In с email и password |
| Статьи | Создание, редактирование, удаление статей |
| Комментарии | Добавление и удаление комментариев |
| Теги | Фильтрация статей по тегам |
| Лента | Global Feed, Your Feed, Tag Feed |
| Профиль | Просмотр и редактирование профиля |
| Подписки | Follow/Unfollow авторов |
| Лайки | Favorite/Unfavorite статей |

### Конкретные задания

1. **Задание 1 — Регистрация:** Автоматизируйте регистрацию с уникальными данными (добавьте timestamp к username). Проверьте, что пользователь вошёл в систему (имя в навбаре)
2. **Задание 2 — Логин + Создание статьи:** Войдите, создайте статью (Title, Description, Body, Tags). Проверьте, что статья появилась в "Your Feed"
3. **Задание 3 — CRUD статьи:** Создайте статью → отредактируйте → проверьте изменения → удалите → проверьте отсутствие
4. **Задание 4 — Комментарии:** Перейдите к любой статье. Добавьте комментарий. Проверьте, что он отображается. Удалите комментарий
5. **Задание 5 — Фильтрация по тегам:** Кликните на тег в правой панели. Убедитесь, что статьи в ленте содержат выбранный тег
6. **Задание 6 — SPA-навигация:** Проверьте, что при навигации между страницами URL обновляется, но страница не перезагружается (проверьте через отсутствие полной перезагрузки DOM)

---

## Сводная таблица навыков

| Навык | SauceDemo | The Internet | DemoQA | Practice Auto | UI Playground | Conduit |
|-------|:---------:|:------------:|:------:|:------------:|:-------------:|:-------:|
| Логин/регистрация | + | + | - | - | + | + |
| Формы | + | - | + | + | + | + |
| Таблицы | - | + | + | + | + | - |
| Drag & Drop | - | + | + | - | - | - |
| Alerts/Popups | - | + | + | + | - | - |
| Frames | - | + | + | + | + | - |
| File Upload | - | + | + | + | - | - |
| Waits/Dynamic | - | + | - | + | + | + |
| Навигация | + | - | - | - | - | + |
| E2E Сценарии | + | - | - | - | - | + |
| Shadow DOM | - | + | - | - | + | - |
| Hover | - | + | - | + | + | - |
| Keyboard | - | + | - | - | - | - |

---

## Рекомендуемый план обучения

### Неделя 1: Основы
- **SauceDemo**: логин, каталог, корзина
- **The Internet**: checkboxes, dropdown, alerts

### Неделя 2: Формы и элементы
- **DemoQA**: text box, practice form, web tables
- **The Internet**: file upload, hovers, frames

### Неделя 3: Ожидания и динамика
- **UI Testing Playground**: dynamic ID, AJAX data, client delay, progress bar
- **The Internet**: dynamic loading, dynamic controls

### Неделя 4: Полноценные проекты
- **SauceDemo**: полный E2E (логин → каталог → корзина → checkout)
- **Conduit**: регистрация, создание статьи, комментарии

### Неделя 5: Page Object Model и рефакторинг
- **Practice Automation**: POM для всех страниц
- Рефакторинг проекта SauceDemo с полноценным POM

### Неделя 6: Портфолио
- Оформите проект SauceDemo с Allure-отчётом
- Выложите на GitHub с README
- Добавьте CI/CD через GitHub Actions
