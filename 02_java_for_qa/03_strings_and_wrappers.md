# Строки и обёртки

## Обзор

Строки — один из наиболее часто используемых типов данных в автоматизации тестирования. Формирование URL,
сравнение текстов на странице, парсинг ответов API, валидация сообщений об ошибках, генерация тестовых данных —
всё это работа со строками. Понимание внутреннего устройства `String`, разницы между `String`, `StringBuilder`
и `StringBuffer`, а также тонкостей wrapper-классов (обёрток) позволяет писать корректные assertions,
избегать коварных багов при сравнении объектов и эффективно работать с тестовыми данными.

---

## String: неизменяемость и String Pool

### Immutability (Неизменяемость)

Объекты `String` в Java являются неизменяемыми (immutable). Каждая «модификация» создаёт новый объект.

```java
@Test
void stringImmutability() {
    String url = "https://api.example.com";
    String modified = url.concat("/users");

    // Оригинальная строка не изменилась!
    Assertions.assertEquals("https://api.example.com", url);
    Assertions.assertEquals("https://api.example.com/users", modified);

    // replace тоже создаёт новый объект
    String upper = url.toUpperCase();
    Assertions.assertNotSame(url, upper); // Разные объекты в памяти
}
```

**Почему String неизменяем:**
- Безопасность — строки используются как ключи в HashMap, параметры подключения к БД
- Потокобезопасность — можно безопасно передавать между потоками
- Кэширование hashCode — вычисляется один раз и кэшируется
- String Pool — позволяет переиспользовать строки

### String Pool

String Pool — область памяти в heap, где Java хранит уникальные строковые литералы.

```java
@Test
void stringPoolBehavior() {
    // Оба литерала указывают на один объект в String Pool
    String s1 = "test";
    String s2 = "test";
    Assertions.assertSame(s1, s2); // true — один объект

    // new String() всегда создаёт новый объект в heap
    String s3 = new String("test");
    Assertions.assertNotSame(s1, s3); // false — разные объекты

    // intern() возвращает ссылку из Pool
    String s4 = s3.intern();
    Assertions.assertSame(s1, s4); // true — объект из Pool

    // Сравнение содержимого — ВСЕГДА через equals()
    Assertions.assertEquals(s1, s3); // true — содержимое одинаковое
}
```

### Правило для QA: всегда используйте `equals()` для сравнения строк

```java
@Test
void correctStringComparison() {
    String expectedMessage = "Order created successfully";
    String actualMessage = getSuccessMessage(); // Может вернуть new String(...)

    // НЕПРАВИЛЬНО: == сравнивает ссылки, а не содержимое
    // if (actualMessage == expectedMessage) { ... }

    // ПРАВИЛЬНО: equals() сравнивает содержимое
    Assertions.assertEquals(expectedMessage, actualMessage);

    // Безопасное сравнение, если actual может быть null
    Assertions.assertTrue(expectedMessage.equals(actualMessage));
    // Или ещё безопаснее — Objects.equals()
    Assertions.assertTrue(Objects.equals(expectedMessage, actualMessage));
}
```

---

## String vs StringBuilder vs StringBuffer

| Характеристика    | String             | StringBuilder        | StringBuffer         |
|-------------------|--------------------|----------------------|----------------------|
| Изменяемость      | Immutable          | Mutable              | Mutable              |
| Потокобезопасность| Да (immutable)     | Нет                  | Да (synchronized)    |
| Производительность| Низкая при конкатенации | Высокая          | Средняя (synchronized overhead)|
| Когда использовать| Малое число операций| Один поток, много конкатенаций | Многопоточность |

### Пример: формирование отчёта о тестах

```java
@Test
void stringBuilderForTestReport() {
    List<TestResult> results = runAllTests();

    // ПЛОХО: каждая конкатенация создаёт новый объект String
    String reportBad = "";
    for (TestResult r : results) {
        reportBad += r.getName() + ": " + r.getStatus() + "\n"; // O(n^2)!
    }

    // ХОРОШО: StringBuilder модифицирует один объект
    StringBuilder reportGood = new StringBuilder();
    reportGood.append("=== Отчёт о тестировании ===\n");
    reportGood.append("Дата: ").append(LocalDate.now()).append("\n\n");

    for (TestResult r : results) {
        reportGood.append(String.format("%-30s [%s]%n",
                r.getName(), r.getStatus()));
    }

    reportGood.append("\nИтого: ")
              .append(results.size())
              .append(" тестов");

    String finalReport = reportGood.toString();
    Assertions.assertTrue(finalReport.contains("Отчёт о тестировании"));
}
```

