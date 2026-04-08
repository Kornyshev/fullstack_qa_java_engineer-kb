# Стратегии выбора локаторов

## Обзор

Локатор — это механизм поиска элемента на веб-странице. Правильный выбор стратегии локаторов напрямую влияет
на стабильность, читаемость и поддерживаемость автотестов. Нестабильные локаторы — причина номер один "мигающих"
(flaky) тестов в UI-автоматизации. QA-инженер должен уверенно владеть CSS Selectors, XPath, понимать приоритеты
выбора локаторов и уметь строить устойчивые к изменениям выражения.

---

## CSS Selectors

### Базовый синтаксис

| Селектор | Описание | Пример |
|---|---|---|
| `tag` | По имени тега | `input` |
| `#id` | По атрибуту `id` | `#login-btn` |
| `.class` | По имени класса | `.btn-primary` |
| `[attr]` | По наличию атрибута | `[data-testid]` |
| `[attr=value]` | По точному значению атрибута | `[type="submit"]` |
| `[attr*=value]` | Атрибут содержит подстроку | `[class*="active"]` |
| `[attr^=value]` | Атрибут начинается с | `[id^="user-"]` |
| `[attr$=value]` | Атрибут заканчивается на | `[href$=".pdf"]` |

### Комбинаторы

```
/* Потомок (любой уровень вложенности) */
div .error-message

/* Прямой потомок (дочерний элемент) */
form > input

/* Соседний элемент (непосредственно следующий) */
label + input

/* Все последующие соседи */
h2 ~ p
```

### Pseudo-классы

```java
// Первый дочерний элемент
driver.findElement(By.cssSelector("ul > li:first-child"));

// Последний дочерний элемент
driver.findElement(By.cssSelector("ul > li:last-child"));

// N-й дочерний элемент (нумерация с 1)
driver.findElement(By.cssSelector("tr:nth-child(3)"));

// N-й элемент с конца
driver.findElement(By.cssSelector("tr:nth-last-child(2)"));

// Элемент, не соответствующий селектору
driver.findElement(By.cssSelector("input:not([disabled])"));

// Пустой элемент (без дочерних узлов)
driver.findElement(By.cssSelector("div:empty"));
```

### Составные селекторы

```java
// Элемент с несколькими классами
driver.findElement(By.cssSelector(".btn.btn-primary.active"));

// Тег + класс + атрибут
driver.findElement(By.cssSelector("input.form-control[name='email']"));

// Цепочка потомков
driver.findElement(By.cssSelector("div.container > form#login input[type='text']"));
```

---

## XPath

### Базовый синтаксис

| Выражение | Описание | Пример |
|---|---|---|
| `//tag` | Поиск по тегу в любом месте DOM | `//input` |
| `/html/body/div` | Абсолютный путь от корня | (хрупкий, избегать) |
| `//tag[@attr='val']` | По атрибуту | `//input[@type='text']` |
| `//tag[text()='val']` | По тексту элемента | `//button[text()='Submit']` |
| `//tag[contains(@attr,'val')]` | Атрибут содержит подстроку | `//div[contains(@class,'error')]` |
| `//tag[starts-with(@attr,'val')]` | Атрибут начинается с | `//input[starts-with(@id,'user')]` |
| `//tag[normalize-space()='val']` | Текст без лишних пробелов | `//span[normalize-space()='OK']` |

### Оси (Axes)

Оси — мощный инструмент XPath, позволяющий перемещаться по дереву DOM относительно текущего узла.

```java
// ancestor — все предки (родитель, дед и т.д.)
// Найти форму, содержащую кнопку "Submit"
driver.findElement(By.xpath("//button[text()='Submit']/ancestor::form"));

// descendant — все потомки (на любом уровне вложенности)
// Найти все input внутри конкретной формы
driver.findElements(By.xpath("//form[@id='registration']//descendant::input"));

// parent — непосредственный родитель
// Найти родительский div для элемента с ошибкой
driver.findElement(By.xpath("//span[@class='error']/parent::div"));

// following-sibling — все следующие соседи на том же уровне
// Найти ячейку таблицы, следующую за ячейкой с текстом "Цена"
driver.findElement(By.xpath("//td[text()='Price']/following-sibling::td[1]"));

// preceding-sibling — все предшествующие соседи
// Найти label, стоящий перед input с ошибкой
driver.findElement(By.xpath("//input[@class='error']/preceding-sibling::label"));

// following — все узлы после текущего в DOM
driver.findElement(By.xpath("//h2[text()='Section']/following::p[1]"));

// preceding — все узлы перед текущим в DOM
driver.findElement(By.xpath("//footer/preceding::div[1]"));

// self — текущий узел (полезен в комбинациях)
driver.findElement(By.xpath("//div[@class='wrapper']/self::div"));
```

