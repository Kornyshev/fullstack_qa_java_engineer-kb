# HTTP протокол глубже

## Обзор

HTTP (HyperText Transfer Protocol) — основной протокол взаимодействия между клиентом и сервером в вебе.
Для QA-инженера глубокое понимание HTTP — это не просто теория, а ежедневный рабочий инструмент.
Каждый API-тест, каждый анализ бага в Network tab, каждый отчёт о дефекте связан с HTTP.
В этом разделе мы разберём HTTP/1.1 vs HTTP/2, методы, заголовки, cookies, кэширование, CORS
и другие механизмы, критически важные для тестирования.

---

## HTTP/1.1 vs HTTP/2

### HTTP/1.1 (1997)

- **Текстовый протокол** — заголовки и тело передаются в виде текста
- **Одно соединение — один запрос** (или Keep-Alive с очередью)
- **Head-of-line blocking** — следующий запрос ждёт завершения предыдущего
- **Нет сжатия заголовков** — заголовки повторяются в каждом запросе

```
GET /api/users HTTP/1.1        ← Текстовая строка запроса
Host: example.com              ← Обязательный заголовок
Accept: application/json
Authorization: Bearer token123
```

### HTTP/2 (2015)

- **Бинарный протокол** — данные передаются в бинарном формате (frames)
- **Мультиплексирование** — множество запросов по одному TCP-соединению одновременно
- **Server Push** — сервер может отправить ресурсы до того, как клиент их запросит
- **Сжатие заголовков (HPACK)** — экономия трафика при повторяющихся заголовках
- **Приоритизация потоков** — важные ресурсы загружаются первыми

### Сравнение для QA

| Аспект | HTTP/1.1 | HTTP/2 |
|--------|----------|--------|
| Формат | Текстовый | Бинарный |
| Параллельные запросы | До 6 соединений к домену | Неограниченно в одном соединении |
| Заголовки | Без сжатия | HPACK-сжатие |
| Анализ в DevTools | Читается напрямую | Нужен декодер |
| TLS | Опционально | Фактически обязателен |
| Влияние на тесты | Простой анализ трафика | Меньше latency, другие patterns |

**Что важно для тестирования:**
- В HTTP/2 ошибки могут проявляться иначе из-за мультиплексирования
- Производительность приложения существенно отличается на HTTP/1.1 и HTTP/2
- При тестировании нагрузки необходимо учитывать версию протокола

---

## HTTP-методы: глубокое погружение

### Основные методы

| Метод | Назначение | Idempotent | Safe | Body |
|-------|-----------|------------|------|------|
| **GET** | Получение ресурса | Да | Да | Нет (обычно) |
| **POST** | Создание ресурса | Нет | Нет | Да |
| **PUT** | Полная замена ресурса | Да | Нет | Да |
| **PATCH** | Частичное обновление | Нет* | Нет | Да |
| **DELETE** | Удаление ресурса | Да | Нет | Нет (обычно) |
| **HEAD** | Как GET, но без тела ответа | Да | Да | Нет |
| **OPTIONS** | Запрос допустимых методов | Да | Да | Нет |

**Idempotent** — повторный вызов даёт тот же результат. PUT отправленный 5 раз — ресурс в том же
состоянии. POST отправленный 5 раз — 5 новых ресурсов.

**Safe** — не изменяет состояние сервера. GET только читает данные.

### PUT vs PATCH: ключевое различие

```http
# PUT — полная замена ресурса (нужно передать ВСЕ поля)
PUT /api/users/1 HTTP/1.1
Content-Type: application/json

{
  "name": "Иван Иванов",
  "email": "ivan@example.com",
  "phone": "+7-999-123-4567",
  "address": "Москва"
}

# PATCH — частичное обновление (только изменённые поля)
PATCH /api/users/1 HTTP/1.1
Content-Type: application/json

{
  "phone": "+7-999-765-4321"
}
```

**Что тестировать:**
- PUT без обязательного поля — должен вернуть 400
- PATCH с несуществующим полем — поведение зависит от реализации
- Idempotency: повторный PUT не создаёт дублей

---

## HTTP Headers: важнейшие заголовки для QA

### Content-Type

Определяет формат тела запроса/ответа.

```http
# JSON — самый распространённый для REST API
Content-Type: application/json

# Отправка формы
Content-Type: application/x-www-form-urlencoded

# Загрузка файлов
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary

# XML (SOAP, legacy API)
Content-Type: application/xml

# Plain text
Content-Type: text/plain; charset=utf-8
```

**Что тестировать:**
- Отправка запроса с неправильным Content-Type (например, `text/plain` вместо `application/json`)
- Отсутствие Content-Type в запросе
- Несоответствие Content-Type и фактического содержимого
- Charset: кириллица может ломаться при неправильной кодировке

### Authorization

