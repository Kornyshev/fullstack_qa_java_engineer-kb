# Основы Java для QA

## Обзор

Java — один из наиболее востребованных языков в автоматизации тестирования. Фреймворки Selenium, RestAssured,
Appium, JUnit, TestNG — все построены на Java. Понимание базовых конструкций языка критически важно для написания
надёжных, читаемых и поддерживаемых автотестов. В этом разделе рассматриваются примитивные типы, обёртки,
операторы, условия, циклы и массивы — фундамент, на котором строится любой тестовый фреймворк.

---

## Примитивные типы и обёртки

Java имеет 8 примитивных типов. Каждый из них имеет соответствующий класс-обёртку (wrapper).

| Примитив  | Размер  | Диапазон                                    | Обёртка     | Значение по умолчанию |
|-----------|---------|---------------------------------------------|-------------|-----------------------|
| `byte`    | 8 бит   | -128 ... 127                                | `Byte`      | `0`                   |
| `short`   | 16 бит  | -32 768 ... 32 767                          | `Short`     | `0`                   |
| `int`     | 32 бит  | -2^31 ... 2^31 - 1                          | `Integer`   | `0`                   |
| `long`    | 64 бит  | -2^63 ... 2^63 - 1                          | `Long`      | `0L`                  |
| `float`   | 32 бит  | ~±3.4E+38 (6-7 значащих цифр)              | `Float`     | `0.0f`                |
| `double`  | 64 бит  | ~±1.7E+308 (15 значащих цифр)              | `Double`    | `0.0d`                |
| `char`    | 16 бит  | 0 ... 65 535 (Unicode)                      | `Character` | `'\u0000'`            |
| `boolean` | ~1 бит  | `true` / `false`                            | `Boolean`   | `false`               |

### Пример: выбор типа для тестовых данных

```java
public class TestDataTypes {

    // Код ответа HTTP — достаточно int
    int statusCode = 200;

    // Идентификатор заказа может быть очень большим
    long orderId = 9_223_372_036_854L;

    // Цена товара — используем double (но не для финансовых расчётов!)
    double price = 49.99;

    // Флаг: тест пройден или нет
    boolean isPassed = true;

    // Символ валюты
    char currencySymbol = '$';
}
```

### Autoboxing и Unboxing

**Autoboxing** — автоматическое преобразование примитива в обёртку.
**Unboxing** — обратное преобразование обёртки в примитив.

```java
@Test
void autoboxingExample() {
    // Autoboxing: int -> Integer
    Integer count = 5;

    // Unboxing: Integer -> int
    int value = count;

    // Опасность: NullPointerException при unboxing null
    Integer nullableCount = null;
    // int result = nullableCount; // NullPointerException!

    // Безопасная проверка перед unboxing
    if (nullableCount != null) {
        int safeResult = nullableCount;
    }
}
```

### Ловушка сравнения обёрток

```java
@Test
void wrapperComparisonTrap() {
    Integer a = 127;
    Integer b = 127;
    // true — значения из Integer cache (-128..127)
    Assertions.assertTrue(a == b);

    Integer c = 128;
    Integer d = 128;
    // false — разные объекты в heap!
    Assertions.assertFalse(c == d);

    // Правильное сравнение — всегда через equals()
    Assertions.assertEquals(c, d);
}
```

---

## Операторы

### Арифметические операторы

| Оператор | Описание       | Пример     |
|----------|----------------|------------|
| `+`      | Сложение       | `5 + 3 = 8`|
| `-`      | Вычитание      | `5 - 3 = 2`|
| `*`      | Умножение      | `5 * 3 = 15`|
| `/`      | Деление        | `7 / 2 = 3` (целочисленное)|
| `%`      | Остаток        | `7 % 2 = 1`|

### Операторы сравнения

```java
@Test
void comparisonOperatorsInAssertions() {
    int expected = 200;
    int actual = getStatusCode();

    // Все эти проверки могут встретиться в тестах
    Assertions.assertTrue(actual == expected);       // равно
    Assertions.assertTrue(actual != 404);            // не равно
    Assertions.assertTrue(actual >= 200);            // больше или равно
    Assertions.assertTrue(actual < 300);             // меньше
}
```

### Логические операторы

| Оператор | Описание          | Short-circuit |
|----------|-------------------|---------------|
| `&&`     | Логическое И      | Да            |
| `\|\|`  | Логическое ИЛИ    | Да            |
| `!`      | Логическое НЕ     | —             |
| `&`      | Побитовое И       | Нет           |
| `\|`     | Побитовое ИЛИ     | Нет           |