### Пример: построение динамического URL

```java
public class UrlBuilder {
    private final StringBuilder url;

    public UrlBuilder(String baseUrl) {
        this.url = new StringBuilder(baseUrl);
    }

    public UrlBuilder addPath(String path) {
        if (!path.startsWith("/")) {
            url.append("/");
        }
        url.append(path);
        return this; // Fluent API
    }

    public UrlBuilder addQueryParam(String key, String value) {
        // Первый параметр — ?, остальные — &
        char separator = url.toString().contains("?") ? '&' : '?';
        url.append(separator).append(key).append("=").append(value);
        return this;
    }

    public String build() {
        return url.toString();
    }
}

@Test
void testUrlBuilder() {
    String url = new UrlBuilder("https://api.example.com")
            .addPath("users")
            .addQueryParam("page", "1")
            .addQueryParam("limit", "50")
            .build();

    Assertions.assertEquals(
            "https://api.example.com/users?page=1&limit=50", url);
}
```

---

## Полезные методы String для QA

### Проверка содержимого

```java
@Test
void stringCheckMethods() {
    String response = "  {\"status\": \"success\", \"code\": 200}  ";

    // Удаление пробелов по краям — часто нужно при парсинге
    String trimmed = response.trim();
    // Java 11+: strip() лучше обрабатывает Unicode-пробелы
    String stripped = response.strip();

    Assertions.assertTrue(trimmed.contains("success"));
    Assertions.assertTrue(trimmed.startsWith("{"));
    Assertions.assertTrue(trimmed.endsWith("}"));
    Assertions.assertFalse(trimmed.isEmpty());
    Assertions.assertFalse(trimmed.isBlank()); // Java 11+: проверяет и пробелы
}
```

### Поиск и замена

```java
@Test
void searchAndReplace() {
    String logEntry = "2024-01-15 ERROR [UserService] User not found: id=12345";

    // Поиск подстроки
    int errorIndex = logEntry.indexOf("ERROR");
    Assertions.assertTrue(errorIndex >= 0);

    // Извлечение подстроки
    String timestamp = logEntry.substring(0, 10);
    Assertions.assertEquals("2024-01-15", timestamp);

    // Замена — полезна для очистки тестовых данных
    String sanitized = logEntry.replace("12345", "***");
    Assertions.assertTrue(sanitized.contains("id=***"));

    // Замена по регулярному выражению
    String noTimestamp = logEntry.replaceAll("\\d{4}-\\d{2}-\\d{2}", "DATE");
    Assertions.assertTrue(noTimestamp.startsWith("DATE"));
}
```

### split() — разбиение строки

```java
@Test
void splitForParsing() {
    // Парсинг CSV-данных для тестов
    String csvLine = "John,Doe,john@test.com,admin";
    String[] fields = csvLine.split(",");

    Assertions.assertEquals(4, fields.length);
    Assertions.assertEquals("John", fields[0]);
    Assertions.assertEquals("admin", fields[3]);

    // Парсинг лога
    String log = "2024-01-15 10:30:00 | ERROR | Connection timeout";
    String[] parts = log.split("\\|"); // Экранируем спецсимвол
    Assertions.assertEquals(3, parts.length);
    Assertions.assertEquals(" ERROR ", parts[1]);

    // Ограничение количества частей
    String path = "/api/v1/users/123/orders";
    String[] segments = path.split("/", 4); // Максимум 4 части
    // ["", "api", "v1", "users/123/orders"]
}
```

### format() и formatted() — форматирование

```java
@Test
void stringFormatting() {
    String testUser = "admin";
    int statusCode = 404;
    double responseTime = 1.567;

    // String.format() — классический способ
    String message = String.format(
            "Пользователь '%s' получил код %d за %.2f сек",
            testUser, statusCode, responseTime);
    Assertions.assertEquals(
            "Пользователь 'admin' получил код 404 за 1.57 сек", message);

    // formatted() — Java 15+ (метод экземпляра)
    String message2 = "Endpoint /users вернул %d".formatted(statusCode);
    Assertions.assertEquals("Endpoint /users вернул 404", message2);

    // Полезные спецификаторы для тестовых логов
    String.format("%-20s", "test_name");      // Выравнивание влево
    String.format("%010d", 42);                // Ведущие нули: 0000000042
    String.format("%,d", 1_000_000);           // Разделитель тысяч: 1,000,000
}
```