Передаёт credentials для аутентификации.

```http
# Bearer Token (JWT) — самый распространённый для современных API
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIn0.dozjgNryP4J3jVmNHl0w5N_XgL0n3I9PlFUP0THsR8U

# Basic Auth — логин:пароль в Base64
Authorization: Basic dXNlcjpwYXNzd29yZA==

# API Key (нестандартный, но частый)
X-API-Key: abc123def456
```

**Что тестировать:**
- Запрос без Authorization → 401 Unauthorized
- Просроченный token → 401 Unauthorized
- Невалидный token → 401 Unauthorized
- Token другого пользователя → 403 Forbidden
- Token с недостаточными правами → 403 Forbidden

### Cache-Control

Управляет кэшированием ответов.

```http
# Запрет кэширования
Cache-Control: no-store

# Кэширование на 1 час
Cache-Control: max-age=3600

# Кэширование с обязательной ревалидацией
Cache-Control: no-cache

# Только приватный кэш (браузер), не CDN
Cache-Control: private, max-age=600

# Публичный кэш (CDN может кэшировать)
Cache-Control: public, max-age=86400
```

**Разница `no-cache` и `no-store`:**
- `no-cache` — кэшировать можно, но перед использованием нужно проверить актуальность на сервере
- `no-store` — не кэшировать вообще (для sensitive data)

### ETag и условные запросы

ETag — «отпечаток» ресурса для проверки актуальности кэша.

```http
# Сервер возвращает ETag в ответе
HTTP/1.1 200 OK
ETag: "abc123"
Content-Type: application/json

{"name": "Иван"}

# Клиент отправляет условный запрос
GET /api/users/1 HTTP/1.1
If-None-Match: "abc123"

# Если ресурс не изменился — 304 Not Modified (без тела)
HTTP/1.1 304 Not Modified

# Если ресурс изменился — 200 OK с новыми данными и новым ETag
HTTP/1.1 200 OK
ETag: "def456"
```

**Что тестировать:**
- Сервер возвращает ETag в ответе
- Условный запрос с актуальным ETag возвращает 304
- Условный запрос с устаревшим ETag возвращает 200 с новыми данными

### CORS Headers

```http
# Ответ сервера, разрешающий кросс-доменные запросы
Access-Control-Allow-Origin: https://frontend.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 86400
```

(CORS подробнее — в отдельном разделе ниже.)

---

## Cookies

### Установка Cookie

```http
# Сервер устанавливает cookie
HTTP/1.1 200 OK
Set-Cookie: session_id=abc123; Path=/; HttpOnly; Secure; SameSite=Strict; Max-Age=3600
Set-Cookie: theme=dark; Path=/; Max-Age=31536000
```

### Атрибуты Cookie

| Атрибут | Назначение | Пример |
|---------|-----------|--------|
| **Path** | Путь, для которого cookie доступна | `Path=/api` |
| **Domain** | Домен, для которого cookie доступна | `Domain=.example.com` |
| **Max-Age** | Время жизни в секундах | `Max-Age=3600` |
| **Expires** | Дата истечения | `Expires=Thu, 01 Jan 2026 00:00:00 GMT` |
| **HttpOnly** | Недоступна из JavaScript | Защита от XSS |
| **Secure** | Передаётся только по HTTPS | Защита от перехвата |
| **SameSite** | Контроль отправки при кросс-доменных запросах | `Strict`, `Lax`, `None` |

### SameSite подробнее

- **Strict** — cookie не отправляется при любых кросс-доменных запросах (даже при клике по ссылке)
- **Lax** (по умолчанию) — отправляется при навигации (GET по ссылке), но не при POST/AJAX
- **None** — отправляется всегда (требует `Secure`)

**Что тестировать:**
- Cookie устанавливается с правильными атрибутами
- HttpOnly cookie недоступна через `document.cookie` в браузере
- Secure cookie не передаётся по HTTP (только HTTPS)
- Session cookie удаляется при закрытии браузера (без Max-Age/Expires)
- SameSite=Strict блокирует cookie при кросс-доменном POST

---

## Кэширование: механизмы

### Уровни кэширования

```
Браузер (Browser Cache)
    ↓
CDN (Content Delivery Network)
    ↓
Reverse Proxy (Nginx)
    ↓
Application Cache (Redis)
    ↓
Database Cache (Query Cache)
```

### Flow принятия решения о кэшировании

```
Запрос → Есть ли ответ в кэше?
            │
            ├── Нет → Запрос к серверу → Кэшировать ответ → Вернуть клиенту
            │
            └── Да → Кэш свежий (max-age не истёк)?
                        │
                        ├── Да → Вернуть из кэша (без запроса к серверу)
                        │
                        └── Нет → Условный запрос (If-None-Match / If-Modified-Since)
                                    │
                                    ├── 304 Not Modified → Использовать кэш
                                    │
                                    └── 200 OK → Обновить кэш, вернуть новые данные
```

