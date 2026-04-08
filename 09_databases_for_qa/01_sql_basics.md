# Основы SQL для QA

## Обзор

SQL (Structured Query Language) — основной язык для работы с реляционными базами данных.
Для QA-инженера владение SQL — это не опция, а необходимость. Каждый день тестировщик проверяет,
корректно ли приложение записывает данные в базу, правильно ли формирует выборки, не теряются ли записи
при обновлении. Без SQL эти проверки превращаются в угадывание.

В этом разделе рассматриваются базовые операции: выборка данных, фильтрация, сортировка, группировка,
объединение таблиц, а также операции модификации данных. Все примеры построены на типичных QA-сценариях.

---

## SELECT и FROM — получение данных

Оператор `SELECT` определяет, **какие столбцы** вернуть, а `FROM` указывает **откуда** брать данные.

```sql
-- Получить все столбцы из таблицы заказов
SELECT * FROM orders;

-- Получить только нужные столбцы (хорошая практика для QA — выбирать конкретные поля)
SELECT order_id, customer_id, status, total_amount
FROM orders;

-- Использование алиасов для читаемости отчётов
SELECT
    o.order_id    AS "Номер заказа",
    o.status      AS "Статус",
    o.created_at  AS "Дата создания"
FROM orders o;
```

**QA-контекст:** при проверке API-ответа полезно выполнить `SELECT` к таблице и сравнить результат
с тем, что вернул endpoint. Если есть расхождения — это баг.

---

## WHERE — фильтрация строк

### Операторы сравнения

```sql
-- Заказы со статусом 'COMPLETED'
SELECT * FROM orders WHERE status = 'COMPLETED';

-- Заказы на сумму больше 1000
SELECT * FROM orders WHERE total_amount > 1000;

-- Заказы, созданные после определённой даты
SELECT * FROM orders WHERE created_at >= '2025-01-01';
```

### Логические операторы AND, OR, NOT

```sql
-- Заказы в статусе 'PENDING', созданные за последнюю неделю
SELECT * FROM orders
WHERE status = 'PENDING'
  AND created_at >= CURRENT_DATE - INTERVAL '7 days';

-- Заказы со статусом 'CANCELLED' или 'REFUNDED'
SELECT * FROM orders
WHERE status = 'CANCELLED' OR status = 'REFUNDED';

-- Все заказы, КРОМЕ удалённых
SELECT * FROM orders
WHERE NOT is_deleted;
```

### LIKE — поиск по шаблону

```sql
-- Пользователи, чей email заканчивается на '@test.com' (тестовые аккаунты)
SELECT * FROM users WHERE email LIKE '%@test.com';

-- Пользователи, чьё имя начинается с 'Test'
SELECT * FROM users WHERE first_name LIKE 'Test%';

-- Поиск по маске: один символ — _
SELECT * FROM users WHERE phone LIKE '+7___555____';
```

### IN — проверка принадлежности к набору

```sql
-- Заказы с определёнными статусами
SELECT * FROM orders
WHERE status IN ('PENDING', 'PROCESSING', 'SHIPPED');

-- Пользователи из определённых городов
SELECT * FROM users
WHERE city IN ('Москва', 'Санкт-Петербург', 'Казань');
```

### BETWEEN — проверка диапазона

```sql
-- Заказы на сумму от 500 до 2000 (включительно обе границы)
SELECT * FROM orders
WHERE total_amount BETWEEN 500 AND 2000;

-- Заказы за определённый период
SELECT * FROM orders
WHERE created_at BETWEEN '2025-01-01' AND '2025-01-31';
```

### IS NULL / IS NOT NULL

```sql
-- Пользователи без указанного телефона (поле пустое)
SELECT * FROM users WHERE phone IS NULL;

-- Заказы, которым назначен курьер
SELECT * FROM orders WHERE courier_id IS NOT NULL;
```

**Частая ошибка QA:** писать `WHERE phone = NULL` вместо `WHERE phone IS NULL`.
В SQL `NULL` — это не значение, а отсутствие значения. Сравнение через `=` всегда даёт `NULL`, не `TRUE`.

---

## ORDER BY — сортировка

```sql
-- Заказы по дате создания (от новых к старым)
SELECT * FROM orders ORDER BY created_at DESC;

-- Сортировка по нескольким столбцам
SELECT * FROM orders
ORDER BY status ASC, total_amount DESC;
```

---

## LIMIT и OFFSET — ограничение выборки

