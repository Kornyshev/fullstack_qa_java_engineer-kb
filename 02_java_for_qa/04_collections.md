# Коллекции Java

## Обзор

Коллекции — фундамент работы с данными в Java и неотъемлемая часть тестовой автоматизации.
Хранение тестовых данных, наборов ожидаемых результатов, маппинг конфигурации, управление очередями задач —
всё это реализуется через коллекции. Знание внутреннего устройства `HashMap`, различий между `ArrayList`
и `LinkedList`, выбор правильной структуры данных — важный навык для QA-инженера, который пишет
производительные и надёжные автотесты. В этом разделе подробно разобрана иерархия коллекций,
внутреннее устройство ключевых структур и практические паттерны их использования в тестовом коде.

---

## Иерархия коллекций

```
Iterable
  └── Collection
        ├── List (упорядоченная коллекция с дубликатами)
        │     ├── ArrayList
        │     ├── LinkedList
        │     ├── Vector (устаревший)
        │     └── CopyOnWriteArrayList
        │
        ├── Set (коллекция без дубликатов)
        │     ├── HashSet
        │     ├── LinkedHashSet
        │     ├── TreeSet
        │     ├── EnumSet
        │     └── CopyOnWriteArraySet
        │
        └── Queue (очередь)
              ├── LinkedList
              ├── PriorityQueue
              ├── ArrayDeque
              └── ConcurrentLinkedQueue

Map (не наследуется от Collection!)
  ├── HashMap
  ├── LinkedHashMap
  ├── TreeMap
  ├── Hashtable (устаревший)
  ├── ConcurrentHashMap
  └── EnumMap
```

---

## List — упорядоченная коллекция

### ArrayList vs LinkedList

| Операция              | ArrayList        | LinkedList       |
|-----------------------|------------------|------------------|
| Доступ по индексу     | **O(1)**         | O(n)             |
| Вставка в конец       | **O(1)** amortized| O(1)            |
| Вставка в начало/середину| O(n)          | **O(1)** *       |
| Удаление по индексу   | O(n)             | O(n) **          |
| Поиск по значению     | O(n)             | O(n)             |
| Потребление памяти    | **Меньше**       | Больше (ссылки)  |

\* O(1) — только если уже есть ссылка на узел. Поиск узла — O(n).
\** Удаление = поиск O(n) + удаление узла O(1).

**Правило для QA:** в 95% случаев используйте `ArrayList`. `LinkedList` оправдан только
при частых вставках/удалениях в начале списка и работе как Queue/Deque.

### ArrayList в тестах

```java
@Test
void arrayListInTests() {
    // Создание списка тестовых данных
    List<String> testUrls = new ArrayList<>();
    testUrls.add("https://example.com/api/users");
    testUrls.add("https://example.com/api/orders");
    testUrls.add("https://example.com/api/products");

    // Неизменяемый список (Java 9+) — удобен для expected values
    List<String> expectedStatuses = List.of("active", "pending", "completed");

    // Проверка размера
    Assertions.assertEquals(3, testUrls.size());

    // Проверка содержимого
    Assertions.assertTrue(testUrls.contains("https://example.com/api/users"));

    // Доступ по индексу — O(1) для ArrayList
    Assertions.assertEquals("https://example.com/api/orders", testUrls.get(1));
}
```

### Создание списков: разные способы

```java
@Test
void listCreationMethods() {
    // 1. List.of() — неизменяемый список (Java 9+)
    List<String> immutable = List.of("chrome", "firefox", "edge");
    Assertions.assertThrows(UnsupportedOperationException.class,
            () -> immutable.add("safari"));

    // 2. List.copyOf() — неизменяемая копия (Java 10+)
    List<String> mutableSource = new ArrayList<>(List.of("a", "b"));
    List<String> immutableCopy = List.copyOf(mutableSource);

    // 3. Arrays.asList() — фиксированный размер, изменяемые элементы
    List<String> fixedSize = Arrays.asList("a", "b", "c");
    fixedSize.set(0, "x"); // OK — можно менять элементы
    Assertions.assertThrows(UnsupportedOperationException.class,
            () -> fixedSize.add("d")); // Нельзя менять размер

    // 4. new ArrayList<>() — полностью изменяемый
    List<String> mutable = new ArrayList<>(List.of("a", "b"));
    mutable.add("c");
    mutable.remove("a");
    Assertions.assertEquals(List.of("b", "c"), mutable);
}
```

