# Открытые API для практики

## Содержание

1. [Обзор ресурсов](#обзор-ресурсов)
2. [JSONPlaceholder — CRUD-операции](#jsonplaceholder--crud-операции)
3. [Reqres — Авторизация и пагинация](#reqres--авторизация-и-пагинация)
4. [PetStore — OpenAPI-спецификация](#petstore--openapi-спецификация)
5. [RestCountries — Фильтрация и поиск](#restcountries--фильтрация-и-поиск)
6. [CoinGecko — Rate Limiting](#coingecko--rate-limiting)
7. [OpenWeatherMap — Работа с API Key](#openweathermap--работа-с-api-key)
8. [GitHub API — OAuth и пагинация](#github-api--oauth-и-пагинация)
9. [Сводная таблица](#сводная-таблица)

---

## Обзор ресурсов

Ниже представлены 7 открытых API, каждый из которых позволяет отработать различные навыки API-тестирования. Все API бесплатны (или имеют бесплатный тарифный план).

| API | URL | Требует ключ | Основной навык |
|-----|-----|:------------:|----------------|
| JSONPlaceholder | `https://jsonplaceholder.typicode.com` | Нет | CRUD, REST-методы |
| Reqres | `https://reqres.in` | Нет | Авторизация, регистрация |
| PetStore | `https://petstore.swagger.io` | Нет | OpenAPI, Swagger UI |
| RestCountries | `https://restcountries.com` | Нет | Фильтрация, query params |
| CoinGecko | `https://api.coingecko.com` | Нет (free tier) | Rate limiting, пагинация |
| OpenWeatherMap | `https://api.openweathermap.org` | Да (бесплатный) | API Key, authentication |
| GitHub API | `https://api.github.com` | Опционально | OAuth, pagination, headers |

---

## JSONPlaceholder — CRUD-операции

**URL:** [https://jsonplaceholder.typicode.com](https://jsonplaceholder.typicode.com)

**Описание:** Фейковый REST API для прототипирования и тестирования. Поддерживает все CRUD-операции, но данные не сохраняются на сервере (имитация).

### Доступные ресурсы

| Ресурс | Эндпоинт | Кол-во записей |
|--------|----------|:--------------:|
| Posts | `/posts` | 100 |
| Comments | `/comments` | 500 |
| Albums | `/albums` | 100 |
| Photos | `/photos` | 5000 |
| Todos | `/todos` | 200 |
| Users | `/users` | 10 |

### Примеры запросов

```http
# Получить все посты
GET https://jsonplaceholder.typicode.com/posts

# Получить пост по id
GET https://jsonplaceholder.typicode.com/posts/1

# Получить комментарии к посту (вложенный ресурс)
GET https://jsonplaceholder.typicode.com/posts/1/comments

# Фильтрация по query-параметру
GET https://jsonplaceholder.typicode.com/comments?postId=1

# Создать пост
POST https://jsonplaceholder.typicode.com/posts
Content-Type: application/json

{
  "title": "Тестовый пост",
  "body": "Содержимое",
  "userId": 1
}

# Обновить пост (полное обновление)
PUT https://jsonplaceholder.typicode.com/posts/1
Content-Type: application/json

{
  "id": 1,
  "title": "Обновлённый",
  "body": "Новое содержимое",
  "userId": 1
}

# Частичное обновление
PATCH https://jsonplaceholder.typicode.com/posts/1
Content-Type: application/json

{
  "title": "Только заголовок обновлён"
}

# Удалить пост
DELETE https://jsonplaceholder.typicode.com/posts/1
```

### Задания

1. **CRUD-цикл**: Выполните полный цикл — создайте пост (POST), получите его (GET), обновите (PUT), частично измените (PATCH), удалите (DELETE). Проверьте статус-коды: 201, 200, 200, 200, 200.
2. **Фильтрация**: Получите все `todos` пользователя с `userId=3`. Проверьте, что все элементы действительно принадлежат этому пользователю.
3. **Вложенные ресурсы**: Сравните результаты `/posts/1/comments` и `/comments?postId=1` — они должны быть идентичны.
4. **Negative-тест**: Отправьте GET на `/posts/999`. Ожидаемый результат — пустой объект `{}` со статусом `200` (это особенность JSONPlaceholder).
5. **Валидация структуры**: Для каждого ресурса проверьте, что объекты содержат все обязательные поля.

---

## Reqres — Авторизация и пагинация

**URL:** [https://reqres.in](https://reqres.in)

**Описание:** API, специально созданный для тестирования. Имитирует аутентификацию (register, login) и возвращает реалистичные данные с пагинацией.

### Примеры запросов

```http
# Список пользователей (страница 2)
GET https://reqres.in/api/users?page=2

# Один пользователь
GET https://reqres.in/api/users/2

# Несуществующий пользователь
GET https://reqres.in/api/users/23

# Успешная регистрация
POST https://reqres.in/api/register
Content-Type: application/json

{
  "email": "eve.holt@reqres.in",
  "password": "pistol"
}

# Неуспешная регистрация (без пароля)
POST https://reqres.in/api/register
Content-Type: application/json

{
  "email": "sydney@fife"
}

# Успешный логин
POST https://reqres.in/api/login
Content-Type: application/json

{
  "email": "eve.holt@reqres.in",
  "password": "cityslicka"
}

# Неуспешный логин
POST https://reqres.in/api/login
Content-Type: application/json

{
  "email": "peter@klaven"
}

# Отложенный ответ (имитация медленного сервера)
GET https://reqres.in/api/users?delay=3
```

### Задания

1. **Пагинация**: Отправьте запросы на `/api/users?page=1` и `/api/users?page=2`. Убедитесь, что `total_pages`, `total`, `per_page` одинаковы, а `data` — разные.
2. **Регистрация + Логин**: Зарегистрируйте пользователя, извлеките `token`, используйте его для логина. Проверьте, что токен — непустая строка.
3. **Негативные сценарии**: Отправьте запрос на регистрацию без `password`. Ожидайте статус `400` и сообщение `"Missing password"`.
4. **404**: Запросите пользователя `/api/users/23`. Ожидайте статус `404` и пустой объект `{}`.
5. **Delayed Response**: Отправьте запрос с `?delay=5` и проверьте, что ответ приходит не ранее чем через 5 секунд.

---

## PetStore — OpenAPI-спецификация

**URL:** [https://petstore.swagger.io](https://petstore.swagger.io)

**API endpoint:** `https://petstore.swagger.io/v2`

**Описание:** Классический пример Swagger-документации. API для управления зоомагазином: питомцы, заказы, пользователи.

### Примеры запросов

```http
# Получить питомца по ID
GET https://petstore.swagger.io/v2/pet/1

# Найти питомцев по статусу
GET https://petstore.swagger.io/v2/pet/findByStatus?status=available

# Добавить нового питомца
POST https://petstore.swagger.io/v2/pet
Content-Type: application/json

{
  "id": 12345,
  "name": "Барсик",
  "status": "available",
  "category": {
    "id": 1,
    "name": "Cats"
  },
  "photoUrls": ["https://example.com/cat.jpg"],
  "tags": [
    {"id": 1, "name": "cute"}
  ]
}

# Обновить статус питомца
PUT https://petstore.swagger.io/v2/pet
Content-Type: application/json

{
  "id": 12345,
  "name": "Барсик",
  "status": "sold"
}

# Создать заказ
POST https://petstore.swagger.io/v2/store/order
Content-Type: application/json

{
  "id": 10,
  "petId": 12345,
  "quantity": 1,
  "shipDate": "2026-03-25T00:00:00.000Z",
  "status": "placed",
  "complete": true
}

# Инвентарь магазина
GET https://petstore.swagger.io/v2/store/inventory
```

### Задания

1. **Swagger UI**: Откройте [https://petstore.swagger.io](https://petstore.swagger.io), изучите документацию. Найдите все эндпоинты и их параметры.
2. **Полный цикл**: Создайте питомца → найдите его по ID → обновите статус → удалите → убедитесь, что GET по ID возвращает 404.
3. **Поиск по статусу**: Получите питомцев со статусом `sold`. Проверьте, что у всех `status` = `"sold"`.
4. **Заказ**: Создайте заказ на питомца, получите его по ID, проверьте поля.
5. **Невалидные данные**: Попробуйте создать питомца без обязательных полей. Изучите сообщения об ошибках.

---

## RestCountries — Фильтрация и поиск

**URL:** [https://restcountries.com](https://restcountries.com)

**Описание:** Информация о странах мира. Поддерживает различные виды фильтрации и выбор полей.

### Примеры запросов

```http
# Все страны
GET https://restcountries.com/v3.1/all

# Поиск по имени
GET https://restcountries.com/v3.1/name/russia

# Поиск по полному имени
GET https://restcountries.com/v3.1/name/russian federation?fullText=true

# По коду страны
GET https://restcountries.com/v3.1/alpha/ru

# По региону
GET https://restcountries.com/v3.1/region/europe

# По валюте
GET https://restcountries.com/v3.1/currency/rub

# По языку
GET https://restcountries.com/v3.1/lang/russian

# Выбор конкретных полей
GET https://restcountries.com/v3.1/name/germany?fields=name,capital,population,flags
```

### Задания

1. **Фильтрация по региону**: Получите все страны Азии (`/region/asia`). Проверьте, что `region` = `"Asia"` для каждой страны.
2. **Выбор полей**: Запросите только `name` и `capital` для России. Убедитесь, что в ответе нет лишних полей.
3. **Поиск**: Найдите страны по частичному имени `"united"`. Проверьте, что в результатах есть `United States`, `United Kingdom`, `United Arab Emirates`.
4. **Несуществующая страна**: Запросите `/name/abcxyz`. Ожидайте статус `404` и сообщение `"Not Found"`.
5. **Сравнение**: Получите данные о России через `/name/russia` и `/alpha/ru`. Сравните результаты.

---

## CoinGecko — Rate Limiting

**URL:** [https://api.coingecko.com/api/v3](https://api.coingecko.com/api/v3)

**Описание:** Данные о криптовалютах в реальном времени. Отличный ресурс для практики работы с rate limiting и большими объёмами данных.

### Примеры запросов

```http
# Проверка статуса API
GET https://api.coingecko.com/api/v3/ping

# Список монет (с пагинацией)
GET https://api.coingecko.com/api/v3/coins/markets?vs_currency=usd&per_page=10&page=1

# Информация о конкретной монете
GET https://api.coingecko.com/api/v3/coins/bitcoin

# Текущая цена
GET https://api.coingecko.com/api/v3/simple/price?ids=bitcoin,ethereum&vs_currencies=usd,eur

# Список поддерживаемых валют
GET https://api.coingecko.com/api/v3/simple/supported_vs_currencies

# Глобальная статистика рынка
GET https://api.coingecko.com/api/v3/global
```

### Задания

1. **Ping**: Отправьте запрос на `/ping`. Ожидаемый ответ: `{"gecko_says": "(V3) To the Moon!"}`.
2. **Пагинация**: Получите первую и вторую страницу списка монет (`per_page=10`). Убедитесь, что монеты не повторяются.
3. **Мультивалюта**: Запросите цену Bitcoin в USD, EUR и RUB одним запросом. Проверьте наличие всех трёх валют в ответе.
4. **Rate Limiting**: Отправьте 50+ запросов подряд. Зафиксируйте, при каком количестве начнёте получать статус `429 Too Many Requests`. Проверьте заголовок `Retry-After`.
5. **Сортировка**: Получите ТОП-10 монет по market cap. Убедитесь, что первый элемент — Bitcoin.

---

## OpenWeatherMap — Работа с API Key

**URL:** [https://openweathermap.org/api](https://openweathermap.org/api)

**API endpoint:** `https://api.openweathermap.org/data/2.5`

**Описание:** Погодные данные. Требует бесплатную регистрацию и API Key. Отличная практика работы с аутентификацией через ключ.

### Получение API Key

1. Зарегистрируйтесь на [https://home.openweathermap.org/users/sign_up](https://home.openweathermap.org/users/sign_up)
2. Перейдите в раздел **API Keys**: [https://home.openweathermap.org/api_keys](https://home.openweathermap.org/api_keys)
3. Скопируйте ключ (активация занимает до 2 часов)

### Примеры запросов

```http
# Текущая погода по городу
GET https://api.openweathermap.org/data/2.5/weather?q=Moscow&appid=YOUR_API_KEY&units=metric&lang=ru

# По координатам
GET https://api.openweathermap.org/data/2.5/weather?lat=55.75&lon=37.62&appid=YOUR_API_KEY&units=metric

# Прогноз на 5 дней
GET https://api.openweathermap.org/data/2.5/forecast?q=Moscow&appid=YOUR_API_KEY&units=metric

# Невалидный ключ
GET https://api.openweathermap.org/data/2.5/weather?q=Moscow&appid=INVALID_KEY

# Несуществующий город
GET https://api.openweathermap.org/data/2.5/weather?q=NonExistentCity123&appid=YOUR_API_KEY
```

### Задания

1. **Регистрация и ключ**: Получите API Key, сохраните его в Environment Variable в Postman как `weather_api_key`. Никогда не коммитьте ключ в Git.
2. **Текущая погода**: Запросите погоду в Москве. Проверьте, что ответ содержит `main.temp`, `wind.speed`, `weather[0].description`.
3. **Единицы измерения**: Сделайте запрос с `units=metric` и `units=imperial`. Проверьте, что температура различается (Цельсий vs Фаренгейт).
4. **Невалидный ключ**: Отправьте запрос с `appid=invalidkey`. Ожидайте статус `401` и сообщение `"Invalid API key"`.
5. **Несуществующий город**: Запросите погоду для `q=xyzabc123`. Ожидайте статус `404` и `"city not found"`.
6. **Прогноз**: Получите 5-дневный прогноз. Проверьте, что `list` содержит 40 записей (8 отметок в сутки × 5 дней).

---

## GitHub API — OAuth и пагинация

**URL:** [https://api.github.com](https://api.github.com)

**Документация:** [https://docs.github.com/en/rest](https://docs.github.com/en/rest)

**Описание:** Один из самых качественных публичных API. Поддерживает OAuth, имеет подробную документацию, rate limiting, пагинацию и множество ресурсов.

### Получение Personal Access Token

1. Перейдите на [https://github.com/settings/tokens](https://github.com/settings/tokens)
2. Нажмите **Generate new token (classic)**
3. Выберите scopes: `repo`, `user`
4. Скопируйте токен (он показывается только один раз!)

### Примеры запросов

```http
# Публичная информация о пользователе (без авторизации)
GET https://api.github.com/users/octocat

# Репозитории пользователя
GET https://api.github.com/users/octocat/repos

# Поиск репозиториев
GET https://api.github.com/search/repositories?q=language:java+stars:>1000&sort=stars&per_page=10

# Авторизованный запрос — информация о себе
GET https://api.github.com/user
Authorization: Bearer YOUR_TOKEN

# Репозиторий с деталями
GET https://api.github.com/repos/SeleniumHQ/selenium

# Список Issues
GET https://api.github.com/repos/SeleniumHQ/selenium/issues?state=open&per_page=5

# Пагинация (вторая страница)
GET https://api.github.com/users/octocat/repos?page=2&per_page=5

# Rate Limit информация
GET https://api.github.com/rate_limit
```

### Задания

1. **Без авторизации**: Запросите информацию о пользователе `octocat`. Проверьте поля `login`, `id`, `type`, `public_repos`.
2. **Rate Limit**: Сделайте запрос на `/rate_limit`. Запомните лимит для неавторизованных запросов (60/час). Повторите с токеном — лимит увеличится до 5000/час.
3. **Пагинация**: Получите репозитории организации `google` с `per_page=5`. Проверьте заголовок `Link` — он содержит ссылки на следующую и последнюю страницы.
4. **Поиск**: Найдите Java-репозитории с более чем 10000 звёзд. Убедитесь, что `total_count > 0` и все элементы содержат `language: "Java"`.
5. **Авторизация**: С помощью Personal Access Token получите информацию о своём профиле (`/user`). Проверьте, что `login` соответствует вашему.
6. **Заголовки**: Изучите заголовки ответа: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`. Напишите тест, проверяющий, что `Remaining > 0`.

---

## Сводная таблица

| Навык | JSONPlaceholder | Reqres | PetStore | RestCountries | CoinGecko | OpenWeather | GitHub |
|-------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| GET-запросы | + | + | + | + | + | + | + |
| POST/PUT/DELETE | + | + | + | - | - | - | + |
| Авторизация | - | + | - | - | - | + | + |
| Query Parameters | + | + | + | + | + | + | + |
| Пагинация | - | + | - | - | + | - | + |
| Rate Limiting | - | - | - | - | + | + | + |
| OpenAPI/Swagger | - | - | + | - | - | - | + |
| Фильтрация | + | - | + | + | + | - | + |
| Вложенные ресурсы | + | - | + | - | - | - | + |
| Негативные тесты | + | + | + | + | + | + | + |

---

## Рекомендуемый порядок изучения

1. **JSONPlaceholder** — начните здесь: простой, без авторизации, CRUD
2. **Reqres** — добавьте авторизацию и работу с ошибками
3. **RestCountries** — практикуйте фильтрацию и query parameters
4. **PetStore** — научитесь читать Swagger-документацию
5. **OpenWeatherMap** — работа с API Key
6. **CoinGecko** — rate limiting и реальные данные
7. **GitHub API** — комплексный API с OAuth, пагинацией и headers

---

## Дополнительные ресурсы

- Список бесплатных API: [https://github.com/public-apis/public-apis](https://github.com/public-apis/public-apis)
- HTTP-статус-коды: [https://httpstatuses.com](https://httpstatuses.com)
- HTTP-утилиты для тестирования: [https://httpbin.org](https://httpbin.org)
- Генератор мок-данных: [https://mockaroo.com](https://mockaroo.com)