```sql
-- Получить первые 10 заказов (полезно при проверке пагинации)
SELECT * FROM orders ORDER BY created_at DESC LIMIT 10;

-- Пропустить 10 и взять следующие 10 (вторая страница)
SELECT * FROM orders ORDER BY created_at DESC LIMIT 10 OFFSET 10;
```

**QA-контекст:** при тестировании пагинации на UI выполните запрос к базе с `LIMIT` и `OFFSET`,
соответствующими текущей странице, и сравните результат с тем, что отображается в интерфейсе.

---

## JOIN — объединение таблиц

Объединения позволяют получать данные из нескольких связанных таблиц одним запросом.

### INNER JOIN — только совпадающие записи

```
Таблица A       Таблица B
  +---+           +---+
  |   |  +-----+  |   |
  |   |  |INNER|  |   |
  |   |  +-----+  |   |
  +---+           +---+
```

```sql
-- Заказы с информацией о пользователях (только те заказы, у которых есть пользователь)
SELECT o.order_id, u.first_name, u.email, o.total_amount
FROM orders o
INNER JOIN users u ON o.customer_id = u.user_id;
```

### LEFT JOIN — все записи из левой таблицы

```
Таблица A       Таблица B
  +---+           +---+
  |   +--+-----+  |   |
  |LEFT|  INNER|  |   |
  |   +--+-----+  |   |
  +---+           +---+
```

```sql
-- Все пользователи и их заказы (включая тех, кто ещё ничего не заказал)
SELECT u.user_id, u.email, o.order_id, o.status
FROM users u
LEFT JOIN orders o ON u.user_id = o.customer_id;

-- Найти пользователей БЕЗ заказов (проверка: нет ли "мёртвых" аккаунтов)
SELECT u.user_id, u.email
FROM users u
LEFT JOIN orders o ON u.user_id = o.customer_id
WHERE o.order_id IS NULL;
```

### RIGHT JOIN — все записи из правой таблицы

```
Таблица A       Таблица B
  +---+           +---+
  |   +-----+--+ |   |
  |   |INNER|RIGHT   |
  |   +-----+--+ |   |
  +---+           +---+
```

```sql
-- Все заказы и информация о курьерах (включая заказы без курьера — NULL)
SELECT o.order_id, o.status, c.courier_name
FROM couriers c
RIGHT JOIN orders o ON c.courier_id = o.courier_id;
```

### FULL OUTER JOIN — все записи из обеих таблиц

```
Таблица A       Таблица B
  +---+           +---+
  |   +--+-----+-+   |
  |LEFT| INNER |RIGHT|
  |   +--+-----+-+   |
  +---+           +---+
```

```sql
-- Полное соединение: все пользователи и все заказы
SELECT u.user_id, u.email, o.order_id
FROM users u
FULL OUTER JOIN orders o ON u.user_id = o.customer_id;
```

**QA-контекст:** `LEFT JOIN ... WHERE right.id IS NULL` — мощный приём для поиска "осиротевших"
записей. Например, можно найти позиции заказа, ссылающиеся на несуществующий товар.

---

## GROUP BY и HAVING — агрегация

### GROUP BY — группировка строк

```sql
-- Количество заказов по статусам
SELECT status, COUNT(*) AS order_count
FROM orders
GROUP BY status;

-- Сумма продаж по каждому клиенту
SELECT customer_id, SUM(total_amount) AS total_spent
FROM orders
GROUP BY customer_id;
```

### Агрегатные функции

| Функция   | Назначение                  | Пример QA-использования                         |
|-----------|-----------------------------|-------------------------------------------------|
| `COUNT()` | Количество строк            | Сколько заказов в каждом статусе                |
| `SUM()`   | Сумма значений              | Общая сумма продаж за период                    |
| `AVG()`   | Среднее арифметическое      | Средний чек заказа                               |
| `MIN()`   | Минимальное значение        | Самый ранний заказ                               |
| `MAX()`   | Максимальное значение       | Самый дорогой заказ                              |

```sql
-- Статистика по заказам (QA-отчёт)
SELECT
    COUNT(*)                        AS total_orders,
    COUNT(DISTINCT customer_id)     AS unique_customers,
    SUM(total_amount)               AS revenue,
    AVG(total_amount)               AS avg_order_value,
    MIN(total_amount)               AS min_order,
    MAX(total_amount)               AS max_order
FROM orders
WHERE created_at >= '2025-01-01';
```