### Сортировка и фильтрация

```java
@Test
void sortingAndFiltering() {
    // Список результатов тестов
    List<TestResult> results = new ArrayList<>();
    results.add(new TestResult("Login test", 1200, true));
    results.add(new TestResult("Search test", 3500, false));
    results.add(new TestResult("Cart test", 800, true));
    results.add(new TestResult("Checkout test", 5000, false));

    // Сортировка по времени выполнения
    results.sort(Comparator.comparingLong(TestResult::duration));
    Assertions.assertEquals("Cart test", results.get(0).name());

    // Фильтрация: только провалившиеся тесты
    List<TestResult> failures = results.stream()
            .filter(r -> !r.passed())
            .toList(); // Java 16+

    Assertions.assertEquals(2, failures.size());

    // Извлечение имён провалившихся тестов
    List<String> failedNames = failures.stream()
            .map(TestResult::name)
            .toList();

    Assertions.assertTrue(failedNames.contains("Search test"));
    Assertions.assertTrue(failedNames.contains("Checkout test"));
}

// Record для результата теста
record TestResult(String name, long duration, boolean passed) {}
```

---

## Set — коллекция без дубликатов

### HashSet vs TreeSet vs LinkedHashSet

| Характеристика       | HashSet          | TreeSet          | LinkedHashSet     |
|-----------------------|------------------|------------------|-------------------|
| Порядок элементов     | Не гарантирован  | Отсортированный  | Порядок вставки   |
| `add()`               | **O(1)**         | O(log n)         | O(1)              |
| `contains()`          | **O(1)**         | O(log n)         | O(1)              |
| `remove()`            | **O(1)**         | O(log n)         | O(1)              |
| null-элементы         | Допускает один   | Не допускает     | Допускает один    |
| Реализация            | HashMap          | Красно-чёрное дерево | HashMap + связный список|

### Set в тестах

```java
@Test
void setInTests() {
    // Проверка уникальности элементов в ответе API
    List<String> userIds = getUserIdsFromApi();
    Set<String> uniqueIds = new HashSet<>(userIds);

    // Если размеры совпадают — все ID уникальны
    Assertions.assertEquals(userIds.size(), uniqueIds.size(),
            "Обнаружены дубликаты в ID пользователей!");

    // LinkedHashSet сохраняет порядок вставки — полезно для проверки порядка
    Set<String> orderedStatuses = new LinkedHashSet<>();
    orderedStatuses.add("created");
    orderedStatuses.add("processing");
    orderedStatuses.add("completed");

    List<String> asList = new ArrayList<>(orderedStatuses);
    Assertions.assertEquals("created", asList.get(0));
    Assertions.assertEquals("completed", asList.get(2));

    // TreeSet — отсортированный набор
    Set<Integer> sortedCodes = new TreeSet<>(List.of(500, 200, 404, 301, 201));
    // Порядок: 200, 201, 301, 404, 500
    Assertions.assertEquals(200, sortedCodes.iterator().next());
}
```

### Операции над множествами