### matches() — проверка по регулярному выражению

```java
@Test
void regexMatching() {
    // Валидация формата email в тестовых данных
    String email = "user@example.com";
    Assertions.assertTrue(
            email.matches("[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}"));

    // Проверка формата UUID
    String uuid = "550e8400-e29b-41d4-a716-446655440000";
    Assertions.assertTrue(
            uuid.matches("[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}"));

    // Проверка формата даты
    String date = "2024-01-15";
    Assertions.assertTrue(date.matches("\\d{4}-\\d{2}-\\d{2}"));

    // Проверка, что строка содержит только цифры
    String numeric = "12345";
    Assertions.assertTrue(numeric.matches("\\d+"));
}
```

### Текстовые блоки (Java 13+)

```java
@Test
void textBlocks() {
    // Удобны для JSON в тестах
    String expectedJson = """
            {
                "name": "John",
                "email": "john@test.com",
                "role": "admin"
            }
            """;

    // Удобны для SQL-запросов в тестах
    String query = """
            SELECT u.name, u.email
            FROM users u
            WHERE u.role = 'admin'
              AND u.active = true
            ORDER BY u.name
            """;

    Assertions.assertTrue(expectedJson.contains("\"name\": \"John\""));
}
```

---

## Wrapper-классы (Обёртки)

### Основные обёртки и их методы

```java
@Test
void wrapperClassMethods() {
    // Парсинг строк в числа — частая задача при работе с тестовыми данными
    int statusCode = Integer.parseInt("200");
    long orderId = Long.parseLong("9876543210");
    double price = Double.parseDouble("49.99");
    boolean flag = Boolean.parseBoolean("true");

    Assertions.assertEquals(200, statusCode);

    // valueOf() возвращает обёртку (использует кэш для Integer -128..127)
    Integer cached1 = Integer.valueOf(100);
    Integer cached2 = Integer.valueOf(100);
    Assertions.assertSame(cached1, cached2); // Один объект из кэша

    // Конвертация в другие системы счисления
    String binary = Integer.toBinaryString(255);   // "11111111"
    String hex = Integer.toHexString(255);          // "ff"
    String octal = Integer.toOctalString(255);      // "377"

    Assertions.assertEquals("ff", hex);

    // Полезные константы
    Assertions.assertEquals(2147483647, Integer.MAX_VALUE);
    Assertions.assertEquals(-2147483648, Integer.MIN_VALUE);
}
```

### Autoboxing: ловушки и подводные камни

```java
@Test
void autoboxingPitfalls() {
    // Ловушка 1: Integer cache — работает только для -128..127
    Integer a = 127;
    Integer b = 127;
    Assertions.assertTrue(a == b);  // true — из кэша

    Integer c = 128;
    Integer d = 128;
    Assertions.assertFalse(c == d); // false — разные объекты!
    Assertions.assertEquals(c, d);  // true — по значению

    // Ловушка 2: NullPointerException при unboxing
    Integer nullValue = null;
    Assertions.assertThrows(NullPointerException.class, () -> {
        int result = nullValue; // Unboxing null -> NPE!
    });

    // Ловушка 3: Производительность в цикле
    // ПЛОХО: на каждой итерации создаётся новый объект Long
    Long sumBad = 0L;
    for (int i = 0; i < 1000; i++) {
        sumBad += i; // Unboxing + операция + autoboxing
    }

    // ХОРОШО: примитив
    long sumGood = 0L;
    for (int i = 0; i < 1000; i++) {
        sumGood += i; // Чистая арифметика, без объектов
    }
}
```

### Integer Cache подробнее