### HAVING — фильтрация после группировки

```sql
-- Клиенты, сделавшие более 5 заказов (потенциальные кандидаты для loyalty-программы)
SELECT customer_id, COUNT(*) AS order_count
FROM orders
GROUP BY customer_id
HAVING COUNT(*) > 5;

-- Статусы, в которых «застряло» много заказов (возможная проблема)
SELECT status, COUNT(*) AS cnt
FROM orders
GROUP BY status
HAVING COUNT(*) > 100
ORDER BY cnt DESC;
```

**Разница WHERE и HAVING:**
- `WHERE` фильтрует **строки до** группировки
- `HAVING` фильтрует **группы после** агрегации

---

## INSERT — вставка данных

```sql
-- Создание тестового пользователя
INSERT INTO users (first_name, last_name, email, phone)
VALUES ('Test', 'User', 'test_user_001@test.com', '+70001112233');

-- Массовая вставка тестовых данных
INSERT INTO users (first_name, last_name, email)
VALUES
    ('Test', 'Alpha', 'alpha@test.com'),
    ('Test', 'Beta', 'beta@test.com'),
    ('Test', 'Gamma', 'gamma@test.com');
```

**QA-контекст:** при подготовке тестовых данных используйте уникальные маркеры (например, email
с доменом `@test.com`), чтобы потом легко было их найти и удалить.

---

## UPDATE — обновление данных

```sql
-- Перевести заказ в статус 'CANCELLED' (подготовка данных для теста отмены)
UPDATE orders
SET status = 'CANCELLED', updated_at = NOW()
WHERE order_id = 12345;

-- Массовое обновление: пометить все тестовые заказы
UPDATE orders
SET status = 'TEST'
WHERE customer_id IN (
    SELECT user_id FROM users WHERE email LIKE '%@test.com'
);
```

**Важно:** всегда добавляйте `WHERE` в `UPDATE`. Без него обновятся **все** строки таблицы!

---

## DELETE — удаление данных

```sql
-- Удалить конкретного тестового пользователя
DELETE FROM users WHERE email = 'test_user_001@test.com';

-- Удалить все тестовые записи после прогона тестов
DELETE FROM orders WHERE customer_id IN (
    SELECT user_id FROM users WHERE email LIKE '%@test.com'
);
DELETE FROM users WHERE email LIKE '%@test.com';
```

**Важно:** порядок удаления имеет значение из-за foreign key constraints. Сначала удаляйте
дочерние записи (заказы), потом родительские (пользователи).

---

## Связь с тестированием

| Задача QA                          | SQL-инструмент                              |
|------------------------------------|---------------------------------------------|
| Проверка данных после API-запроса  | `SELECT ... WHERE`                          |
| Подготовка тестовых данных         | `INSERT INTO`                               |
| Изменение состояния для теста      | `UPDATE ... SET ... WHERE`                  |
| Очистка после тестов               | `DELETE FROM ... WHERE`                     |
| Проверка целостности данных        | `JOIN` + `IS NULL`                          |
| Отчёт о тестовом покрытии данных   | `GROUP BY` + `COUNT`                        |
| Тестирование пагинации             | `LIMIT` + `OFFSET`                          |

---

## Типичные ошибки

1. **`WHERE column = NULL`** — надо `IS NULL`. Сравнение с `NULL` через `=` даёт `NULL`, а не `TRUE`.

2. **`UPDATE` / `DELETE` без `WHERE`** — обновляются или удаляются все строки таблицы.
   Всегда сначала выполните `SELECT` с тем же `WHERE`, чтобы убедиться в корректности условия.

3. **Путаница `WHERE` и `HAVING`** — `WHERE` работает до группировки, `HAVING` — после.
   `WHERE COUNT(*) > 5` вызовет ошибку; нужно `HAVING COUNT(*) > 5`.

4. **Забыть про `ORDER BY` при `LIMIT`** — без сортировки результат `LIMIT` не детерминирован.
   Записи могут приходить в произвольном порядке.

5. **Путаница `LEFT JOIN` и `INNER JOIN`** — `INNER JOIN` отбрасывает строки без совпадений,
   что может скрыть проблемы с данными. Для поиска несоответствий используйте `LEFT JOIN`.

6. **Использование `SELECT *` в проде** — для ad-hoc проверок допустимо, но в тестовом коде
   лучше явно перечислять нужные столбцы: схема может измениться.