```java
@Test
void setOperations() {
    Set<String> expectedPermissions = Set.of("read", "write", "delete", "admin");
    Set<String> actualPermissions = Set.of("read", "write", "execute");

    // Пересечение — общие элементы
    Set<String> intersection = new HashSet<>(expectedPermissions);
    intersection.retainAll(actualPermissions);
    Assertions.assertEquals(Set.of("read", "write"), intersection);

    // Разность — чего не хватает
    Set<String> missing = new HashSet<>(expectedPermissions);
    missing.removeAll(actualPermissions);
    Assertions.assertEquals(Set.of("delete", "admin"), missing);

    // Объединение
    Set<String> union = new HashSet<>(expectedPermissions);
    union.addAll(actualPermissions);
    Assertions.assertEquals(5, union.size()); // read, write, delete, admin, execute

    // Проверка подмножества
    Set<String> requiredMinimum = Set.of("read", "write");
    Assertions.assertTrue(actualPermissions.containsAll(requiredMinimum),
            "Не все обязательные права присутствуют");
}
```

---

## Map — словарь ключ-значение

### HashMap: внутреннее устройство

HashMap — самая часто используемая реализация Map. Понимание её устройства важно для интервью.

**Основные концепции:**

1. **Массив бакетов (buckets)** — внутренний массив `Node<K,V>[]`, начальный размер — 16.
2. **Хеш-функция** — определяет индекс бакета: `index = hash(key) & (capacity - 1)`.
3. **Коллизия** — два ключа попадают в один бакет. Решение — связный список, при длине >= 8 — красно-чёрное дерево (treeification, Java 8+).
4. **Load factor** — порог заполненности (по умолчанию 0.75). При превышении — rehashing (увеличение массива в 2 раза).
5. **Rehashing** — пересчёт хешей и перераспределение элементов по новому массиву.

```
HashMap внутри (упрощённо):

buckets[]:
  [0] -> null
  [1] -> Node(key="user1", value=User1) -> null
  [2] -> null
  [3] -> Node(key="user3", value=User3) -> Node(key="user7", value=User7) -> null  // коллизия!
  [4] -> null
  ...
  [15] -> Node(key="user15", value=User15) -> null
```

**Требования к ключам HashMap:**
- Корректная реализация `hashCode()` и `equals()`
- Immutability ключей (желательно)
- Если `a.equals(b)`, то `a.hashCode() == b.hashCode()` (обязательно)

### HashMap vs TreeMap vs LinkedHashMap

| Характеристика       | HashMap          | TreeMap          | LinkedHashMap     |
|-----------------------|------------------|------------------|-------------------|
| Порядок ключей        | Не гарантирован  | Отсортированный  | Порядок вставки   |
| `get()` / `put()`    | **O(1)**         | O(log n)         | O(1)              |
| null-ключи            | Допускает один   | Не допускает     | Допускает один    |
| Реализация            | Хеш-таблица      | Красно-чёрное дерево | Хеш-таблица + связный список|
| Применение в тестах   | Хранение данных  | Сортированная конфигурация | Порядок шагов |

### Map в тестах

```java
@Test
void mapInTests() {
    // HashMap — хранение тестовых данных
    Map<String, String> testUser = new HashMap<>();
    testUser.put("username", "admin");
    testUser.put("password", "secret123");
    testUser.put("role", "ADMIN");

    Assertions.assertEquals("admin", testUser.get("username"));
    Assertions.assertTrue(testUser.containsKey("role"));

    // Map.of() — неизменяемая карта (Java 9+, до 10 пар)
    Map<String, Integer> expectedCodes = Map.of(
            "/users", 200,
            "/admin", 403,
            "/nonexistent", 404
    );

    // Map.ofEntries() — для более 10 пар
    Map<String, Integer> moreEntries = Map.ofEntries(
            Map.entry("/users", 200),
            Map.entry("/orders", 200),
            Map.entry("/products", 200)
    );

    // LinkedHashMap — сохраняет порядок вставки (полезно для шагов теста)
    Map<String, Runnable> testSteps = new LinkedHashMap<>();
    testSteps.put("Открыть страницу", () -> driver.get(url));
    testSteps.put("Ввести логин", () -> loginPage.enterUsername("admin"));
    testSteps.put("Ввести пароль", () -> loginPage.enterPassword("pass"));
    testSteps.put("Нажать войти", () -> loginPage.clickLogin());

    // Шаги выполняются в порядке вставки
    testSteps.forEach((step, action) -> {
        System.out.println("Шаг: " + step);
        action.run();
    });
}
```