### Логические операторы

```java
// AND — оба условия
driver.findElement(By.xpath("//input[@type='text' and @name='username']"));

// OR — хотя бы одно условие
driver.findElement(By.xpath("//input[@type='submit' or @type='button']"));

// NOT — отрицание
driver.findElement(By.xpath("//input[not(@disabled)]"));

// Комбинация условий
driver.findElement(By.xpath("//div[contains(@class,'item') and not(contains(@class,'hidden'))]"));
```

---

## data-testid атрибуты

Специальные атрибуты, добавляемые разработчиками исключительно для целей тестирования. Это золотой стандарт
для стабильных локаторов.

### Преимущества

- Не меняются при рефакторинге стилей или структуры
- Явно указывают на тестовое назначение
- Не затрагиваются CSS-фреймворками и сборщиками
- Легко читаются и понятны всей команде

### Соглашения об именовании

```html
<!-- Хорошо: описательные, уникальные имена -->
<button data-testid="login-submit-btn">Войти</button>
<input data-testid="email-input" type="email" />
<div data-testid="user-profile-card">...</div>

<!-- Плохо: бессмысленные или дублирующиеся имена -->
<button data-testid="btn1">Войти</button>
<input data-testid="input" type="email" />
```

### Использование в Selenium

```java
// Поиск по data-testid с CSS
driver.findElement(By.cssSelector("[data-testid='login-submit-btn']"));

// Поиск по data-testid с XPath
driver.findElement(By.xpath("//*[@data-testid='login-submit-btn']"));
```

### Удаление из production-сборки

Многие проекты удаляют `data-testid` из production-кода с помощью плагинов сборки (например,
`babel-plugin-react-remove-properties`), чтобы не увеличивать размер HTML. В этом случае
атрибуты доступны только в dev/staging-окружениях.

---

## Приоритет стратегий выбора локаторов

Рекомендуемый порядок предпочтения (от лучшего к худшему):

| Приоритет | Стратегия | Обоснование |
|---|---|---|
| 1 | `By.id("uniqueId")` | Уникальный по спецификации HTML, самый быстрый |
| 2 | `By.cssSelector("[data-testid='name']")` | Стабильный, предназначен для тестирования |
| 3 | `By.name("fieldName")` | Стабильный для форм |
| 4 | `By.cssSelector(...)` | Быстрый, компактный, покрывает большинство случаев |
| 5 | `By.xpath(...)` | Гибкий, но медленнее и сложнее читается |
| 6 | `By.linkText("text")` / `By.partialLinkText("text")` | Только для ссылок, зависит от текста (i18n!) |
| 7 | `By.className("cls")` | Часто неуникальный, меняется при редизайне |
| 8 | `By.tagName("tag")` | Слишком общий, редко полезен |

---

## Сравнение CSS Selectors и XPath

| Критерий | CSS Selectors | XPath |
|---|---|---|
| Скорость | Быстрее (нативная поддержка браузеров) | Медленнее (парсинг выражения) |
| Читаемость | Компактный синтаксис | Более многословный |
| Навигация вверх по DOM | Невозможна | `parent::`, `ancestor::` |
| Поиск по тексту | Невозможен | `text()`, `contains(text(),'...')` |
| Поиск по индексу | `:nth-child()` (от 1) | `[position()]` (от 1) |
| Поддержка в браузерах | Отличная | Отличная |
| Поиск по частичному совпадению | `*=`, `^=`, `$=` | `contains()`, `starts-with()` |
| Логические операторы | `:not()` | `and`, `or`, `not()` |
| Регулярные выражения | Нет | Только XPath 2.0 (не в браузерах) |

**Вывод:** CSS — основной инструмент. XPath — когда нужна навигация вверх по DOM или поиск по тексту.

---

## Anti-patterns (чего избегать)

### 1. Хрупкие локаторы

```java
// ПЛОХО: абсолютный XPath — сломается при любом изменении структуры
driver.findElement(By.xpath("/html/body/div[1]/div[2]/form/div[3]/input"));

// ХОРОШО: поиск по атрибутам
driver.findElement(By.cssSelector("input[name='username']"));
```

### 2. Индексные локаторы

```java
// ПЛОХО: зависимость от порядка элементов
driver.findElement(By.xpath("//div[3]/button[2]"));

// ХОРОШО: поиск по содержимому или атрибуту
driver.findElement(By.xpath("//button[text()='Save']"));
```

