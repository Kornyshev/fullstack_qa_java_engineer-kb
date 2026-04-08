# Продвинутый SQL для QA

## Обзор

Базовых SQL-запросов достаточно для ежедневной проверки данных, но реальные задачи QA нередко
требуют более сложных конструкций. Поиск дубликатов, сравнение срезов данных, построение отчётов
с ранжированием, работа с иерархическими данными — всё это требует знания подзапросов,
общих табличных выражений (CTE), оконных функций и операций над множествами.

Этот раздел поднимает SQL-навыки QA-инженера на уровень, достаточный для самостоятельного
анализа данных без привлечения разработчиков или аналитиков.

---

## Подзапросы (Subqueries)

Подзапрос — это `SELECT` внутри другого запроса. Различают несколько видов.

### Скалярный подзапрос (возвращает одно значение)

```sql
-- Найти заказы, сумма которых выше средней
SELECT order_id, total_amount
FROM orders
WHERE total_amount > (
    SELECT AVG(total_amount) FROM orders
);

-- Показать заказ и разницу со средней суммой
SELECT
    order_id,
    total_amount,
    total_amount - (SELECT AVG(total_amount) FROM orders) AS diff_from_avg
FROM orders;
```

### Подзапрос в WHERE (возвращает набор значений)

```sql
-- Заказы от пользователей из Москвы
SELECT * FROM orders
WHERE customer_id IN (
    SELECT user_id FROM users WHERE city = 'Москва'
);

-- Товары, которые ни разу не заказывали (поиск невостребованных товаров)
SELECT * FROM products
WHERE product_id NOT IN (
    SELECT DISTINCT product_id FROM order_items
    WHERE product_id IS NOT NULL  -- важно! NOT IN с NULL даёт пустой результат
);
```

**Внимание:** `NOT IN` ведёт себя неожиданно, если подзапрос возвращает `NULL`-значения.
Если хотя бы одно значение `NULL`, весь `NOT IN` вернёт пустой результат. Всегда
добавляйте `WHERE column IS NOT NULL` или используйте `NOT EXISTS`.

### Подзапрос в FROM (производная таблица)

```sql
-- Средняя сумма заказов по городам клиентов
SELECT
    city_orders.city,
    AVG(city_orders.total_amount) AS avg_city_order
FROM (
    SELECT u.city, o.total_amount
    FROM orders o
    JOIN users u ON o.customer_id = u.user_id
) AS city_orders
GROUP BY city_orders.city
ORDER BY avg_city_order DESC;
```

### Коррелированный подзапрос

Выполняется **для каждой строки** внешнего запроса — обращается к столбцам внешнего запроса.

```sql
-- Последний заказ каждого клиента
SELECT * FROM orders o1
WHERE o1.created_at = (
    SELECT MAX(o2.created_at)
    FROM orders o2
    WHERE o2.customer_id = o1.customer_id  -- ссылка на внешний запрос
);

-- Клиенты, у которых хотя бы один заказ дороже 10 000
SELECT * FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.customer_id = u.user_id
      AND o.total_amount > 10000
);
```

**QA-контекст:** коррелированные подзапросы удобны, когда нужно проверить условие «для каждой
записи существует связанная запись, удовлетворяющая критерию». Например, для каждого пользователя
проверить наличие записи в audit log.

---

## CTE — Common Table Expressions (WITH)

CTE делает запросы читаемыми, разбивая сложную логику на именованные шаги.

```sql
-- Шаг 1: собрать статистику по клиентам
-- Шаг 2: классифицировать клиентов
-- Шаг 3: вывести результат
WITH customer_stats AS (
    SELECT
        customer_id,
        COUNT(*)          AS order_count,
        SUM(total_amount) AS total_spent,
        MAX(created_at)   AS last_order
    FROM orders
    GROUP BY customer_id
),
customer_segments AS (
    SELECT
        cs.*,
        CASE
            WHEN total_spent >= 100000 THEN 'VIP'
            WHEN total_spent >= 10000  THEN 'Regular'
            ELSE 'New'
        END AS segment
    FROM customer_stats cs
)
SELECT
    u.email,
    seg.segment,
    seg.order_count,
    seg.total_spent,
    seg.last_order
FROM customer_segments seg
JOIN users u ON seg.customer_id = u.user_id
ORDER BY seg.total_spent DESC;
```