### Полезные методы Map (Java 8+)

```java
@Test
void mapModernMethods() {
    Map<String, Integer> testRunCounts = new HashMap<>();

    // putIfAbsent — добавить, только если ключа нет
    testRunCounts.putIfAbsent("LoginTest", 0);
    testRunCounts.putIfAbsent("LoginTest", 99); // Не перезапишет
    Assertions.assertEquals(0, testRunCounts.get("LoginTest"));

    // getOrDefault — значение по умолчанию
    int count = testRunCounts.getOrDefault("UnknownTest", -1);
    Assertions.assertEquals(-1, count);

    // computeIfAbsent — ленивая инициализация
    Map<String, List<String>> testsByCategory = new HashMap<>();
    testsByCategory.computeIfAbsent("smoke", k -> new ArrayList<>())
            .add("LoginTest");
    testsByCategory.computeIfAbsent("smoke", k -> new ArrayList<>())
            .add("SearchTest");
    Assertions.assertEquals(2, testsByCategory.get("smoke").size());

    // merge — объединение значений
    Map<String, Integer> failCounts = new HashMap<>();
    failCounts.merge("LoginTest", 1, Integer::sum); // 1
    failCounts.merge("LoginTest", 1, Integer::sum); // 2
    failCounts.merge("LoginTest", 1, Integer::sum); // 3
    Assertions.assertEquals(3, failCounts.get("LoginTest"));

    // replaceAll — массовая замена значений
    Map<String, String> config = new HashMap<>(Map.of(
            "baseUrl", "http://localhost",
            "apiUrl", "http://localhost/api"
    ));
    config.replaceAll((key, value) -> value.replace("http://", "https://"));
    Assertions.assertEquals("https://localhost", config.get("baseUrl"));
}
```

---

## Queue — очередь

### Основные реализации

```java
@Test
void queueInTests() {
    // ArrayDeque — быстрее LinkedList для стека и очереди
    Queue<String> testQueue = new ArrayDeque<>();
    testQueue.offer("Test 1"); // Добавить в конец
    testQueue.offer("Test 2");
    testQueue.offer("Test 3");

    // Получить и удалить первый элемент
    String first = testQueue.poll();
    Assertions.assertEquals("Test 1", first);
    Assertions.assertEquals(2, testQueue.size());

    // Посмотреть первый элемент без удаления
    String peeked = testQueue.peek();
    Assertions.assertEquals("Test 2", peeked);
    Assertions.assertEquals(2, testQueue.size()); // Размер не изменился

    // PriorityQueue — приоритетная очередь (сортирует элементы)
    Queue<Integer> priorityQueue = new PriorityQueue<>();
    priorityQueue.offer(30);
    priorityQueue.offer(10);
    priorityQueue.offer(20);

    // Извлечение в порядке приоритета (натуральный порядок)
    Assertions.assertEquals(10, priorityQueue.poll());
    Assertions.assertEquals(20, priorityQueue.poll());
    Assertions.assertEquals(30, priorityQueue.poll());
}
```

### Deque как стек

```java
@Test
void dequeAsStack() {
    // Deque (double-ended queue) может работать как стек (LIFO)
    Deque<String> callStack = new ArrayDeque<>();

    // push = добавить в начало (вершину стека)
    callStack.push("Открыть главную страницу");
    callStack.push("Перейти в каталог");
    callStack.push("Открыть товар");

    // pop = удалить с вершины стека
    Assertions.assertEquals("Открыть товар", callStack.pop());
    Assertions.assertEquals("Перейти в каталог", callStack.pop());
    Assertions.assertEquals("Открыть главную страницу", callStack.pop());
}
```

---

## Паттерны итерации