```java
@Test
void logicalOperatorsInTestConditions() {
    int statusCode = 200;
    String body = "{\"success\": true}";

    // Short-circuit: если statusCode != 200, body.contains не вызовется
    boolean isSuccessful = statusCode == 200 && body.contains("success");
    Assertions.assertTrue(isSuccessful);

    // Проверка: код ответа в диапазоне успешных
    boolean isOk = statusCode >= 200 && statusCode < 300;
    Assertions.assertTrue(isOk);
}
```

### Тернарный оператор

```java
@Test
void ternaryOperatorForTestData() {
    String env = System.getProperty("env", "dev");

    // Выбор base URL в зависимости от окружения
    String baseUrl = env.equals("prod")
            ? "https://api.example.com"
            : "https://dev-api.example.com";

    Assertions.assertTrue(baseUrl.startsWith("https://"));
}
```

---

## Условные конструкции

### if / else if / else

```java
@Test
void validateStatusCode() {
    int statusCode = 201;

    // Классическая проверка HTTP-статуса в тестах
    if (statusCode >= 200 && statusCode < 300) {
        System.out.println("Успешный ответ: " + statusCode);
    } else if (statusCode >= 400 && statusCode < 500) {
        Assertions.fail("Клиентская ошибка: " + statusCode);
    } else if (statusCode >= 500) {
        Assertions.fail("Серверная ошибка: " + statusCode);
    } else {
        Assertions.fail("Неожиданный статус: " + statusCode);
    }
}
```

### switch (классический и улучшенный в Java 17+)

```java
// Классический switch
String getEnvironmentUrl(String env) {
    switch (env) {
        case "dev":
            return "https://dev.example.com";
        case "staging":
            return "https://staging.example.com";
        case "prod":
            return "https://example.com";
        default:
            throw new IllegalArgumentException("Неизвестное окружение: " + env);
    }
}

// Switch expressions (Java 14+) — компактнее и безопаснее
String getEnvironmentUrlModern(String env) {
    return switch (env) {
        case "dev" -> "https://dev.example.com";
        case "staging" -> "https://staging.example.com";
        case "prod" -> "https://example.com";
        default -> throw new IllegalArgumentException("Неизвестное окружение: " + env);
    };
}
```

### Pattern Matching для instanceof (Java 17+)

```java
@Test
void patternMatchingInAssertions() {
    Object response = getApiResponse();

    // До Java 16: нужен явный каст
    if (response instanceof String) {
        String body = (String) response;
        Assertions.assertTrue(body.contains("success"));
    }

    // Java 16+: переменная сразу доступна
    if (response instanceof String body) {
        Assertions.assertTrue(body.contains("success"));
    }
}
```

---

## Циклы

### for — классический

```java
@Test
void classicForLoop() {
    // Проверяем несколько endpoint'ов
    String[] endpoints = {"/users", "/orders", "/products"};

    for (int i = 0; i < endpoints.length; i++) {
        int statusCode = sendGetRequest(endpoints[i]);
        Assertions.assertEquals(200, statusCode,
                "Endpoint " + endpoints[i] + " вернул неожиданный статус");
    }
}
```

### for-each (enhanced for)

```java
@Test
void forEachLoop() {
    List<String> usernames = List.of("admin", "user1", "guest");

    // Перебираем тестовые данные
    for (String username : usernames) {
        boolean exists = checkUserExists(username);
        Assertions.assertTrue(exists, "Пользователь " + username + " не найден");
    }
}
```

### while

```java
@Test
void whileLoopForPolling() {
    // Polling-паттерн: ждём, пока задача завершится (с timeout)
    int maxAttempts = 10;
    int attempt = 0;
    boolean taskCompleted = false;

    while (!taskCompleted && attempt < maxAttempts) {
        taskCompleted = checkTaskStatus("task-123");
        attempt++;

        if (!taskCompleted) {
            try {
                Thread.sleep(1000); // Ждём 1 секунду между попытками
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }

    Assertions.assertTrue(taskCompleted,
            "Задача не завершилась за " + maxAttempts + " попыток");
}
```

### do-while