**QA-сценарий:** при тестировании loyalty-программы удобно через CTE сначала рассчитать сегмент
клиента, а затем проверить, совпадает ли он с тем, что показывает система.

### Рекурсивный CTE

```sql
-- Иерархия категорий товаров (дерево)
WITH RECURSIVE category_tree AS (
    -- Базовый случай: корневые категории
    SELECT category_id, name, parent_id, 1 AS depth
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    -- Рекурсивный шаг: дочерние категории
    SELECT c.category_id, c.name, c.parent_id, ct.depth + 1
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.category_id
)
SELECT * FROM category_tree ORDER BY depth, name;
```

---

## Оконные функции (Window Functions)

Оконные функции выполняют вычисления **по набору строк, связанных с текущей**, не сворачивая
результат в одну строку (в отличие от `GROUP BY`).

### Синтаксис

```sql
-- функция() OVER (
--     PARTITION BY столбец_группировки
--     ORDER BY столбец_сортировки
--     ROWS BETWEEN ... AND ...
-- )
```

### ROW_NUMBER, RANK, DENSE_RANK

```sql
-- Нумерация заказов каждого клиента по дате
SELECT
    customer_id,
    order_id,
    created_at,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at) AS order_num
FROM orders;
```

Разница между функциями при одинаковых значениях:

```
Значение | ROW_NUMBER | RANK | DENSE_RANK
---------|------------|------|-----------
   100   |     1      |   1  |     1
   100   |     2      |   1  |     1
    90   |     3      |   3  |     2       -- RANK пропускает 2, DENSE_RANK — нет
    80   |     4      |   4  |     3
```

```sql
-- Найти последний заказ каждого клиента (альтернатива коррелированному подзапросу)
WITH ranked_orders AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY created_at DESC
        ) AS rn
    FROM orders
)
SELECT * FROM ranked_orders WHERE rn = 1;
```

**QA-сценарий:** проверка, что система показывает именно последний заказ клиента на dashboard.

### LAG и LEAD — доступ к соседним строкам

```sql
-- Время между заказами клиента (для анализа активности)
SELECT
    customer_id,
    order_id,
    created_at,
    LAG(created_at) OVER (
        PARTITION BY customer_id ORDER BY created_at
    ) AS prev_order_date,
    created_at - LAG(created_at) OVER (
        PARTITION BY customer_id ORDER BY created_at
    ) AS days_between_orders
FROM orders;
```

```sql
-- Изменение цены товара: текущая vs предыдущая
SELECT
    product_id,
    price,
    effective_date,
    LAG(price) OVER (PARTITION BY product_id ORDER BY effective_date) AS prev_price,
    price - LAG(price) OVER (PARTITION BY product_id ORDER BY effective_date) AS price_change
FROM product_prices;
```

### SUM / AVG OVER — накопительные вычисления

```sql
-- Нарастающий итог продаж по дням (running total)
SELECT
    DATE(created_at)   AS order_date,
    SUM(total_amount)  AS daily_total,
    SUM(SUM(total_amount)) OVER (ORDER BY DATE(created_at)) AS running_total
FROM orders
GROUP BY DATE(created_at)
ORDER BY order_date;

-- Скользящее среднее за 7 дней
SELECT
    DATE(created_at) AS order_date,
    COUNT(*)         AS daily_orders,
    AVG(COUNT(*)) OVER (
        ORDER BY DATE(created_at)
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS avg_7_days
FROM orders
GROUP BY DATE(created_at);
```

---

## CASE WHEN — условная логика