```java
@Test
void iterationPatterns() {
    List<String> browsers = List.of("chrome", "firefox", "edge", "safari");

    // 1. for-each (самый читаемый)
    for (String browser : browsers) {
        System.out.println("Тестируем в: " + browser);
    }

    // 2. forEach с lambda
    browsers.forEach(browser -> System.out.println("Браузер: " + browser));

    // 3. forEach с method reference
    browsers.forEach(System.out::println);

    // 4. Stream API (фильтрация + преобразование)
    List<String> chromiumBased = browsers.stream()
            .filter(b -> b.equals("chrome") || b.equals("edge"))
            .map(String::toUpperCase)
            .toList();
    Assertions.assertEquals(List.of("CHROME", "EDGE"), chromiumBased);

    // 5. Итерация по Map
    Map<String, Integer> results = Map.of("Test1", 200, "Test2", 404, "Test3", 500);

    for (Map.Entry<String, Integer> entry : results.entrySet()) {
        System.out.printf("%s -> %d%n", entry.getKey(), entry.getValue());
    }

    // 6. Итерация только по ключам или значениям
    for (String testName : results.keySet()) {
        System.out.println(testName);
    }

    for (Integer code : results.values()) {
        Assertions.assertTrue(code > 0);
    }
}
```

---

## Concurrent-коллекции для параллельных тестов

При параллельном запуске тестов стандартные коллекции небезопасны. Необходимо использовать
потокобезопасные (thread-safe) альтернативы.

| Обычная коллекция   | Concurrent-аналог             | Особенности                          |
|---------------------|-------------------------------|--------------------------------------|
| `ArrayList`         | `CopyOnWriteArrayList`        | Быстрое чтение, медленная запись     |
| `HashSet`           | `CopyOnWriteArraySet`         | Быстрое чтение, медленная запись     |
| `HashMap`           | `ConcurrentHashMap`           | Сегментированная блокировка          |
| `LinkedList` (Queue)| `ConcurrentLinkedQueue`       | Lock-free очередь                    |
| `TreeMap`           | `ConcurrentSkipListMap`       | Отсортированная concurrent-карта     |

```java
// Потокобезопасный сбор результатов при параллельном запуске
public class ParallelTestCollector {
    // ConcurrentHashMap — безопасен для параллельного доступа
    private final Map<String, TestResult> results = new ConcurrentHashMap<>();

    // CopyOnWriteArrayList — безопасен для параллельного добавления
    private final List<String> logs = new CopyOnWriteArrayList<>();

    public void addResult(String testName, TestResult result) {
        results.put(testName, result);
        logs.add("[" + testName + "] " + result.status());
    }

    public Map<String, TestResult> getResults() {
        return Collections.unmodifiableMap(results);
    }

    public List<String> getLogs() {
        return Collections.unmodifiableList(logs);
    }
}

@Test
void concurrentCollectionExample() {
    Map<String, String> sharedConfig = new ConcurrentHashMap<>();
    sharedConfig.put("baseUrl", "https://api.test.com");
    sharedConfig.put("timeout", "30");

    // Безопасно использовать из нескольких потоков
    // (например, при параллельных тестах в TestNG/JUnit 5)
    String url = sharedConfig.get("baseUrl");
    Assertions.assertEquals("https://api.test.com", url);
}
```

### Collections.unmodifiable* vs Collections.synchronized*

```java
@Test
void collectionsWrappers() {
    List<String> original = new ArrayList<>(List.of("a", "b", "c"));

    // Неизменяемая обёртка — защита от случайного изменения
    List<String> unmodifiable = Collections.unmodifiableList(original);
    Assertions.assertThrows(UnsupportedOperationException.class,
            () -> unmodifiable.add("d"));

    // НО! Изменения оригинала отражаются в обёртке
    original.add("d");
    Assertions.assertEquals(4, unmodifiable.size()); // 4!

    // Синхронизированная обёртка — потокобезопасность
    List<String> synced = Collections.synchronizedList(new ArrayList<>());
    synced.add("test1");
    synced.add("test2");

    // Итерация требует явной синхронизации
    synchronized (synced) {
        for (String s : synced) {
            System.out.println(s);
        }
    }
}
```