### 3. Автогенерируемые имена классов

```java
// ПЛОХО: классы, сгенерированные CSS-modules или styled-components
driver.findElement(By.cssSelector(".sc-bdnxRM.jPwKLQ"));
driver.findElement(By.cssSelector(".styles_button__3xKD2"));

// ХОРОШО: использовать data-testid или более стабильные атрибуты
driver.findElement(By.cssSelector("[data-testid='save-button']"));
```

### 4. Зависимость от текста (проблема i18n)

```java
// ПЛОХО: текст может измениться при локализации
driver.findElement(By.xpath("//button[text()='Отправить']"));

// ХОРОШО: не зависит от языка
driver.findElement(By.cssSelector("[data-testid='submit-btn']"));
```

### 5. Слишком общие локаторы

```java
// ПЛОХО: вернёт первый попавшийся div
driver.findElement(By.tagName("div"));

// ПЛОХО: слишком много элементов с таким классом
driver.findElement(By.className("btn"));

// ХОРОШО: более точный локатор
driver.findElement(By.cssSelector("form#checkout .btn-submit"));
```

---

## Relative Locators (Selenium 4)

Selenium 4 ввёл концепцию "дружественных" (relative/friendly) локаторов. Они позволяют искать элемент
относительно другого, уже известного элемента — по визуальному расположению на странице.

### Доступные методы

```java
import static org.openqa.selenium.support.locators.RelativeLocator.with;

// Элемент выше указанного
WebElement element = driver.findElement(
    with(By.tagName("input")).above(By.id("password"))
);

// Элемент ниже указанного
WebElement element = driver.findElement(
    with(By.tagName("input")).below(By.id("username"))
);

// Элемент левее указанного
WebElement label = driver.findElement(
    with(By.tagName("label")).toLeftOf(By.id("email"))
);

// Элемент правее указанного
WebElement icon = driver.findElement(
    with(By.tagName("span")).toRightOf(By.id("username"))
);

// Рядом с указанным элементом (в радиусе ~50px по умолчанию)
WebElement helpText = driver.findElement(
    with(By.cssSelector(".help-text")).near(By.id("password"))
);
```

### Цепочка относительных условий

```java
// Элемент ниже username и выше password
WebElement emailField = driver.findElement(
    with(By.tagName("input"))
        .below(By.id("username"))
        .above(By.id("password"))
);
```

### Ограничения

- Работают на основе координат элементов, что зависит от viewport и разрешения экрана
- Не подходят для headless-режима с нестандартными размерами окна
- Менее стабильны, чем прямые CSS/XPath-локаторы
- Лучше использовать как вспомогательный инструмент, а не как основную стратегию

---

## Построение устойчивых локаторов: алгоритм

1. **Проверить наличие уникального `id`** — если есть и он стабильный (не автогенерируемый), использовать его
2. **Проверить наличие `data-testid`** — если есть, использовать
3. **Использовать стабильные атрибуты** — `name`, `type`, `role`, `aria-label`, `href`
4. **Построить короткий CSS Selector** — с минимумом уровней вложенности
5. **Использовать XPath только при необходимости** — навигация вверх по DOM, поиск по тексту
6. **Валидировать в DevTools** — `$$('css-selector')` или `$x('xpath')` в консоли браузера
7. **Проверить уникальность** — убедиться, что локатор возвращает ровно один элемент

```java
/**
 * Вспомогательный метод для проверки уникальности локатора.
 * Полезен при отладке тестов.
 */
public void assertLocatorIsUnique(By locator) {
    List<WebElement> elements = driver.findElements(locator);
    if (elements.size() != 1) {
        throw new AssertionError(
            String.format("Локатор '%s' нашёл %d элементов, ожидался 1",
                locator, elements.size())
        );
    }
}
```

---

## Связь с тестированием

- **Стабильность тестов** — правильные локаторы = меньше flaky-тестов
- **Page Object Model** — локаторы инкапсулируются в Page Object-классах, облегчая поддержку
- **Code Review** — при ревью автотестов оценка качества локаторов — обязательный пункт
- **Взаимодействие с разработчиками** — QA должен уметь обосновать запрос на добавление `data-testid`
- **Производительность** — неоптимальные XPath-выражения замедляют выполнение тестов
- **Кросс-браузерность** — некоторые pseudo-классы CSS могут работать по-разному в разных браузерах

---

## Типичные ошибки