```sql
-- Классификация заказов по сумме
SELECT
    order_id,
    total_amount,
    CASE
        WHEN total_amount >= 10000 THEN 'Large'
        WHEN total_amount >= 1000  THEN 'Medium'
        ELSE 'Small'
    END AS order_size
FROM orders;

-- Подсчёт заказов по статусам в одной строке (pivot)
SELECT
    customer_id,
    COUNT(*) FILTER (WHERE status = 'COMPLETED')  AS completed,   -- PostgreSQL
    COUNT(*) FILTER (WHERE status = 'CANCELLED')   AS cancelled,
    COUNT(*) FILTER (WHERE status = 'PENDING')     AS pending
FROM orders
GROUP BY customer_id;

-- Альтернатива через CASE (работает везде)
SELECT
    customer_id,
    SUM(CASE WHEN status = 'COMPLETED' THEN 1 ELSE 0 END) AS completed,
    SUM(CASE WHEN status = 'CANCELLED' THEN 1 ELSE 0 END) AS cancelled,
    SUM(CASE WHEN status = 'PENDING'   THEN 1 ELSE 0 END) AS pending
FROM orders
GROUP BY customer_id;
```

**QA-контекст:** `CASE WHEN` + `GROUP BY` позволяют быстро построить сводную таблицу для
анализа данных перед или после тестового прогона.

---

## UNION / INTERSECT / EXCEPT — операции над множествами

### UNION и UNION ALL

```sql
-- Объединить клиентов из двух систем (удаляя дубликаты)
SELECT email, first_name FROM users_old
UNION
SELECT email, first_name FROM users_new;

-- UNION ALL: оставить дубликаты (быстрее, так как не нужна дедупликация)
SELECT email, first_name FROM users_old
UNION ALL
SELECT email, first_name FROM users_new;
```

### INTERSECT — пересечение

```sql
-- Email, которые есть в обеих системах (мигрированные пользователи)
SELECT email FROM users_old
INTERSECT
SELECT email FROM users_new;
```

### EXCEPT — разность

```sql
-- Пользователи, которые есть в старой системе, но отсутствуют в новой
-- (потерянные при миграции)
SELECT email FROM users_old
EXCEPT
SELECT email FROM users_new;
```

**QA-сценарий:** при тестировании миграции данных `EXCEPT` незаменим — позволяет
мгновенно найти записи, которые не перенеслись.

---

## EXISTS — проверка существования

```sql
-- Клиенты, у которых есть хотя бы один заказ
SELECT * FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = u.user_id
);

-- Клиенты без заказов (альтернатива LEFT JOIN ... IS NULL)
SELECT * FROM users u
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = u.user_id
);
```

`EXISTS` vs `IN`:
- `EXISTS` останавливается при первом найденном совпадении — эффективнее на больших подзапросах
- `IN` загружает весь список значений в память
- `NOT EXISTS` корректно работает с `NULL`, а `NOT IN` — нет

---

## Практические QA-сценарии

### Поиск дубликатов

```sql
-- Найти дублирующиеся email в таблице пользователей
SELECT email, COUNT(*) AS cnt
FROM users
GROUP BY email
HAVING COUNT(*) > 1;

-- Детально: показать все записи-дубликаты с их id
WITH duplicates AS (
    SELECT email
    FROM users
    GROUP BY email
    HAVING COUNT(*) > 1
)
SELECT u.*
FROM users u
JOIN duplicates d ON u.email = d.email
ORDER BY u.email, u.user_id;
```

### Сравнение данных между таблицами

```sql
-- Проверка, что сумма позиций заказа совпадает с total_amount
WITH order_item_totals AS (
    SELECT
        order_id,
        SUM(quantity * unit_price) AS calculated_total
    FROM order_items
    GROUP BY order_id
)
SELECT
    o.order_id,
    o.total_amount     AS stored_total,
    oit.calculated_total,
    o.total_amount - oit.calculated_total AS discrepancy
FROM orders o
JOIN order_item_totals oit ON o.order_id = oit.order_id
WHERE ABS(o.total_amount - oit.calculated_total) > 0.01;  -- допуск на округление
```

### Генерация тестового отчёта из БД