---

## Когда что использовать в тестах

| Задача                                | Рекомендуемая коллекция       |
|---------------------------------------|-------------------------------|
| Список тестовых данных                | `ArrayList` или `List.of()`   |
| Проверка уникальности                 | `HashSet`                     |
| Хранение конфигурации ключ-значение   | `HashMap` или `Map.of()`      |
| Упорядоченные шаги теста              | `LinkedHashMap`               |
| Отсортированные данные                | `TreeSet` / `TreeMap`         |
| Очередь задач                         | `ArrayDeque`                  |
| Параллельные тесты — сбор результатов | `ConcurrentHashMap`           |
| Параллельные тесты — список логов     | `CopyOnWriteArrayList`        |
| Неизменяемые expected values          | `List.of()`, `Set.of()`, `Map.of()` |

---

## Связь с тестированием

| Концепция                   | Применение в QA                                         |
|-----------------------------|---------------------------------------------------------|
| `ArrayList`                 | Хранение тестовых данных, списки URL, locators          |
| `HashSet`                   | Проверка уникальности ID, дедупликация данных           |
| `HashMap`                   | Конфигурация тестов, заголовки HTTP, параметры запросов  |
| `LinkedHashMap`             | Упорядоченные шаги теста, сохранение порядка данных      |
| `TreeMap`                   | Отсортированные отчёты, конфигурация по алфавиту        |
| `ConcurrentHashMap`         | Сбор результатов при параллельном запуске тестов         |
| `Collections.unmodifiable*` | Защита тестовых данных от случайного изменения           |
| Stream API                  | Фильтрация, преобразование, агрегация данных            |

---

## Типичные ошибки

1. **Модификация `List.of()` / `Map.of()`** — неизменяемые коллекции бросают
   `UnsupportedOperationException` при попытке изменения. Оборачивайте в `new ArrayList<>(List.of(...))`.

2. **Отсутствие `hashCode()`/`equals()` у ключей HashMap** — объекты не находятся по ключу.
   Всегда переопределяйте оба метода (или используйте `record`).

3. **ConcurrentModificationException** — модификация коллекции во время итерации.
   Используйте `Iterator.remove()`, `removeIf()` или копию коллекции.

```java
@Test
void concurrentModificationFix() {
    List<String> tests = new ArrayList<>(List.of("test1", "flaky_test", "test3"));

    // ПЛОХО: ConcurrentModificationException
    // for (String test : tests) {
    //     if (test.startsWith("flaky")) tests.remove(test);
    // }

    // ХОРОШО: removeIf
    tests.removeIf(test -> test.startsWith("flaky"));
    Assertions.assertEquals(2, tests.size());
}
```

4. **`HashMap` с `null`-ключом** — допускает один null-ключ, но `ConcurrentHashMap` — нет.
   `TreeMap` тоже не допускает null-ключи.

5. **Использование обычных коллекций в параллельных тестах** — race conditions,
   потеря данных. Используйте Concurrent-аналоги.

6. **Неправильный выбор коллекции** — `LinkedList` для произвольного доступа по индексу,
   `ArrayList` для частых вставок в начало.

7. **Путаница `remove(int index)` и `remove(Object o)` в `List<Integer>`** —
   `list.remove(1)` удаляет по индексу, а `list.remove(Integer.valueOf(1))` — по значению.

```java
@Test
void removeAmbiguity() {
    List<Integer> numbers = new ArrayList<>(List.of(10, 20, 30));

    numbers.remove(1);           // Удаляет элемент по индексу 1 -> [10, 30]
    Assertions.assertEquals(List.of(10, 30), numbers);

    numbers = new ArrayList<>(List.of(10, 20, 30));
    numbers.remove(Integer.valueOf(20)); // Удаляет объект 20 -> [10, 30]
    Assertions.assertEquals(List.of(10, 30), numbers);
}
```