```java
@Test
void doWhileForRetry() {
    int maxRetries = 3;
    int attempt = 0;
    boolean success;

    // Выполняем запрос хотя бы 1 раз, повторяем при неудаче
    do {
        attempt++;
        success = sendRequestWithPossibleFlakiness();
    } while (!success && attempt < maxRetries);

    Assertions.assertTrue(success,
            "Запрос не удался после " + maxRetries + " попыток");
}
```

### Управление циклом: break и continue

```java
@Test
void breakAndContinueExample() {
    List<String> logs = getApplicationLogs();

    // Ищем первую ошибку в логах
    String firstError = null;
    for (String log : logs) {
        if (log.startsWith("DEBUG")) {
            continue; // Пропускаем debug-записи
        }
        if (log.contains("ERROR")) {
            firstError = log;
            break; // Нашли — выходим из цикла
        }
    }

    Assertions.assertNotNull(firstError, "Ожидалась ошибка в логах");
}
```

---

## Массивы

### Объявление и инициализация

```java
@Test
void arrayBasics() {
    // Статическая инициализация
    String[] browsers = {"chrome", "firefox", "edge"};

    // Динамическая инициализация
    int[] responseTimes = new int[100];

    // Двумерный массив — матрица тестовых данных
    String[][] testData = {
            {"admin", "admin123", "true"},
            {"user", "wrong_pass", "false"},
            {"", "", "false"}
    };

    Assertions.assertEquals(3, browsers.length);
    Assertions.assertEquals(3, testData.length);
    Assertions.assertEquals("admin", testData[0][0]);
}
```

### Работа с массивами через Arrays

```java
import java.util.Arrays;

@Test
void arraysUtilityMethods() {
    int[] responseTimes = {250, 100, 320, 80, 190};

    // Сортировка для нахождения медианы / перцентилей
    Arrays.sort(responseTimes);
    // Теперь: [80, 100, 190, 250, 320]

    // Бинарный поиск в отсортированном массиве
    int index = Arrays.binarySearch(responseTimes, 190);
    Assertions.assertTrue(index >= 0);

    // Копирование первых N элементов
    int[] fastest = Arrays.copyOf(responseTimes, 3);
    Assertions.assertArrayEquals(new int[]{80, 100, 190}, fastest);

    // Преобразование в строку (для логирования в тестах)
    String formatted = Arrays.toString(responseTimes);
    Assertions.assertEquals("[80, 100, 190, 250, 320]", formatted);

    // Заполнение массива значением по умолчанию
    int[] defaults = new int[5];
    Arrays.fill(defaults, -1);
    Assertions.assertArrayEquals(new int[]{-1, -1, -1, -1, -1}, defaults);
}
```

### Многомерные массивы для тестовых данных

```java
@Test
void multiDimensionalArrayForTestData() {
    // Таблица тестовых данных: [ввод, ожидаемый результат]
    Object[][] loginTestData = {
            {"admin@test.com", "Password1!", true},
            {"user@test.com", "short", false},
            {"invalid-email", "Password1!", false},
            {"", "", false}
    };

    for (Object[] testCase : loginTestData) {
        String email = (String) testCase[0];
        String password = (String) testCase[1];
        boolean expectedResult = (boolean) testCase[2];

        boolean actualResult = attemptLogin(email, password);
        Assertions.assertEquals(expectedResult, actualResult,
                String.format("Логин с email='%s', password='%s'", email, password));
    }
}
```

---

## Связь с тестированием

| Концепция             | Применение в QA                                              |
|-----------------------|--------------------------------------------------------------|
| Примитивные типы      | Хранение status code, timeout, retry count                   |
| Обёртки               | Nullable-значения в тестовых данных, коллекции               |
| Условия               | Ветвление логики тестов в зависимости от окружения           |
| switch expression     | Маршрутизация конфигурации, выбор стратегии                  |
| for-each              | Итерация по тестовым данным, проверка элементов списка       |
| while                 | Polling, ожидание асинхронных операций                       |
| Массивы               | Параметризация тестов, хранение тестовых данных              |
| Тернарный оператор    | Компактный выбор значений в конфигурации                     |

---

## Типичные ошибки

1. **Сравнение обёрток через `==`** — работает только для Integer в диапазоне -128..127 (Integer cache).
   Всегда используйте `equals()`.

2. **Целочисленное деление** — `7 / 2` даёт `3`, а не `3.5`. Для дробного результата нужен каст:
   `(double) 7 / 2`.

3. **NullPointerException при unboxing** — `Integer x = null; int y = x;` бросит NPE.
   Всегда проверяйте на null.