```sql
-- Ежедневная статистика заказов за последний месяц
WITH daily_stats AS (
    SELECT
        DATE(created_at)    AS day,
        COUNT(*)            AS total_orders,
        SUM(CASE WHEN status = 'COMPLETED' THEN 1 ELSE 0 END) AS completed,
        SUM(CASE WHEN status = 'CANCELLED' THEN 1 ELSE 0 END) AS cancelled,
        SUM(CASE WHEN status = 'FAILED'    THEN 1 ELSE 0 END) AS failed,
        ROUND(AVG(total_amount), 2) AS avg_amount
    FROM orders
    WHERE created_at >= CURRENT_DATE - INTERVAL '30 days'
    GROUP BY DATE(created_at)
)
SELECT
    day,
    total_orders,
    completed,
    cancelled,
    failed,
    ROUND(100.0 * completed / NULLIF(total_orders, 0), 1) AS success_rate_pct,
    avg_amount
FROM daily_stats
ORDER BY day;
```

---

## Связь с тестированием

| Задача QA                                 | Инструмент SQL                          |
|-------------------------------------------|-----------------------------------------|
| Найти дубликаты после импорта             | `GROUP BY` + `HAVING COUNT(*) > 1`      |
| Проверить миграцию данных                 | `EXCEPT`, `INTERSECT`                   |
| Найти последнюю запись для каждой группы  | `ROW_NUMBER() OVER (PARTITION BY ...)`  |
| Сравнить расчётные и хранимые значения    | CTE + `JOIN` + разность                |
| Анализ трендов (скользящее среднее)       | `AVG() OVER (ROWS BETWEEN ...)`         |
| Проверка наличия связанных записей        | `EXISTS` / `NOT EXISTS`                 |
| Pivot-отчёт по статусам                   | `CASE WHEN` + `SUM`                     |

---

## Типичные ошибки

1. **`NOT IN` с подзапросом, содержащим `NULL`** — результат всегда пуст. Используйте
   `NOT EXISTS` или добавьте `WHERE column IS NOT NULL` в подзапрос.

2. **Путаница `UNION` и `UNION ALL`** — `UNION` удаляет дубликаты (медленнее), `UNION ALL`
   оставляет все строки. Если дубликаты не мешают, используйте `UNION ALL`.

3. **Оконная функция в `WHERE`** — нельзя писать `WHERE ROW_NUMBER() OVER (...) = 1`.
   Оберните в CTE или подзапрос.

4. **Коррелированный подзапрос на большой таблице** — может быть крайне медленным. Рассмотрите
   замену на `JOIN` или оконную функцию.

5. **Забыть `PARTITION BY` в оконной функции** — без `PARTITION BY` функция работает
   по всему набору данных, а не по группам. `ROW_NUMBER() OVER (ORDER BY ...)` нумерует
   все строки, а не строки внутри каждой группы.

6. **Неверный порядок столбцов в `UNION`** — столбцы должны совпадать по количеству и типам
   в каждом `SELECT`. Имена берутся из первого запроса.

7. **`CASE` без `ELSE`** — если ни одно условие не сработает, результат будет `NULL`.
   Всегда явно указывайте `ELSE`, если `NULL` нежелателен.

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Что такое подзапрос? Где его можно использовать?
2. Чем `UNION` отличается от `UNION ALL`?
3. Для чего нужен оператор `EXISTS`?
4. Что такое `CASE WHEN` и когда его применять?
5. Как найти дубликаты в таблице?

### 🟡 Средний уровень
6. Объясните разницу между `ROW_NUMBER`, `RANK` и `DENSE_RANK`.
7. Что такое CTE и чем он лучше подзапроса?
8. Напишите запрос: для каждого клиента найти его самый дорогой заказ.
9. Как с помощью SQL проверить целостность данных после миграции?
10. Что такое коррелированный подзапрос? Приведите пример.

### 🔴 Продвинутый уровень
11. Напишите запрос с оконной функцией для расчёта скользящего среднего за 7 дней.
12. Почему `NOT IN` с подзапросом, возвращающим `NULL`, даёт пустой результат?
13. Как построить рекурсивный CTE для обхода древовидной структуры данных?
14. В чём разница производительности `EXISTS` vs `IN` vs `JOIN`? Когда что выбрать?
15. Как с помощью `EXCEPT` автоматизировать регрессионное тестирование данных?