**Что тестировать:**
- Статические ресурсы (JS, CSS, изображения) кэшируются с длинным `max-age`
- API-ответы с персональными данными имеют `Cache-Control: no-store` или `private`
- После обновления контента пользователь видит новую версию (cache busting)
- CDN корректно инвалидирует кэш

---

## CORS (Cross-Origin Resource Sharing)

### Что такое CORS

CORS — механизм, который позволяет браузеру делать запросы к домену, отличному от того, на котором
находится frontend. Без CORS браузер блокирует кросс-доменные запросы (Same-Origin Policy).

```
Frontend: https://app.example.com
Backend:  https://api.example.com    ← Другой origin!

Браузер блокирует запрос, если сервер не вернёт правильные CORS-заголовки.
```

### Simple Request vs Preflight Request

**Simple Request** (не требует preflight):
- Метод: GET, HEAD, POST
- Headers: только стандартные (Accept, Content-Type с ограничениями)
- Content-Type: `text/plain`, `multipart/form-data`, `application/x-www-form-urlencoded`

**Preflight Request** (браузер сначала отправляет OPTIONS):
- Метод: PUT, DELETE, PATCH
- Или нестандартные заголовки (Authorization, X-Custom-Header)
- Или Content-Type: `application/json`

```http
# 1. Preflight запрос (автоматически отправляется браузером)
OPTIONS /api/users HTTP/1.1
Host: api.example.com
Origin: https://app.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type, Authorization

# 2. Preflight ответ от сервера
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400

# 3. Основной запрос (если preflight прошёл)
POST /api/users HTTP/1.1
Host: api.example.com
Origin: https://app.example.com
Content-Type: application/json
Authorization: Bearer token123

{"name": "Иван"}

# 4. Основной ответ
HTTP/1.1 201 Created
Access-Control-Allow-Origin: https://app.example.com
```

### Что тестировать (CORS)

- Запрос с разрешённого origin — проходит
- Запрос с неразрешённого origin — блокируется браузером
- Preflight кэшируется (Access-Control-Max-Age) — OPTIONS не отправляется повторно
- `Access-Control-Allow-Credentials: true` не может использоваться с `Allow-Origin: *`
- Ответ содержит корректный `Access-Control-Allow-Origin` (не wildcard `*` для authenticated requests)

**Важно:** CORS — это механизм **браузера**. Postman и curl не проверяют CORS. Баг с CORS
проявится только при тестировании через реальный браузер.

---

## Content Negotiation

Content Negotiation — механизм, позволяющий клиенту и серверу договориться о формате данных.

```http
# Клиент запрашивает предпочтительный формат
GET /api/users/1 HTTP/1.1
Accept: application/json, application/xml;q=0.9, text/plain;q=0.5

# q=0.9 означает приоритет 0.9 (из 1.0)
# Приоритет: JSON (1.0) > XML (0.9) > Text (0.5)
```

```http
# Запрос сжатого контента
GET /api/data HTTP/1.1
Accept-Encoding: gzip, deflate, br

# Ответ со сжатием
HTTP/1.1 200 OK
Content-Encoding: gzip
```

```http
# Запрос на определённом языке
GET /api/products/1 HTTP/1.1
Accept-Language: ru-RU, ru;q=0.9, en;q=0.5
```

**Что тестировать:**
- Сервер возвращает данные в запрошенном формате (JSON, XML)
- При неподдерживаемом формате — 406 Not Acceptable
- Сжатие работает корректно (gzip)
- Локализация ответов соответствует Accept-Language

---

## DevTools Network Tab: что анализировать

### Ключевые колонки

| Колонка | Что показывает |
|---------|---------------|
| **Name** | URL запроса |
| **Status** | HTTP status code (200, 301, 404, 500) |
| **Type** | Тип ресурса (xhr, fetch, document, script, stylesheet) |
| **Initiator** | Что инициировало запрос (скрипт, redirect) |
| **Size** | Размер ответа (или "from cache") |
| **Time** | Общее время запроса |
| **Waterfall** | Визуальная timeline запроса |

### Waterfall: фазы запроса

```
DNS Lookup     → Резолв доменного имени в IP
TCP Connection → Установка TCP-соединения
TLS Handshake  → Установка защищённого соединения
Request Sent   → Отправка HTTP-запроса
Waiting (TTFB) → Ожидание первого байта ответа (Time To First Byte)
Content Download → Загрузка тела ответа
```

**Что искать при анализе:**
- Долгий TTFB → проблема на сервере (медленный запрос к БД, тяжёлая логика)
- Долгий DNS Lookup → проблемы с DNS
- Много запросов к одному домену → HTTP/1.1 bottleneck
- Большой размер ответа → нужна пагинация или сжатие
- Запросы со статусом `(from cache)` → данные из кэша, не с сервера

