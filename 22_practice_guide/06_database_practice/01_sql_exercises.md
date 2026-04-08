# SQL-упражнения для QA-инженера

## Введение

SQL — обязательный навык для QA-инженера. При тестировании мы постоянно проверяем данные
в базе: создалась ли запись, обновились ли поля, удалились ли связанные данные.
В этом руководстве — упражнения от базового SELECT до оконных функций, каждое привязано
к реальному QA-сценарию.

> **Цель:** Освоить SQL на уровне, достаточном для ежедневной работы QA-инженера —
> от простых выборок до сложных аналитических запросов.

---

## Рекомендуемые платформы для практики

| Платформа | Уровень | Особенности |
|-----------|---------|-------------|
| [SQLBolt](https://sqlbolt.com/) | Начальный | Интерактивные уроки с визуализацией |
| [W3Schools SQL](https://www.w3schools.com/sql/trysql.asp?filename=trysql_select_all) | Начальный | Песочница с готовой БД |
| [HackerRank SQL](https://www.hackerrank.com/domains/sql) | Средний | Задачи с автопроверкой |
| [LeetCode Database](https://leetcode.com/problemset/database/) | Продвинутый | Сложные задачи, подготовка к собеседованию |
| [PostgreSQL Exercises](https://pgexercises.com/) | Средний | Задачи на PostgreSQL |
| [SQLZoo](https://sqlzoo.net/) | Начальный-средний | Пошаговые упражнения |

---

## Схема базы данных для упражнений

Все упражнения используют схему блог-платформы (аналог Conduit/RealWorld):

```sql
-- Таблица пользователей
CREATE TABLE users (
    id          SERIAL PRIMARY KEY,
    username    VARCHAR(100) NOT NULL UNIQUE,
    email       VARCHAR(255) NOT NULL UNIQUE,
    password    VARCHAR(255) NOT NULL,
    bio         TEXT,
    image       VARCHAR(500),
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Таблица статей
CREATE TABLE articles (
    id          SERIAL PRIMARY KEY,
    slug        VARCHAR(255) NOT NULL UNIQUE,
    title       VARCHAR(255) NOT NULL,
    description TEXT,
    body        TEXT NOT NULL,
    author_id   INTEGER REFERENCES users(id) ON DELETE CASCADE,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Таблица тегов
CREATE TABLE tags (
    id   SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE
);

-- Связь статей и тегов (many-to-many)
CREATE TABLE article_tags (
    article_id INTEGER REFERENCES articles(id) ON DELETE CASCADE,
    tag_id     INTEGER REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (article_id, tag_id)
);

-- Таблица комментариев
CREATE TABLE comments (
    id         SERIAL PRIMARY KEY,
    body       TEXT NOT NULL,
    article_id INTEGER REFERENCES articles(id) ON DELETE CASCADE,
    author_id  INTEGER REFERENCES users(id) ON DELETE CASCADE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Таблица подписок (follower → following)
CREATE TABLE follows (
    follower_id  INTEGER REFERENCES users(id) ON DELETE CASCADE,
    following_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (follower_id, following_id)
);

-- Таблица лайков (favorites)
CREATE TABLE favorites (
    user_id    INTEGER REFERENCES users(id) ON DELETE CASCADE,
    article_id INTEGER REFERENCES articles(id) ON DELETE CASCADE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, article_id)
);
```

---

## Уровень 1: SELECT — базовые выборки

### QA-контекст

После регистрации нового пользователя через UI вам нужно проверить, что запись
появилась в базе данных с корректными данными.

### Упражнение 1.1: Проверка регистрации пользователя

```sql
-- Задание: найти пользователя по email и проверить его данные
-- QA-сценарий: после регистрации через API проверяем запись в БД

SELECT id, username, email, bio, image, created_at
FROM users
WHERE email = 'newuser@example.com';
```

### Упражнение 1.2: Список всех статей автора

```sql
-- Задание: получить все статьи конкретного пользователя
-- QA-сценарий: на странице профиля отображаются статьи пользователя

SELECT title, slug, description, created_at
FROM articles
WHERE author_id = 1
ORDER BY created_at DESC;
```

### Упражнение 1.3: Поиск недавних статей

```sql
-- Задание: найти статьи, созданные за последние 24 часа
-- QA-сценарий: проверить, что новая статья попадает в ленту "Последние"

SELECT title, slug, created_at
FROM articles
WHERE created_at >= CURRENT_TIMESTAMP - INTERVAL '24 hours'
ORDER BY created_at DESC;
```

### Упражнение 1.4: Проверка уникальности email

```sql
-- Задание: проверить, есть ли дубликаты email
-- QA-сценарий: бизнес-правило — email уникален

SELECT email, COUNT(*) AS count
FROM users
GROUP BY email
HAVING COUNT(*) > 1;

-- Если результат пуст — дубликатов нет, всё корректно
```

---

## Уровень 2: WHERE — фильтрация данных

### Упражнение 2.1: Статьи с пустым описанием

```sql
-- Задание: найти статьи, где description пуст или NULL
-- QA-сценарий: проверяем, не пропускает ли API пустые обязательные поля

SELECT id, title, slug, description
FROM articles
WHERE description IS NULL
   OR description = ''
   OR TRIM(description) = '';
```

### Упражнение 2.2: Пользователи без статей

```sql
-- Задание: найти пользователей, которые зарегистрировались, но не написали ни одной статьи
-- QA-сценарий: проверяем секцию "Нет статей" в профиле

SELECT u.id, u.username, u.email, u.created_at
FROM users u
LEFT JOIN articles a ON u.id = a.author_id
WHERE a.id IS NULL
ORDER BY u.created_at DESC;
```

### Упражнение 2.3: Комментарии к удалённым статьям

```sql
-- Задание: проверить, не осталось ли «висящих» комментариев после удаления статьи
-- QA-сценарий: после удаления статьи комментарии тоже должны удалиться (ON DELETE CASCADE)

SELECT c.id, c.body, c.article_id
FROM comments c
LEFT JOIN articles a ON c.article_id = a.id
WHERE a.id IS NULL;

-- Если есть результаты — баг: ON DELETE CASCADE не работает
```

### Упражнение 2.4: Фильтрация по дате и статусу

```sql
-- Задание: найти пользователей, зарегистрированных в текущем месяце,
-- у которых есть статьи
-- QA-сценарий: проверка функции "Активные новые пользователи"

SELECT DISTINCT u.username, u.email, u.created_at
FROM users u
INNER JOIN articles a ON u.id = a.author_id
WHERE u.created_at >= DATE_TRUNC('month', CURRENT_DATE)
  AND u.created_at < DATE_TRUNC('month', CURRENT_DATE) + INTERVAL '1 month'
ORDER BY u.created_at;
```

---

## Уровень 3: JOIN — объединение таблиц

### Упражнение 3.1: Статьи с именем автора

```sql
-- Задание: получить статьи с именем автора (не author_id)
-- QA-сценарий: на странице статьи отображается имя автора

SELECT a.title,
       a.slug,
       a.description,
       u.username AS author,
       u.image    AS author_image,
       a.created_at
FROM articles a
INNER JOIN users u ON a.author_id = u.id
ORDER BY a.created_at DESC
LIMIT 20;
```

### Упражнение 3.2: Статьи с тегами

```sql
-- Задание: получить статьи со списком тегов
-- QA-сценарий: на карточке статьи отображается список тегов

SELECT a.title,
       a.slug,
       STRING_AGG(t.name, ', ' ORDER BY t.name) AS tags
FROM articles a
LEFT JOIN article_tags at2 ON a.id = at2.article_id
LEFT JOIN tags t ON at2.tag_id = t.id
GROUP BY a.id, a.title, a.slug
ORDER BY a.title;
```

### Упражнение 3.3: Комментарии к статье с авторами

```sql
-- Задание: получить все комментарии к статье с именами авторов
-- QA-сценарий: проверка раздела комментариев под статьёй

SELECT c.id        AS comment_id,
       c.body      AS comment_text,
       u.username  AS comment_author,
       u.image     AS author_image,
       c.created_at
FROM comments c
INNER JOIN users u ON c.author_id = u.id
WHERE c.article_id = (
    SELECT id FROM articles WHERE slug = 'my-article-slug'
)
ORDER BY c.created_at DESC;
```

### Упражнение 3.4: Подписки — кто на кого подписан

```sql
-- Задание: для конкретного пользователя показать, на кого он подписан
-- и кто подписан на него
-- QA-сценарий: проверка страницы профиля — секции Following/Followers

-- На кого подписан пользователь
SELECT u.username AS following
FROM follows f
INNER JOIN users u ON f.following_id = u.id
WHERE f.follower_id = 1;

-- Кто подписан на пользователя
SELECT u.username AS follower
FROM follows f
INNER JOIN users u ON f.follower_id = u.id
WHERE f.following_id = 1;
```

---

## Уровень 4: GROUP BY и агрегатные функции

### Упражнение 4.1: Количество статей у каждого автора

```sql
-- Задание: подсчитать количество статей каждого автора
-- QA-сценарий: проверка счётчика "X articles" в профиле

SELECT u.username,
       COUNT(a.id) AS article_count
FROM users u
LEFT JOIN articles a ON u.id = a.author_id
GROUP BY u.id, u.username
ORDER BY article_count DESC;
```

### Упражнение 4.2: Самые популярные теги

```sql
-- Задание: топ-10 самых популярных тегов
-- QA-сценарий: проверка блока "Popular Tags" на главной странице

SELECT t.name AS tag,
       COUNT(at2.article_id) AS usage_count
FROM tags t
INNER JOIN article_tags at2 ON t.id = at2.tag_id
GROUP BY t.id, t.name
ORDER BY usage_count DESC
LIMIT 10;
```

### Упражнение 4.3: Средняя длина статьи по автору

```sql
-- Задание: средняя длина тела статьи для каждого автора
-- QA-сценарий: аналитика контента

SELECT u.username,
       COUNT(a.id) AS articles,
       ROUND(AVG(LENGTH(a.body))) AS avg_body_length,
       MAX(LENGTH(a.body)) AS max_body_length,
       MIN(LENGTH(a.body)) AS min_body_length
FROM users u
INNER JOIN articles a ON u.id = a.author_id
GROUP BY u.id, u.username
HAVING COUNT(a.id) >= 3
ORDER BY avg_body_length DESC;
```

### Упражнение 4.4: Количество лайков по статьям

```sql
-- Задание: статьи с количеством лайков, отсортированные по популярности
-- QA-сценарий: проверка сортировки "Most Favorited"

SELECT a.title,
       a.slug,
       u.username AS author,
       COUNT(f.user_id) AS favorites_count
FROM articles a
INNER JOIN users u ON a.author_id = u.id
LEFT JOIN favorites f ON a.id = f.article_id
GROUP BY a.id, a.title, a.slug, u.username
ORDER BY favorites_count DESC
LIMIT 20;
```

---

## Уровень 5: Подзапросы

### Упражнение 5.1: Статьи авторов с более чем 5 статьями

```sql
-- Задание: найти статьи только тех авторов, у которых более 5 статей
-- QA-сценарий: фильтр "Активные авторы"

SELECT a.title, a.slug, u.username
FROM articles a
INNER JOIN users u ON a.author_id = u.id
WHERE a.author_id IN (
    SELECT author_id
    FROM articles
    GROUP BY author_id
    HAVING COUNT(*) > 5
)
ORDER BY a.created_at DESC;
```

### Упражнение 5.2: Пользователи, лайкнувшие свои же статьи

```sql
-- Задание: найти пользователей, которые поставили лайк своей собственной статье
-- QA-сценарий: бизнес-правило — разрешено ли это?

SELECT u.username, a.title
FROM favorites f
INNER JOIN articles a ON f.article_id = a.id
INNER JOIN users u ON f.user_id = u.id
WHERE f.user_id = a.author_id;
```

### Упражнение 5.3: Лента подписок (Your Feed)

```sql
-- Задание: получить ленту статей от авторов, на которых подписан пользователь
-- QA-сценарий: проверка вкладки "Your Feed" для пользователя с id=1

SELECT a.title,
       a.slug,
       u.username AS author,
       a.created_at
FROM articles a
INNER JOIN users u ON a.author_id = u.id
WHERE a.author_id IN (
    SELECT following_id
    FROM follows
    WHERE follower_id = 1
)
ORDER BY a.created_at DESC
LIMIT 20;
```

### Упражнение 5.4: Самая обсуждаемая статья каждого автора

```sql
-- Задание: для каждого автора найти статью с наибольшим количеством комментариев
-- QA-сценарий: блок "Best Discussions" в профиле

SELECT u.username,
       a.title,
       comment_counts.cnt AS comments
FROM (
    SELECT article_id,
           COUNT(*) AS cnt,
           ROW_NUMBER() OVER (
               PARTITION BY a2.author_id
               ORDER BY COUNT(*) DESC
           ) AS rn
    FROM comments c
    INNER JOIN articles a2 ON c.article_id = a2.id
    GROUP BY article_id, a2.author_id
) comment_counts
INNER JOIN articles a ON comment_counts.article_id = a.id
INNER JOIN users u ON a.author_id = u.id
WHERE comment_counts.rn = 1
ORDER BY comment_counts.cnt DESC;
```

---

## Уровень 6: Оконные функции (Window Functions)

### Упражнение 6.1: Ранжирование статей по количеству лайков

```sql
-- Задание: присвоить ранг каждой статье по количеству лайков
-- QA-сценарий: проверка корректности позиции в рейтинге

SELECT a.title,
       u.username AS author,
       COUNT(f.user_id) AS likes,
       RANK() OVER (ORDER BY COUNT(f.user_id) DESC) AS rank,
       DENSE_RANK() OVER (ORDER BY COUNT(f.user_id) DESC) AS dense_rank
FROM articles a
INNER JOIN users u ON a.author_id = u.id
LEFT JOIN favorites f ON a.id = f.article_id
GROUP BY a.id, a.title, u.username
ORDER BY rank;
```

### Упражнение 6.2: Нарастающий итог статей по дням

```sql
-- Задание: показать количество статей по дням с нарастающим итогом
-- QA-сценарий: проверка графика "Рост контента" в админ-панели

SELECT DATE(created_at) AS day,
       COUNT(*) AS articles_today,
       SUM(COUNT(*)) OVER (ORDER BY DATE(created_at)) AS total_articles
FROM articles
GROUP BY DATE(created_at)
ORDER BY day;
```

### Упражнение 6.3: Разница во времени между комментариями

```sql
-- Задание: для каждого комментария показать время с момента предыдущего комментария
-- QA-сценарий: анализ активности обсуждения

SELECT c.id,
       c.body,
       u.username,
       c.created_at,
       LAG(c.created_at) OVER (
           PARTITION BY c.article_id
           ORDER BY c.created_at
       ) AS prev_comment_at,
       c.created_at - LAG(c.created_at) OVER (
           PARTITION BY c.article_id
           ORDER BY c.created_at
       ) AS time_since_prev
FROM comments c
INNER JOIN users u ON c.author_id = u.id
WHERE c.article_id = 1
ORDER BY c.created_at;
```

### Упражнение 6.4: Процент статей каждого автора от общего числа

```sql
-- Задание: показать долю статей каждого автора
-- QA-сценарий: аналитика вклада авторов

SELECT u.username,
       COUNT(a.id) AS articles,
       ROUND(
           COUNT(a.id) * 100.0 / SUM(COUNT(a.id)) OVER (),
           2
       ) AS percentage
FROM users u
INNER JOIN articles a ON u.id = a.author_id
GROUP BY u.id, u.username
ORDER BY percentage DESC;
```

---

## Бонус: SQL-запросы для верификации тестов

### Проверка после создания статьи через API

```sql
-- После POST /api/articles проверяем:
-- 1. Статья создана
SELECT COUNT(*) FROM articles WHERE slug = 'test-article-slug';

-- 2. Теги привязаны
SELECT t.name
FROM article_tags at2
INNER JOIN tags t ON at2.tag_id = t.id
INNER JOIN articles a ON at2.article_id = a.id
WHERE a.slug = 'test-article-slug';

-- 3. Автор корректный
SELECT u.username
FROM articles a
INNER JOIN users u ON a.author_id = u.id
WHERE a.slug = 'test-article-slug';
```

### Проверка целостности данных после удаления

```sql
-- После DELETE /api/articles/:slug проверяем каскадное удаление:
-- 1. Статья удалена
SELECT COUNT(*) FROM articles WHERE slug = 'deleted-article';

-- 2. Комментарии удалены
SELECT COUNT(*) FROM comments WHERE article_id = 42;

-- 3. Лайки удалены
SELECT COUNT(*) FROM favorites WHERE article_id = 42;

-- 4. Связи с тегами удалены
SELECT COUNT(*) FROM article_tags WHERE article_id = 42;
```

---

## Практическое задание

### Задание 1: Базовые запросы (уровни 1-2)

1. Зарегистрируйте пользователя через API
2. Напишите SQL-запрос для проверки, что пользователь создан
3. Создайте 3 статьи через API
4. Напишите запрос для получения всех статей этого пользователя
5. Удалите одну статью и проверьте, что комментарии тоже удалились

### Задание 2: Аналитические запросы (уровни 3-4)

1. Напишите запрос для получения топ-5 авторов по количеству лайков
2. Напишите запрос для ленты "Your Feed" конкретного пользователя
3. Найдите статьи без тегов (возможный баг)
4. Подсчитайте среднее количество комментариев на статью

### Задание 3: Сложные запросы (уровни 5-6)

1. Используя оконные функции, создайте рейтинг авторов
2. Напишите запрос для поиска аномалий: статьи с необычно большим количеством лайков
3. Постройте нарастающий итог регистраций по дням

**Критерии оценки:**
- Все запросы выполняются без ошибок
- Результаты соответствуют ожидаемым данным
- Используются корректные JOIN-типы
- Оконные функции применены уместно

---

## Чек-лист самопроверки

- [ ] Могу написать SELECT с WHERE, ORDER BY, LIMIT
- [ ] Понимаю разницу между INNER JOIN, LEFT JOIN, RIGHT JOIN
- [ ] Умею использовать GROUP BY с агрегатными функциями
- [ ] Могу написать подзапрос в WHERE и FROM
- [ ] Знаю оконные функции: ROW_NUMBER, RANK, LAG, SUM OVER
- [ ] Могу написать запрос для проверки каскадного удаления
- [ ] Умею искать аномалии в данных через SQL
- [ ] Практикуюсь на минимум одной из рекомендованных платформ