---

## Практические задания

### Задание 1. Поиск потерянных записей при миграции
Имеются таблицы `orders_v1` (старая система) и `orders_v2` (новая). Найдите заказы, которые
не мигрировались.

<details>
<summary>Решение</summary>

```sql
-- Заказы, которые есть в v1, но отсутствуют в v2
SELECT order_id FROM orders_v1
EXCEPT
SELECT old_order_id FROM orders_v2;

-- Или через NOT EXISTS для детальной информации
SELECT * FROM orders_v1 o1
WHERE NOT EXISTS (
    SELECT 1 FROM orders_v2 o2
    WHERE o2.old_order_id = o1.order_id
);
```

</details>

### Задание 2. Проверка расчётов
Проверьте, что `total_amount` в таблице `orders` равен сумме `quantity * price` из `order_items`.

<details>
<summary>Решение</summary>

```sql
WITH calculated AS (
    SELECT
        order_id,
        SUM(quantity * unit_price) AS calc_total
    FROM order_items
    GROUP BY order_id
)
SELECT
    o.order_id,
    o.total_amount,
    c.calc_total,
    o.total_amount - c.calc_total AS diff
FROM orders o
JOIN calculated c ON o.order_id = c.order_id
WHERE ABS(o.total_amount - c.calc_total) > 0.01;
```

</details>

### Задание 3. Последний заказ каждого клиента
Выведите информацию о последнем заказе каждого клиента.

<details>
<summary>Решение</summary>

```sql
WITH ranked AS (
    SELECT
        o.*,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY created_at DESC
        ) AS rn
    FROM orders o
)
SELECT order_id, customer_id, status, total_amount, created_at
FROM ranked
WHERE rn = 1;
```

</details>

### Задание 4. Динамика заказов
Для каждого дня рассчитайте количество заказов и изменение по сравнению с предыдущим днём.

<details>
<summary>Решение</summary>

```sql
WITH daily AS (
    SELECT
        DATE(created_at) AS day,
        COUNT(*)         AS order_count
    FROM orders
    GROUP BY DATE(created_at)
)
SELECT
    day,
    order_count,
    LAG(order_count) OVER (ORDER BY day) AS prev_day_count,
    order_count - LAG(order_count) OVER (ORDER BY day) AS change
FROM daily
ORDER BY day;
```

</details>

### Задание 5. Сводный отчёт по сегментам
Классифицируйте клиентов на VIP (>100k), Regular (>10k), New (<10k) и покажите количество
клиентов в каждом сегменте и средний чек.

<details>
<summary>Решение</summary>

```sql
WITH customer_totals AS (
    SELECT
        customer_id,
        SUM(total_amount) AS lifetime_value,
        AVG(total_amount) AS avg_order
    FROM orders
    GROUP BY customer_id
),
segmented AS (
    SELECT
        *,
        CASE
            WHEN lifetime_value >= 100000 THEN 'VIP'
            WHEN lifetime_value >= 10000  THEN 'Regular'
            ELSE 'New'
        END AS segment
    FROM customer_totals
)
SELECT
    segment,
    COUNT(*)                    AS customer_count,
    ROUND(AVG(avg_order), 2)   AS avg_order_value,
    ROUND(AVG(lifetime_value), 2) AS avg_lifetime_value
FROM segmented
GROUP BY segment
ORDER BY avg_lifetime_value DESC;
```

</details>

---

## Дополнительные ресурсы

- [PostgreSQL Window Functions](https://www.postgresql.org/docs/current/tutorial-window.html) — официальная документация по оконным функциям
- [Modern SQL](https://modern-sql.com/) — продвинутые возможности SQL с примерами
- [Window Functions Explained](https://www.windowfunctions.com/) — интерактивный тренажёр оконных функций
- [SQLZoo](https://sqlzoo.net/) — практические задания на SQL разной сложности
- [HackerRank SQL](https://www.hackerrank.com/domains/sql) — задачи на SQL с проверкой
- [Explain PostgreSQL](https://explain.dalibo.com/) — визуализатор планов выполнения запросов