```java
@Test
void integerCacheDeepDive() {
    // Java кэширует Integer объекты в диапазоне -128..127
    // Это касается autoboxing и Integer.valueOf()

    // Все эти пары ссылаются на одни и те же объекты
    Assertions.assertSame(Integer.valueOf(0), Integer.valueOf(0));
    Assertions.assertSame(Integer.valueOf(-128), Integer.valueOf(-128));
    Assertions.assertSame(Integer.valueOf(127), Integer.valueOf(127));

    // А эти — на разные
    Assertions.assertNotSame(Integer.valueOf(128), Integer.valueOf(128));
    Assertions.assertNotSame(Integer.valueOf(-129), Integer.valueOf(-129));

    // Для Boolean кэшируются оба значения
    Boolean t1 = Boolean.valueOf(true);
    Boolean t2 = Boolean.valueOf(true);
    Assertions.assertSame(t1, t2);

    // Для Byte кэшируется весь диапазон (-128..127)
    Byte b1 = Byte.valueOf((byte) 100);
    Byte b2 = Byte.valueOf((byte) 100);
    Assertions.assertSame(b1, b2);
}
```

---

## Практика: строки в тестовой автоматизации

### Генерация уникальных тестовых данных

```java
public class TestDataGenerator {

    /**
     * Генерирует уникальный email для тестов.
     * Использует timestamp для уникальности.
     */
    public static String generateUniqueEmail(String prefix) {
        return String.format("%s_%d@test.com",
                prefix, System.currentTimeMillis());
    }

    /**
     * Генерирует случайную строку заданной длины.
     * Полезна для тестирования граничных значений полей ввода.
     */
    public static String generateRandomString(int length) {
        var sb = new StringBuilder(length);
        var chars = "abcdefghijklmnopqrstuvwxyz0123456789";
        var random = new java.util.Random();

        for (int i = 0; i < length; i++) {
            sb.append(chars.charAt(random.nextInt(chars.length())));
        }
        return sb.toString();
    }

    /**
     * Маскирует конфиденциальные данные в логах.
     */
    public static String maskSensitiveData(String input) {
        // Маскируем email
        String masked = input.replaceAll(
                "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+",
                "***@***");
        // Маскируем номера карт (16 цифр)
        masked = masked.replaceAll("\\d{16}", "****-****-****-****");
        return masked;
    }
}

@Test
void testDataGeneration() {
    String email = TestDataGenerator.generateUniqueEmail("user");
    Assertions.assertTrue(email.matches("user_\\d+@test\\.com"));

    String randomStr = TestDataGenerator.generateRandomString(10);
    Assertions.assertEquals(10, randomStr.length());

    String masked = TestDataGenerator.maskSensitiveData(
            "User john@mail.com paid with 1234567890123456");
    Assertions.assertFalse(masked.contains("john@mail.com"));
    Assertions.assertFalse(masked.contains("1234567890123456"));
}
```

### Парсинг и валидация ответов API

```java
@Test
void parseApiResponse() {
    String jsonResponse = """
            {"id": 42, "status": "active", "name": "Test User"}
            """;

    // Простой парсинг без библиотек (для демонстрации работы со строками)
    Assertions.assertTrue(jsonResponse.contains("\"status\": \"active\""));

    // Извлечение значения по ключу (упрощённый вариант)
    String statusValue = extractJsonValue(jsonResponse, "status");
    Assertions.assertEquals("active", statusValue);
}

// Вспомогательный метод для извлечения значения из JSON
private String extractJsonValue(String json, String key) {
    String searchKey = "\"" + key + "\": \"";
    int startIndex = json.indexOf(searchKey);
    if (startIndex == -1) return null;

    startIndex += searchKey.length();
    int endIndex = json.indexOf("\"", startIndex);
    return json.substring(startIndex, endIndex);
}
```

---

## Связь с тестированием

| Концепция               | Применение в QA                                             |
|-------------------------|-------------------------------------------------------------|
| String immutability     | Безопасная передача expected values между методами           |
| String Pool             | Понимание поведения `==` vs `equals()` в assertions         |
| StringBuilder           | Формирование отчётов, логов, динамических запросов          |
| `split()`               | Парсинг CSV, логов, конфигурационных файлов                 |
| `matches()` / regex     | Валидация формата данных (email, UUID, дата)                |
| `format()` / `formatted()` | Читаемые assertion messages, формирование тестовых данных|
| Text blocks             | Удобное хранение JSON/XML/SQL в тестах                      |
| Wrapper classes         | Nullable тестовые данные, работа с коллекциями              |
| Integer cache           | Правильное сравнение в assertions                           |

---

## Типичные ошибки

1. **Сравнение строк через `==`** — самая частая ошибка. Используйте `equals()` или `assertEquals()`.

2. **Конкатенация в цикле** — `str += ...` в цикле создаёт O(n^2) объектов. Используйте `StringBuilder`.