4. **ArrayIndexOutOfBoundsException** — обращение к элементу за пределами массива.
   Проверяйте `array.length` перед доступом по индексу.

5. **Забытый `break` в `switch`** — без `break` выполнение «проваливается» в следующий `case` (fall-through).
   Используйте switch expressions для защиты от этой ошибки.

6. **Бесконечный цикл** — забыли инкрементировать счётчик или изменить условие выхода.
   В тестах всегда добавляйте timeout или ограничение по попыткам.

7. **Путаница `=` и `==`** — `=` присваивание, `==` сравнение. В условии `if (x = 5)` Java не скомпилирует
   (кроме boolean), но лучше быть внимательным.

8. **Потеря точности float/double** — `0.1 + 0.2 != 0.3`. Для точных финансовых расчётов — `BigDecimal`.

---

## Вопросы на интервью

### 🟢 Базовый уровень

1. Перечислите все примитивные типы Java и их размеры.
2. Чем отличается `int` от `Integer`?
3. Что такое autoboxing и unboxing?
4. Какие виды циклов существуют в Java? Когда какой использовать?
5. Чем `==` отличается от `equals()` для обёрток?
6. Что произойдёт, если забыть `break` в `switch`?
7. Какой размер массива после создания `new int[10]`? Чем он заполнен?

### 🟡 Средний уровень

8. Объясните Integer cache. Почему `Integer.valueOf(127) == Integer.valueOf(127)` возвращает `true`,
   а `Integer.valueOf(128) == Integer.valueOf(128)` — `false`?
9. Чем switch expression (Java 14+) лучше классического switch?
10. Как избежать `ArrayIndexOutOfBoundsException` при работе с тестовыми данными?
11. Приведите пример использования тернарного оператора в тестовой конфигурации.
12. Почему `0.1 + 0.2 != 0.3` в Java? Как это влияет на assertions в тестах?

### 🔴 Продвинутый уровень

13. Как работает pattern matching для `instanceof` в Java 17+? Как это упрощает тестовый код?
14. Объясните разницу между `short-circuit` и `non-short-circuit` логическими операторами.
    Приведите пример, когда это критично.
15. Как устроена память для примитивов и обёрток? Stack vs Heap.
16. Почему использование `float`/`double` для денежных сумм в тестовых данных — плохая идея?
    Какой тип использовать?

---

## Практические задания

### Задание 1: Валидация HTTP-статусов
Напишите метод `categorizeStatusCode(int code)`, возвращающий строку:
- `"informational"` для 1xx
- `"success"` для 2xx
- `"redirection"` для 3xx
- `"client_error"` для 4xx
- `"server_error"` для 5xx
- `"unknown"` для остальных

Напишите тесты для всех граничных значений (100, 199, 200, 299, ...).

### Задание 2: Анализ массива response times
Дан массив `int[] responseTimes`. Напишите методы:
- `findMin(int[])` — минимальное время
- `findMax(int[])` — максимальное время
- `calculateAverage(int[])` — среднее время
- `countAboveThreshold(int[], int threshold)` — количество значений выше порога

Напишите тесты, включая случаи с пустым массивом и одним элементом.

### Задание 3: Retry-механизм
Реализуйте метод `executeWithRetry(Runnable action, int maxRetries)`, который:
- Выполняет `action`
- При исключении повторяет до `maxRetries` раз
- Возвращает `true` при успехе, `false` если все попытки исчерпаны

Напишите тесты с mock-действием, которое падает первые N раз, а потом срабатывает.

### Задание 4: Генератор тестовых данных
Напишите метод, который на основе двумерного массива тестовых данных
генерирует читаемый отчёт в формате строки:

```
Test Case 1: email=admin@test.com, password=***, expected=true
Test Case 2: email=user@test.com, password=***, expected=false
```

---

## Дополнительные ресурсы

- [Oracle Java Tutorial: Language Basics](https://docs.oracle.com/javase/tutorial/java/nutsandbolts/)
- [JLS: Primitive Types](https://docs.oracle.com/javase/specs/jls/se17/html/jls-4.html#jls-4.2)
- [Baeldung: Java Primitives vs Objects](https://www.baeldung.com/java-primitives-vs-objects)
- [Effective Java, Joshua Bloch — Item 61: Prefer primitive types to boxed primitives](https://www.oreilly.com/library/view/effective-java/9780134686097/)
- [JUnit 5 User Guide](https://junit.org/junit5/docs/current/user-guide/)