1. **Копирование XPath из DevTools** — браузер генерирует абсолютный путь, который крайне хрупок
2. **Использование `Thread.sleep()` вместо правильного локатора** — элемент не находится из-за плохого локатора, а не потому, что не загрузился
3. **Игнорирование динамических атрибутов** — ID вида `ember123`, `react-select-2-input` меняются при каждой загрузке
4. **Слишком длинные цепочки** — `div > div > div > ul > li > a` — хрупко и нечитаемо
5. **Неучёт iframe** — элемент внутри iframe не находится обычным локатором без переключения контекста
6. **Неучёт Shadow DOM** — элементы внутри Shadow DOM недоступны стандартными локаторами

---

## Вопросы на интервью

### Уровень Junior

- Какие стратегии поиска элементов вы знаете? Перечислите методы класса `By`.
- В чём разница между `findElement()` и `findElements()`?
- Что такое CSS Selector? Приведите примеры.
- Как найти элемент по атрибуту `data-testid` с помощью CSS?
- Какой локатор самый надёжный и почему?

### Уровень Middle

- Сравните CSS Selectors и XPath. Когда вы предпочтёте один другому?
- Что такое оси (axes) в XPath? Приведите примеры использования `ancestor`, `following-sibling`.
- Как вы строите устойчивые локаторы? Опишите ваш подход.
- Что такое anti-pattern в контексте локаторов? Приведите 3 примера.
- Как работают Relative Locators в Selenium 4? Каковы их ограничения?
- Как вы валидируете локатор перед использованием в тесте?

### Уровень Senior

- Как организовать стратегию локаторов в крупном проекте на 500+ тестов?
- Как убедить команду разработки добавлять `data-testid` в компоненты?
- Как работать с элементами внутри Shadow DOM?
- Как оптимизировать производительность тестов с точки зрения локаторов?
- Что бы вы включили в линтер/код-ревью для проверки качества локаторов?
- Как обрабатывать динамические элементы (списки, таблицы с пагинацией) без индексных локаторов?

---

## Практические задания

### Задание 1: Написание CSS Selectors

Дан HTML-фрагмент:
```html
<div class="product-list">
  <div class="product-card" data-testid="product-1">
    <h3 class="product-title">Laptop</h3>
    <span class="price">$999</span>
    <button class="btn btn-primary add-to-cart">Add to Cart</button>
  </div>
  <div class="product-card" data-testid="product-2">
    <h3 class="product-title">Phone</h3>
    <span class="price sale">$499</span>
    <button class="btn btn-primary add-to-cart" disabled>Out of Stock</button>
  </div>
</div>
```

Напишите CSS Selectors для:
1. Кнопки "Add to Cart" для первого товара
2. Цены со скидкой (класс `sale`)
3. Всех активных (не disabled) кнопок
4. Заголовка товара с `data-testid="product-2"`

### Задание 2: Написание XPath

Используя тот же HTML, напишите XPath для:
1. Карточки товара, содержащей текст "Laptop"
2. Кнопки, следующей за ценой со скидкой
3. Родительского div для заголовка "Phone"
4. Всех заголовков товаров, у которых кнопка не disabled

### Задание 3: Рефакторинг хрупких локаторов

Перепишите данные локаторы, сделав их устойчивыми:
```java
// 1. Абсолютный XPath
By.xpath("/html/body/div[1]/div[2]/div/form/div[3]/div/input");

// 2. Автогенерируемый класс
By.cssSelector(".sc-1a2b3c.kLmNoP");

// 3. Индексный локатор
By.xpath("//table/tbody/tr[5]/td[3]/a");

// 4. Зависимость от текста
By.xpath("//button[text()='Сохранить изменения']");
```

### Задание 4: Page Object с правильными локаторами

Создайте Page Object для страницы логина, содержащей:
- Поле email
- Поле пароля
- Кнопку "Войти"
- Ссылку "Забыли пароль?"
- Сообщение об ошибке

Используйте правильные стратегии локаторов и обоснуйте выбор.

---

## Дополнительные ресурсы

- [Selenium Documentation — Locator Strategies](https://www.selenium.dev/documentation/webdriver/elements/locators/)
- [MDN — CSS Selectors](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_selectors)
- [W3C — XPath Specification](https://www.w3.org/TR/xpath/)
- [Selenium 4 — Relative Locators](https://www.selenium.dev/documentation/webdriver/elements/locators/#relative-locators)
- [Testing Library — Guiding Principles](https://testing-library.com/docs/guiding-principles) — философия приоритета локаторов
- [CSS Diner](https://flukeout.github.io/) — интерактивная игра для изучения CSS Selectors
- [XPath Cheatsheet (devhints.io)](https://devhints.io/xpath)