3. **Забытый `trim()`/`strip()`** — пробелы в начале/конце строки ломают сравнение.
   `"success " != "success"`.

4. **NullPointerException при вызове методов на null** — `nullString.equals("test")` упадёт.
   Безопаснее: `"test".equals(nullString)` или `Objects.equals()`.

5. **Неэкранированные спецсимволы в regex** — `split(".")` разделит по любому символу, а не по точке.
   Правильно: `split("\\.")`.

6. **Сравнение обёрток через `==`** — для `Integer` за пределами -128..127 всегда `false`.
   Используйте `equals()`.

7. **Unboxing `null`** — `Integer x = null; int y = x;` бросит `NullPointerException`.

8. **Использование `new String("...")`** — создаёт лишний объект. Используйте литералы.

---

## Вопросы на интервью

### 🟢 Базовый уровень

1. Почему `String` в Java неизменяем? Назовите минимум три причины.
2. Чем отличается `==` от `equals()` для строк?
3. Что такое String Pool?
4. Назовите 5 часто используемых методов класса `String`.
5. Чем `StringBuilder` отличается от `StringBuffer`?
6. Что такое autoboxing и unboxing?

### 🟡 Средний уровень

7. Объясните Integer cache. Какой диапазон кэшируется?
8. Почему конкатенация строк в цикле — плохая практика?
9. Как работает метод `intern()` у `String`?
10. Что будет при выполнении `Integer a = null; int b = a;`?
11. Чем `strip()` (Java 11) отличается от `trim()`?
12. Как правильно сравнивать строки, если одна из них может быть `null`?

### 🔴 Продвинутый уровень

13. Как String реализован внутри? Что изменилось в Java 9 (Compact Strings)?
14. Почему `String` является хорошим ключом для `HashMap`? Как связаны immutability и hashCode?
15. Объясните, как JVM оптимизирует конкатенацию строк через `invokedynamic` (Java 9+).
16. Можно ли расширить диапазон Integer cache? Как и зачем?
17. Чем `String.format()` отличается от `MessageFormat.format()`? Когда что использовать?

---

## Практические задания

### Задание 1: Валидатор тестовых данных
Напишите класс `TestDataValidator` с методами:
- `isValidEmail(String)` — проверка формата email
- `isValidPhone(String)` — проверка формата телефона (+7XXXXXXXXXX)
- `isValidDate(String)` — проверка формата даты (YYYY-MM-DD)
- `isStrongPassword(String)` — минимум 8 символов, буквы верхнего и нижнего регистра, цифра, спецсимвол

Напишите JUnit-тесты для каждого метода с позитивными и негативными кейсами.

### Задание 2: Генератор тестовых отчётов
Используя `StringBuilder`, создайте класс `TestReportBuilder` с fluent API:

```java
String report = new TestReportBuilder()
    .title("Smoke Test Results")
    .date(LocalDate.now())
    .addResult("Login Test", "PASSED", "1.2s")
    .addResult("Search Test", "FAILED", "3.5s")
    .summary()
    .build();
```

### Задание 3: Маскировщик данных
Напишите утилиту `DataMasker`, которая заменяет в строке:
- Email: `user@mail.com` -> `u***@m***.com`
- Телефон: `+71234567890` -> `+7***890`
- Номер карты: `1234567890123456` -> `1234****3456`

Покройте тестами, включая edge cases (строка без данных, несколько вхождений, null).

### Задание 4: Парсер логов
Напишите `LogParser`, который из строки лога вида:
```
2024-01-15 10:30:45 [ERROR] UserService - User not found: id=123
```
извлекает: дату, время, уровень, сервис, сообщение. Верните результат в виде record.
Напишите тесты для разных уровней логирования и edge cases.

---

## Дополнительные ресурсы

- [Oracle: String Class Documentation](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/String.html)
- [Baeldung: Java String](https://www.baeldung.com/java-string)
- [Baeldung: Java StringBuilder vs StringBuffer](https://www.baeldung.com/java-string-builder-string-buffer)
- [JEP 254: Compact Strings](https://openjdk.org/jeps/254)
- [Effective Java, Joshua Bloch — Item 63: Beware the performance of string concatenation](https://www.oreilly.com/library/view/effective-java/9780134686097/)
- [Baeldung: Guide to Java Regular Expressions](https://www.baeldung.com/regular-expressions-java)