---

## Вопросы на интервью

### 🟢 Базовый уровень

1. Чем `List` отличается от `Set`?
2. Чем `ArrayList` отличается от `LinkedList`? Когда что использовать?
3. Что такое `Map`? Наследуется ли `Map` от `Collection`?
4. Как создать неизменяемый список в Java?
5. Чем `HashSet` отличается от `TreeSet`?
6. Что произойдёт при добавлении дубликата в `Set`?
7. Как перебрать элементы `Map`?

### 🟡 Средний уровень

8. Как устроен `HashMap` внутри? Что такое бакеты?
9. Что произойдёт при коллизии в `HashMap`?
10. Что такое rehashing? Когда он происходит?
11. Почему важно переопределять `hashCode()` и `equals()` для ключей `HashMap`?
12. Чем `LinkedHashMap` отличается от `HashMap`?
13. Что такое `ConcurrentModificationException` и как его избежать?
14. Чем `List.of()` отличается от `Arrays.asList()`?

### 🔴 Продвинутый уровень

15. Как `HashMap` превращает связный список в красно-чёрное дерево при коллизиях (Java 8+)?
    При каком пороге это происходит?
16. Чем `ConcurrentHashMap` отличается от `Collections.synchronizedMap()`?
    Какой механизм блокировки используется?
17. Объясните контракт `hashCode()` и `equals()`. Что произойдёт, если нарушить этот контракт
    при использовании объекта как ключа `HashMap`?
18. Как работает `CopyOnWriteArrayList`? В каких сценариях он эффективен?
19. Почему `TreeMap` не допускает null-ключи, а `HashMap` допускает?
20. Спроектируйте потокобезопасную систему сбора результатов параллельных тестов
    с использованием concurrent-коллекций.

---

## Практические задания

### Задание 1: Анализатор результатов тестов
Создайте класс `TestResultAnalyzer`, который принимает `List<TestResult>` и предоставляет методы:
- `getPassedTests()` — список пройденных тестов
- `getFailedTests()` — список упавших тестов
- `getPassRate()` — процент прохождения
- `getSlowestTests(int n)` — N самых медленных тестов
- `getTestsByCategory()` — `Map<String, List<TestResult>>` — группировка по категории

### Задание 2: Кэш тестовых данных
Реализуйте `TestDataCache` на основе `LinkedHashMap` с ограничением по размеру (LRU-кэш):
- При превышении лимита удаляется самый давно использованный элемент
- Методы: `put(key, value)`, `get(key)`, `size()`, `clear()`
- Напишите тесты, подтверждающие LRU-поведение

### Задание 3: Матрица совместимости
Создайте `CompatibilityMatrix` — структуру, хранящую результаты тестов для комбинаций
браузер + ОС. Используйте вложенные `Map<String, Map<String, Boolean>>`.
Методы: `addResult(browser, os, passed)`, `getFailedCombinations()`, `generateReport()`.

### Задание 4: Параллельный сборщик результатов
Реализуйте `ParallelTestCollector` с использованием `ConcurrentHashMap` и `CopyOnWriteArrayList`.
Напишите тест, который запускает 10 потоков, каждый из которых добавляет результаты,
и убедитесь, что ни один результат не потерян.

---

## Дополнительные ресурсы

- [Oracle: Collections Framework Overview](https://docs.oracle.com/javase/17/docs/api/java.base/java/util/doc-files/coll-overview.html)
- [Baeldung: Java Collections](https://www.baeldung.com/java-collections)
- [Baeldung: Guide to ConcurrentHashMap](https://www.baeldung.com/java-concurrent-map)
- [Effective Java, Joshua Bloch — Chapter 5: Generics, Chapter 9: General Programming](https://www.oreilly.com/library/view/effective-java/9780134686097/)
- [Java Concurrency in Practice, Brian Goetz](https://jcip.net/)
- [Baeldung: Java HashMap Under the Hood](https://www.baeldung.com/java-hashmap-advanced)