7. **Неверный порядок `DELETE` при foreign keys** — попытка удалить родительскую запись
   при наличии дочерних вызовет ошибку constraint violation.

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Чем отличается `WHERE` от `HAVING`?
2. Какие типы `JOIN` вы знаете? Объясните разницу между `INNER` и `LEFT JOIN`.
3. Что вернёт `SELECT * FROM users WHERE name = NULL`?
4. Как получить количество записей в таблице?
5. Для чего нужны `LIMIT` и `OFFSET`?

### 🟡 Средний уровень
6. Напишите запрос, который найдёт пользователей без заказов.
7. Чем `COUNT(*)` отличается от `COUNT(column_name)`?
8. Как найти дубликаты в таблице?
9. В каком порядке СУБД обрабатывает части SELECT-запроса (порядок выполнения)?
10. Как безопасно подготовить и очистить тестовые данные через SQL?

### 🔴 Продвинутый уровень
11. Почему `LEFT JOIN` + `WHERE right.col = 'value'` может работать как `INNER JOIN`? Как исправить?
12. Объясните разницу между `UNION` и `UNION ALL`. Когда что использовать?
13. Как протестировать, что при concurrent `INSERT` не нарушается уникальность?
14. Что такое execution plan и зачем QA может быть полезно его смотреть?

---

## Практические задания

### Задание 1. Проверка данных после регистрации
После вызова API `POST /api/users` с телом `{"email": "new@test.com", "name": "New User"}`
напишите SQL-запрос, который проверит, что пользователь создан корректно.

<details>
<summary>Решение</summary>

```sql
SELECT user_id, email, first_name, created_at
FROM users
WHERE email = 'new@test.com';

-- Проверяем:
-- 1. Запись существует (не пустой результат)
-- 2. first_name = 'New User'
-- 3. created_at примерно соответствует текущему времени
```

</details>

### Задание 2. Поиск осиротевших заказов
Найдите заказы, которые ссылаются на несуществующих пользователей (нарушение целостности данных).

<details>
<summary>Решение</summary>

```sql
SELECT o.order_id, o.customer_id
FROM orders o
LEFT JOIN users u ON o.customer_id = u.user_id
WHERE u.user_id IS NULL;
```

</details>

### Задание 3. Отчёт для QA
Подготовьте отчёт: для каждого статуса заказа покажите количество, среднюю сумму и дату последнего заказа.

<details>
<summary>Решение</summary>

```sql
SELECT
    status,
    COUNT(*)            AS order_count,
    ROUND(AVG(total_amount), 2) AS avg_amount,
    MAX(created_at)     AS last_order_date
FROM orders
GROUP BY status
ORDER BY order_count DESC;
```

</details>

### Задание 4. Подготовка и очистка тестовых данных
Напишите скрипт создания тестовых данных (пользователь + заказ) и скрипт очистки после теста.

<details>
<summary>Решение</summary>

```sql
-- === SETUP ===
INSERT INTO users (user_id, first_name, email)
VALUES (99999, 'AutoTest', 'autotest_setup@test.com');

INSERT INTO orders (order_id, customer_id, status, total_amount)
VALUES (99999, 99999, 'PENDING', 150.00);

-- === CLEANUP (порядок важен из-за foreign keys!) ===
DELETE FROM orders WHERE order_id = 99999;
DELETE FROM users WHERE user_id = 99999;
```

</details>

### Задание 5. Тестирование пагинации
API возвращает заказы по 20 на страницу. Напишите SQL для проверки данных на 3-й странице.

<details>
<summary>Решение</summary>

```sql
-- Страница 3: пропустить 40 записей (2 * 20), взять 20
SELECT order_id, status, total_amount, created_at
FROM orders
ORDER BY created_at DESC
LIMIT 20 OFFSET 40;
```

</details>

---

## Дополнительные ресурсы

- [SQLBolt](https://sqlbolt.com/) — интерактивный тренажёр SQL с упражнениями
- [PostgreSQL Tutorial](https://www.postgresqltutorial.com/) — подробное руководство по PostgreSQL
- [W3Schools SQL](https://www.w3schools.com/sql/) — справочник по SQL с примерами
- [LeetCode SQL](https://leetcode.com/problemset/database/) — задачи на SQL для подготовки к интервью
- [Use The Index, Luke](https://use-the-index-luke.com/) — понимание индексов и производительности SQL
- [SQL Style Guide](https://www.sqlstyle.guide/) — руководство по стилю SQL-запросов
