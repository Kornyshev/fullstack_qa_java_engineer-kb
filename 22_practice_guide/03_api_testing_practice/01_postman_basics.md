# Первые шаги в Postman

## Содержание

1. [Установка и настройка](#установка-и-настройка)
2. [Интерфейс Postman](#интерфейс-postman)
3. [Первый GET-запрос](#первый-get-запрос)
4. [Коллекции](#коллекции)
5. [Переменные](#переменные)
6. [Написание тестов](#написание-тестов)
7. [Runner и Data-Driven тестирование](#runner-и-data-driven-тестирование)
8. [Экспорт коллекции для портфолио](#экспорт-коллекции-для-портфолио)

---

## Установка и настройка

### Шаг 1: Скачивание Postman

Перейдите на официальный сайт: [https://www.postman.com/downloads/](https://www.postman.com/downloads/)

Скачайте версию для вашей ОС (Windows / macOS / Linux). Установите приложение стандартным способом.

### Шаг 2: Регистрация аккаунта

Зарегистрируйтесь — это бесплатно и даёт доступ к синхронизации коллекций между устройствами. Можно войти через Google или создать аккаунт через email.

### Шаг 3: Создание Workspace

1. Откройте Postman после установки
2. Нажмите **Workspaces** в верхнем меню
3. Выберите **Create Workspace**
4. Укажите имя: `QA Practice`
5. Выберите тип: **Personal**
6. Нажмите **Create Workspace**

> **Ожидаемый результат:** У вас появится пустой workspace с именем "QA Practice", в который мы будем добавлять коллекции.

---

## Интерфейс Postman

### Основные элементы

| Элемент | Расположение | Назначение |
|---------|-------------|------------|
| Sidebar | Слева | Коллекции, Environments, History |
| Request Tab | Центр | URL, метод, параметры, body |
| Response Panel | Внизу | Тело ответа, статус, время, размер |
| Console | Нижняя панель | Логи запросов, вывод `console.log()` |
| Runner | Меню | Запуск коллекции целиком |

### Практическое задание: Знакомство с интерфейсом

1. Откройте **Postman Console**: View → Show Postman Console (или `Ctrl+Alt+C`)
2. Создайте новую вкладку запроса (`Ctrl+T`)
3. Найдите выпадающий список HTTP-методов (по умолчанию `GET`)
4. Обратите внимание на вкладки: **Params**, **Authorization**, **Headers**, **Body**, **Pre-request Script**, **Tests**

---

## Первый GET-запрос

### Упражнение 1: Получение списка постов

1. В строке URL введите: `https://jsonplaceholder.typicode.com/posts`
2. Убедитесь, что выбран метод **GET**
3. Нажмите **Send**

**Ожидаемый результат:**
- Status: `200 OK`
- Время ответа: менее 1 секунды
- Body: JSON-массив из 100 объектов
- Каждый объект содержит поля: `userId`, `id`, `title`, `body`

### Упражнение 2: Получение конкретного поста

1. URL: `https://jsonplaceholder.typicode.com/posts/1`
2. Метод: **GET**
3. Нажмите **Send**

**Ожидаемый результат:**
```json
{
  "userId": 1,
  "id": 1,
  "title": "sunt aut facere repellat provident occaecati excepturi optio reprehenderit",
  "body": "quia et suscipit..."
}
```

### Упражнение 3: Передача Query Parameters

1. URL: `https://jsonplaceholder.typicode.com/comments`
2. Перейдите на вкладку **Params**
3. Добавьте параметр: Key = `postId`, Value = `1`
4. Обратите внимание — URL автоматически обновился: `...comments?postId=1`
5. Нажмите **Send**

**Ожидаемый результат:** Массив из 5 комментариев, каждый с `postId: 1`.

### Упражнение 4: POST-запрос

1. Создайте новую вкладку (`Ctrl+T`)
2. Выберите метод **POST**
3. URL: `https://jsonplaceholder.typicode.com/posts`
4. Перейдите на вкладку **Body**
5. Выберите **raw** и формат **JSON**
6. Введите тело запроса:

```json
{
  "title": "Мой тестовый пост",
  "body": "Содержимое поста для практики",
  "userId": 1
}
```

7. Нажмите **Send**

**Ожидаемый результат:**
- Status: `201 Created`
- В теле ответа присутствует `"id": 101`

---

## Коллекции

### Создание коллекции

1. В Sidebar нажмите **Collections** → **New Collection**
2. Назовите: `JSONPlaceholder API Tests`
3. Добавьте описание: "Коллекция запросов для тренировки API-тестирования"

### Организация запросов в папки

Создайте следующую структуру:

```
JSONPlaceholder API Tests/
├── Posts/
│   ├── GET All Posts
│   ├── GET Single Post
│   ├── POST Create Post
│   ├── PUT Update Post
│   └── DELETE Post
├── Comments/
│   ├── GET Comments by Post
│   └── GET Single Comment
└── Users/
    ├── GET All Users
    └── GET Single User
```

### Практическое задание: Заполнение коллекции

Для каждого запроса в структуре выше:

1. Нажмите правой кнопкой на папку → **Add Request**
2. Заполните URL и метод
3. Для PUT используйте URL `https://jsonplaceholder.typicode.com/posts/1` и body:

```json
{
  "id": 1,
  "title": "Обновлённый заголовок",
  "body": "Обновлённое содержимое",
  "userId": 1
}
```

4. Для DELETE используйте `https://jsonplaceholder.typicode.com/posts/1`

**Ожидаемый результат:** Коллекция с 9 запросами, организованными в 3 папки.

---

## Переменные

### Типы переменных в Postman

| Тип | Область видимости | Приоритет | Применение |
|-----|------------------|-----------|------------|
| Global | Весь Postman | Низкий | Общие настройки |
| Collection | Одна коллекция | Средний | Переменные коллекции |
| Environment | Выбранный environment | Высокий | URL, ключи для разных сред |
| Local | Один запрос | Наивысший | Временные данные |

### Упражнение 5: Создание Environment

1. Нажмите **Environments** в Sidebar → **Create Environment**
2. Назовите: `JSONPlaceholder`
3. Добавьте переменные:

| Variable | Initial Value | Current Value |
|----------|--------------|---------------|
| `base_url` | `https://jsonplaceholder.typicode.com` | `https://jsonplaceholder.typicode.com` |
| `post_id` | `1` | `1` |
| `user_id` | `1` | `1` |

4. Сохраните Environment
5. Выберите его в выпадающем списке справа вверху

### Упражнение 6: Использование переменных в запросах

1. Откройте запрос "GET Single Post"
2. Замените URL на: `{{base_url}}/posts/{{post_id}}`
3. Нажмите **Send**

**Ожидаемый результат:** Запрос отправляется на тот же адрес, но теперь URL параметризован. При наведении на `{{base_url}}` показывается resolved-значение.

### Упражнение 7: Collection Variables

1. Откройте настройки коллекции (три точки → **Edit**)
2. Перейдите на вкладку **Variables**
3. Добавьте: `content_type` = `application/json`
4. В запросе POST добавьте заголовок: `Content-Type: {{content_type}}`

### Упражнение 8: Динамические переменные в Pre-request Script

Откройте запрос "POST Create Post", вкладка **Pre-request Script**:

```javascript
// Генерируем случайный заголовок для поста
pm.variables.set("random_title", "Test Post " + Math.floor(Math.random() * 1000));
pm.variables.set("random_body", "Автоматически сгенерированный текст поста");
```

Обновите body запроса:

```json
{
  "title": "{{random_title}}",
  "body": "{{random_body}}",
  "userId": {{user_id}}
}
```

**Ожидаемый результат:** Каждый раз при отправке запроса `title` содержит случайное число.

---

## Написание тестов

### Основы тестов в Postman

Тесты пишутся на JavaScript во вкладке **Tests** запроса. Они выполняются после получения ответа.

### Упражнение 9: Проверка статус-кода

Откройте "GET All Posts", вкладка **Tests**:

```javascript
// Проверяем, что сервер вернул статус 200
pm.test("Статус-код равен 200", function () {
    pm.response.to.have.status(200);
});

// Альтернативный синтаксис
pm.test("Успешный ответ", function () {
    pm.expect(pm.response.code).to.equal(200);
});
```

### Упражнение 10: Проверка тела ответа

```javascript
// Парсим JSON из ответа
const responseData = pm.response.json();

// Проверяем, что ответ — массив
pm.test("Ответ является массивом", function () {
    pm.expect(responseData).to.be.an("array");
});

// Проверяем количество элементов
pm.test("Массив содержит 100 постов", function () {
    pm.expect(responseData.length).to.equal(100);
});

// Проверяем структуру первого элемента
pm.test("Пост содержит обязательные поля", function () {
    const firstPost = responseData[0];
    pm.expect(firstPost).to.have.property("userId");
    pm.expect(firstPost).to.have.property("id");
    pm.expect(firstPost).to.have.property("title");
    pm.expect(firstPost).to.have.property("body");
});
```

### Упражнение 11: Проверка заголовков

```javascript
// Проверяем Content-Type
pm.test("Content-Type содержит application/json", function () {
    pm.expect(pm.response.headers.get("Content-Type")).to.include("application/json");
});

// Проверяем время ответа
pm.test("Время ответа менее 500мс", function () {
    pm.expect(pm.response.responseTime).to.be.below(500);
});
```

### Упражнение 12: Тесты для POST-запроса

Откройте "POST Create Post", вкладка **Tests**:

```javascript
// Проверяем статус 201 Created
pm.test("Пост успешно создан (201)", function () {
    pm.response.to.have.status(201);
});

const responseData = pm.response.json();

// Проверяем, что вернулся id
pm.test("Ответ содержит id нового поста", function () {
    pm.expect(responseData).to.have.property("id");
    pm.expect(responseData.id).to.be.a("number");
});

// Сохраняем id для последующих запросов
pm.test("Сохраняем id созданного поста", function () {
    pm.environment.set("created_post_id", responseData.id);
    console.log("Создан пост с id: " + responseData.id);
});
```

### Упражнение 13: Цепочка запросов

Используйте сохранённый `created_post_id` в следующем запросе:

1. Создайте запрос "GET Created Post"
2. URL: `{{base_url}}/posts/{{created_post_id}}`
3. Добавьте тест:

```javascript
pm.test("Получен ранее созданный пост", function () {
    const data = pm.response.json();
    pm.expect(data.id).to.equal(parseInt(pm.environment.get("created_post_id")));
});
```

---

## Runner и Data-Driven тестирование

### Упражнение 14: Запуск коллекции через Runner

1. Нажмите **Run** (иконка ▶ рядом с коллекцией)
2. В Runner выберите все запросы папки **Posts**
3. Установите **Iterations**: `1`
4. Нажмите **Run JSONPlaceholder API Tests**

**Ожидаемый результат:** Все запросы выполнятся последовательно, для каждого запроса отобразятся результаты тестов (PASS/FAIL).

### Упражнение 15: Data-Driven тестирование с CSV

Создайте файл `test_posts.csv` со следующим содержимым:

```csv
post_id,expected_user_id,expected_title_contains
1,1,sunt aut facere
2,1,qui est esse
11,2,et ea vero
51,6,soluta modi
100,10,at nam consequatur
```

#### Настройка запроса

1. Откройте запрос "GET Single Post"
2. URL: `{{base_url}}/posts/{{post_id}}`
3. Вкладка **Tests**:

```javascript
// Получаем ожидаемые значения из CSV-данных
const expectedUserId = parseInt(pm.iterationData.get("expected_user_id"));
const expectedTitleContains = pm.iterationData.get("expected_title_contains");
const postId = parseInt(pm.iterationData.get("post_id"));

const responseData = pm.response.json();

pm.test("Статус 200 для поста " + postId, function () {
    pm.response.to.have.status(200);
});

pm.test("userId соответствует ожидаемому для поста " + postId, function () {
    pm.expect(responseData.userId).to.equal(expectedUserId);
});

pm.test("Заголовок содержит ожидаемый текст для поста " + postId, function () {
    pm.expect(responseData.title).to.include(expectedTitleContains);
});
```

#### Запуск с данными

1. Откройте Runner
2. Выберите только запрос "GET Single Post"
3. В поле **Data** загрузите файл `test_posts.csv`
4. Нажмите **Preview** — убедитесь, что данные распознаны корректно
5. Iterations автоматически установится = 5 (по числу строк)
6. Нажмите **Run**

**Ожидаемый результат:** 5 итераций, для каждой — 3 теста. Все 15 тестов должны пройти (PASS).

---

## Экспорт коллекции для портфолио

### Упражнение 16: Экспорт коллекции

1. Нажмите три точки рядом с коллекцией → **Export**
2. Выберите формат: **Collection v2.1**
3. Сохраните файл: `JSONPlaceholder_API_Tests.postman_collection.json`

### Упражнение 17: Экспорт Environment

1. Environments → три точки рядом с "JSONPlaceholder" → **Export**
2. Сохраните: `JSONPlaceholder.postman_environment.json`

### Как добавить в портфолио (GitHub)

Создайте репозиторий на GitHub со следующей структурой:

```
postman-api-tests/
├── collections/
│   └── JSONPlaceholder_API_Tests.postman_collection.json
├── environments/
│   └── JSONPlaceholder.postman_environment.json
├── data/
│   └── test_posts.csv
└── README.md
```

В `README.md` укажите:
- Описание проекта
- Как импортировать коллекцию
- Как запустить тесты
- Скриншоты результатов Runner

### Упражнение 18: Запуск через Newman (CLI)

Newman — инструмент для запуска Postman-коллекций из командной строки.

```bash
# Установка Newman
npm install -g newman

# Запуск коллекции
newman run collections/JSONPlaceholder_API_Tests.postman_collection.json \
  -e environments/JSONPlaceholder.postman_environment.json \
  -d data/test_posts.csv

# Запуск с HTML-отчётом
npm install -g newman-reporter-htmlextra

newman run collections/JSONPlaceholder_API_Tests.postman_collection.json \
  -e environments/JSONPlaceholder.postman_environment.json \
  -r htmlextra \
  --reporter-htmlextra-export results/report.html
```

**Ожидаемый результат:** В терминале — статистика прохождения тестов. При использовании `htmlextra` — HTML-отчёт в папке `results/`.

---

## Чек-лист освоенных навыков

| Навык | Статус |
|-------|--------|
| Установка Postman и создание Workspace | ☐ |
| Отправка GET, POST, PUT, DELETE запросов | ☐ |
| Создание и организация коллекций | ☐ |
| Использование Environment Variables | ☐ |
| Использование Collection Variables | ☐ |
| Написание тестов на проверку статус-кода | ☐ |
| Написание тестов на проверку тела ответа | ☐ |
| Написание тестов на проверку заголовков | ☐ |
| Цепочка запросов через переменные | ☐ |
| Data-Driven тестирование с CSV | ☐ |
| Экспорт коллекции и Environment | ☐ |
| Запуск через Newman | ☐ |

---

## Дополнительные задания для самостоятельной работы

1. **Тесты для Users**: Напишите тесты для эндпоинта `/users` — проверьте, что каждый пользователь имеет email в корректном формате
2. **PATCH-запрос**: Создайте запрос PATCH к `/posts/1`, обновив только `title`, и напишите тест, что `body` осталось без изменений
3. **Negative-тесты**: Создайте запрос к несуществующему посту `/posts/999` и убедитесь, что возвращается пустой объект `{}`
4. **Pre-request Script**: Напишите Pre-request Script, который генерирует случайный `userId` от 1 до 10 для создания поста
5. **Collection-level Tests**: Добавьте тест на уровне коллекции, который для всех запросов проверяет время ответа < 2000мс