---

## Связь с тестированием

Глубокое знание HTTP позволяет QA-инженеру:

1. **Писать точные API-тесты** — проверять не только body, но и headers, status codes, cookies
2. **Быстро находить баги** — анализ Network tab за секунды показывает причину проблемы
3. **Тестировать безопасность** — CORS, HttpOnly cookies, Authorization
4. **Тестировать производительность** — кэширование, сжатие, HTTP/2
5. **Составлять качественные баг-репорты** — с curl-примером для воспроизведения

---

## Типичные ошибки

1. **Игнорирование заголовков** — тестирование только body, без проверки Content-Type, Cache-Control
2. **Не тестируют CORS** — баг проявляется только в браузере, Postman не покажет
3. **Не проверяют HttpOnly на session cookie** — уязвимость для XSS-атак
4. **Путают 401 и 403** — 401 = «не аутентифицирован», 403 = «нет прав»
5. **Не тестируют кэширование** — пользователь видит устаревшие данные
6. **Забывают про preflight** — OPTIONS-запрос не настроен на сервере, PUT/DELETE не работают
7. **Не проверяют Content-Type** — сервер принимает `text/plain` вместо `application/json`
8. **Не тестируют сжатие** — ответ в 10 МБ без gzip замедляет приложение

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Назовите основные HTTP-методы и их назначение.
2. В чём разница между PUT и PATCH?
3. Что такое idempotent метод? Приведите примеры.
4. Какие status codes вы знаете? Что означают 200, 201, 400, 401, 403, 404, 500?
5. Что такое Content-Type?

### 🟡 Средний уровень
6. В чём разница между HTTP/1.1 и HTTP/2? Как это влияет на тестирование?
7. Объясните механизм CORS. Что такое preflight request?
8. В чём разница между `Cache-Control: no-cache` и `no-store`?
9. Какие атрибуты cookies вы знаете? Зачем нужен HttpOnly?
10. Что такое ETag и как работают условные запросы?
11. Чем отличается аутентификация через cookie и через Bearer token?

### 🔴 Продвинутый уровень
12. Как тестировать кэширование на уровне CDN?
13. Объясните content negotiation. Что произойдёт, если сервер не поддерживает запрошенный формат?
14. Как SameSite cookie влияет на безопасность? В чём разница Strict, Lax и None?
15. Как анализировать waterfall в DevTools для диагностики проблем производительности?
16. Почему CORS не защищает от атак через curl/Postman?

---

## Практические задания

### Задание 1: Анализ заголовков
Откройте DevTools → Network на любом сайте. Найдите:
- Запрос с Bearer token в Authorization
- Ответ с Set-Cookie (проверьте атрибуты)
- Ответ с Cache-Control (определите стратегию кэширования)
- Preflight OPTIONS-запрос

### Задание 2: Тестирование HTTP-методов
Используя Postman или curl, протестируйте REST API:
```bash
# GET — получение списка
curl -v https://jsonplaceholder.typicode.com/posts

# POST — создание ресурса
curl -v -X POST https://jsonplaceholder.typicode.com/posts \
  -H "Content-Type: application/json" \
  -d '{"title": "Test", "body": "Content", "userId": 1}'

# PUT — полная замена
curl -v -X PUT https://jsonplaceholder.typicode.com/posts/1 \
  -H "Content-Type: application/json" \
  -d '{"id": 1, "title": "Updated", "body": "New content", "userId": 1}'

# PATCH — частичное обновление
curl -v -X PATCH https://jsonplaceholder.typicode.com/posts/1 \
  -H "Content-Type: application/json" \
  -d '{"title": "Partially Updated"}'

# DELETE — удаление
curl -v -X DELETE https://jsonplaceholder.typicode.com/posts/1
```
Для каждого запроса зафиксируйте: status code, response headers, response body.

### Задание 3: CORS-тестирование
Создайте простой HTML-файл с `fetch`-запросом к стороннему API. Откройте в браузере и проверьте:
- Работает ли запрос?
- Есть ли CORS-ошибка в Console?
- Отправляется ли preflight OPTIONS?

### Задание 4: Кэширование
Используя curl с флагами `-v` и `-H "If-None-Match: ..."`, протестируйте условные запросы
к API, которое поддерживает ETag. Убедитесь, что получаете 304 Not Modified.

---

## Дополнительные ресурсы

- [MDN: HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP)
- [MDN: CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
- [HTTP/2 Explained](https://http2-explained.haxx.se/)
- [RFC 7231: HTTP Semantics](https://datatracker.ietf.org/doc/html/rfc7231)
- [Chrome DevTools: Network Tab](https://developer.chrome.com/docs/devtools/network/)
- [HTTP Caching (web.dev)](https://web.dev/http-cache/)
