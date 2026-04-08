# Stream API для QA

## Обзор

Stream API, появившийся в Java 8, представляет собой мощный инструмент для обработки коллекций данных
в декларативном стиле. Для QA-инженера это незаменимый навык: фильтрация результатов тестов,
обработка тестовых данных, генерация отчётов, работа с коллекциями WebElement — всё это
эффективно решается с помощью Stream API.

Stream — это не структура данных, а последовательность элементов, поддерживающая цепочку операций.
Stream не изменяет исходную коллекцию, а создаёт конвейер обработки данных.

Ключевые характеристики:
- **Ленивость (Laziness)** — промежуточные операции выполняются только при вызове терминальной операции
- **Одноразовость** — stream можно использовать только один раз
- **Возможность параллельной обработки** — `parallelStream()` для многопоточной обработки

---

## Создание Stream

```java
import java.util.stream.*;
import java.util.*;

// 1. Из коллекции
List<String> testCases = List.of("login_test", "logout_test", "payment_test");
Stream<String> streamFromList = testCases.stream();

// 2. Из массива
String[] browsers = {"Chrome", "Firefox", "Safari"};
Stream<String> streamFromArray = Arrays.stream(browsers);

// 3. С помощью Stream.of()
Stream<String> streamOf = Stream.of("PASSED", "FAILED", "SKIPPED");

// 4. С помощью Stream.generate() — бесконечный stream
Stream<String> generated = Stream.generate(() -> "test_data_" + UUID.randomUUID())
        .limit(10); // ограничиваем, иначе бесконечный

// 5. С помощью Stream.iterate()
Stream<Integer> iterated = Stream.iterate(1, n -> n + 1)
        .limit(100); // идентификаторы тестов от 1 до 100

// 6. Из строки (stream символов)
IntStream chars = "TestString".chars();

// 7. Из файла (каждая строка — элемент stream)
// Stream<String> lines = Files.lines(Path.of("test_data.csv"));

// 8. Stream.builder()
Stream<String> built = Stream.<String>builder()
        .add("unit")
        .add("integration")
        .add("e2e")
        .build();
```

---

## Промежуточные операции (Intermediate Operations)

Промежуточные операции возвращают новый Stream и являются ленивыми — они не выполняются
до вызова терминальной операции.

### filter — фильтрация элементов

```java
// Отфильтровать только упавшие тесты
List<TestResult> allResults = getTestResults();

List<TestResult> failedTests = allResults.stream()
        .filter(result -> result.getStatus() == Status.FAILED)
        .collect(Collectors.toList());

// Отфильтровать активные тест-кейсы с приоритетом HIGH
List<TestCase> criticalActive = testCases.stream()
        .filter(tc -> tc.isActive())
        .filter(tc -> tc.getPriority() == Priority.HIGH)
        .collect(Collectors.toList());
```

### map — преобразование элементов

```java
// Получить имена всех тестов
List<String> testNames = allResults.stream()
        .map(TestResult::getTestName)
        .collect(Collectors.toList());

// Преобразовать строки в верхний регистр
List<String> upperCaseNames = testNames.stream()
        .map(String::toUpperCase)
        .collect(Collectors.toList());

// Преобразовать результат теста в строку для отчёта
List<String> reportLines = allResults.stream()
        .map(r -> String.format("[%s] %s - %dms", r.getStatus(), r.getTestName(), r.getDuration()))
        .collect(Collectors.toList());
```

### flatMap — "разворачивание" вложенных структур

```java
// У каждого TestSuite есть список TestCase
List<TestSuite> suites = getTestSuites();

// Получить все тест-кейсы из всех сьютов одним списком
List<TestCase> allTestCases = suites.stream()
        .flatMap(suite -> suite.getTestCases().stream())
        .collect(Collectors.toList());

// Получить все шаги из всех тестов
List<String> allSteps = testCases.stream()
        .flatMap(tc -> tc.getSteps().stream())
        .collect(Collectors.toList());
```

### sorted — сортировка

```java
// Сортировка по имени теста (естественный порядок)
List<TestResult> sortedByName = allResults.stream()
        .sorted(Comparator.comparing(TestResult::getTestName))
        .collect(Collectors.toList());

// Сортировка по длительности (от самого долгого)
List<TestResult> sortedByDuration = allResults.stream()
        .sorted(Comparator.comparing(TestResult::getDuration).reversed())
        .collect(Collectors.toList());

// Сортировка по нескольким критериям: сначала по статусу, затем по имени
List<TestResult> multiSorted = allResults.stream()
        .sorted(Comparator.comparing(TestResult::getStatus)
                .thenComparing(TestResult::getTestName))
        .collect(Collectors.toList());
```

### distinct — удаление дубликатов

```java
// Уникальные статусы из результатов
List<Status> uniqueStatuses = allResults.stream()
        .map(TestResult::getStatus)
        .distinct()
        .collect(Collectors.toList());

// Уникальные браузеры, на которых запускались тесты
List<String> uniqueBrowsers = allResults.stream()
        .map(TestResult::getBrowser)
        .distinct()
        .collect(Collectors.toList());
```

### peek — просмотр элементов без изменения (для отладки)

```java
// peek полезен для логирования промежуточных шагов
List<TestResult> processed = allResults.stream()
        .filter(r -> r.getStatus() == Status.FAILED)
        .peek(r -> System.out.println("Обрабатываем упавший тест: " + r.getTestName()))
        .sorted(Comparator.comparing(TestResult::getDuration).reversed())
        .peek(r -> System.out.println("После сортировки: " + r.getTestName()))
        .collect(Collectors.toList());
```

> **Важно:** `peek` предназначен для отладки. Не стоит использовать его для побочных эффектов
> в production-коде — для этого есть `forEach`.

---

## Терминальные операции (Terminal Operations)

Терминальные операции запускают конвейер и возвращают результат (не Stream).

### collect — сбор результатов

```java
// Собрать в List
List<String> list = stream.collect(Collectors.toList());

// С Java 16+ — более лаконичный вариант
List<String> list16 = stream.toList(); // возвращает неизменяемый список

// Собрать в Set
Set<String> set = stream.collect(Collectors.toSet());
```

### reduce — свёртка элементов

```java
// Суммарное время выполнения всех тестов
long totalDuration = allResults.stream()
        .map(TestResult::getDuration)
        .reduce(0L, Long::sum);

// Конкатенация имён тестов
String allNames = testNames.stream()
        .reduce("", (a, b) -> a + ", " + b);

// Поиск самого долгого теста
Optional<TestResult> longestTest = allResults.stream()
        .reduce((a, b) -> a.getDuration() > b.getDuration() ? a : b);
```

### forEach — выполнение действия для каждого элемента

```java
// Вывести все упавшие тесты
allResults.stream()
        .filter(r -> r.getStatus() == Status.FAILED)
        .forEach(r -> System.out.println("FAILED: " + r.getTestName()));

// Логирование результатов
allResults.stream()
        .forEach(r -> logger.info("Тест {} завершился со статусом {}", r.getTestName(), r.getStatus()));
```

### count — подсчёт элементов

```java
// Количество пройденных тестов
long passedCount = allResults.stream()
        .filter(r -> r.getStatus() == Status.PASSED)
        .count();
```

### findFirst и findAny

```java
// Найти первый упавший тест
Optional<TestResult> firstFailed = allResults.stream()
        .filter(r -> r.getStatus() == Status.FAILED)
        .findFirst();

// findAny — может вернуть любой подходящий элемент (полезно для parallelStream)
Optional<TestResult> anyFailed = allResults.parallelStream()
        .filter(r -> r.getStatus() == Status.FAILED)
        .findAny();
```

### anyMatch, allMatch, noneMatch

```java
// Есть ли хотя бы один упавший тест?
boolean hasFailed = allResults.stream()
        .anyMatch(r -> r.getStatus() == Status.FAILED);

// Все ли тесты прошли?
boolean allPassed = allResults.stream()
        .allMatch(r -> r.getStatus() == Status.PASSED);

// Нет ли тестов без результата?
boolean noSkipped = allResults.stream()
        .noneMatch(r -> r.getStatus() == Status.SKIPPED);
```

---

## Collectors — продвинутые коллекторы

### toMap — сбор в Map

```java
// Map: имя теста -> статус
Map<String, Status> resultMap = allResults.stream()
        .collect(Collectors.toMap(
                TestResult::getTestName,  // ключ
                TestResult::getStatus     // значение
        ));

// При возможных дубликатах ключей — указываем стратегию разрешения конфликтов
Map<String, Status> resultMapSafe = allResults.stream()
        .collect(Collectors.toMap(
                TestResult::getTestName,
                TestResult::getStatus,
                (existing, replacement) -> replacement // при дубликате берём последний
        ));
```

### groupingBy — группировка

```java
// Группировка тестов по статусу
Map<Status, List<TestResult>> groupedByStatus = allResults.stream()
        .collect(Collectors.groupingBy(TestResult::getStatus));

// Группировка с подсчётом количества
Map<Status, Long> countByStatus = allResults.stream()
        .collect(Collectors.groupingBy(TestResult::getStatus, Collectors.counting()));

// Группировка с получением имён тестов
Map<Status, List<String>> namesByStatus = allResults.stream()
        .collect(Collectors.groupingBy(
                TestResult::getStatus,
                Collectors.mapping(TestResult::getTestName, Collectors.toList())
        ));
```

### joining — объединение строк

```java
// Объединение имён тестов через запятую
String joinedNames = testNames.stream()
        .collect(Collectors.joining(", "));

// С префиксом и суффиксом
String report = testNames.stream()
        .collect(Collectors.joining("\n  - ", "Тесты:\n  - ", "\n"));
// Результат: "Тесты:\n  - test1\n  - test2\n  - test3\n"

// Формирование CSV-строки
String csvLine = allResults.stream()
        .map(r -> String.join(",", r.getTestName(), r.getStatus().name(), String.valueOf(r.getDuration())))
        .collect(Collectors.joining("\n"));
```

### partitioningBy — разбиение на две группы

```java
// Разделить тесты на пройденные и не пройденные
Map<Boolean, List<TestResult>> partitioned = allResults.stream()
        .collect(Collectors.partitioningBy(r -> r.getStatus() == Status.PASSED));

List<TestResult> passed = partitioned.get(true);
List<TestResult> notPassed = partitioned.get(false);
```

---

## Императивный vs Декларативный стиль

### Задача: найти имена упавших тестов, отсортированные по длительности

**Императивный стиль (до Java 8):**

```java
// Императивный подход — описываем КАК делать
List<TestResult> failedTests = new ArrayList<>();
for (TestResult result : allResults) {
    if (result.getStatus() == Status.FAILED) {
        failedTests.add(result);
    }
}
failedTests.sort(Comparator.comparing(TestResult::getDuration).reversed());

List<String> failedNames = new ArrayList<>();
for (TestResult result : failedTests) {
    failedNames.add(result.getTestName());
}
```

**Декларативный стиль (Stream API):**

```java
// Декларативный подход — описываем ЧТО хотим получить
List<String> failedNames = allResults.stream()
        .filter(r -> r.getStatus() == Status.FAILED)
        .sorted(Comparator.comparing(TestResult::getDuration).reversed())
        .map(TestResult::getTestName)
        .collect(Collectors.toList());
```

| Критерий | Императивный | Декларативный (Stream) |
|---|---|---|
| Читаемость | Требует вникания в детали | Описывает намерение |
| Количество кода | Больше | Меньше |
| Параллелизм | Ручное управление | `parallelStream()` |
| Изменяемость | Мутабельные коллекции | Immutable pipeline |
| Отладка | Проще (breakpoints) | Сложнее (используем `peek`) |

---

## Практические примеры для QA

### Генерация отчёта о прогоне тестов

```java
public String generateTestReport(List<TestResult> results) {
    // Статистика по статусам
    Map<Status, Long> stats = results.stream()
            .collect(Collectors.groupingBy(TestResult::getStatus, Collectors.counting()));

    // Топ-5 самых долгих тестов
    String slowTests = results.stream()
            .sorted(Comparator.comparing(TestResult::getDuration).reversed())
            .limit(5)
            .map(r -> String.format("  %s — %dms", r.getTestName(), r.getDuration()))
            .collect(Collectors.joining("\n"));

    // Средняя длительность
    double avgDuration = results.stream()
            .mapToLong(TestResult::getDuration)
            .average()
            .orElse(0.0);

    return String.format("""
            === Отчёт о прогоне ===
            Всего тестов: %d
            Пройдено: %d
            Упало: %d
            Пропущено: %d
            Средняя длительность: %.0fms

            Самые долгие тесты:
            %s
            """,
            results.size(),
            stats.getOrDefault(Status.PASSED, 0L),
            stats.getOrDefault(Status.FAILED, 0L),
            stats.getOrDefault(Status.SKIPPED, 0L),
            avgDuration,
            slowTests);
}
```

### Обработка элементов страницы (Selenium)

```java
// Получить тексты всех видимых кнопок на странице
List<String> visibleButtonTexts = driver.findElements(By.tagName("button")).stream()
        .filter(WebElement::isDisplayed)
        .map(WebElement::getText)
        .filter(text -> !text.isBlank())
        .collect(Collectors.toList());

// Найти ссылку, содержащую определённый текст
Optional<WebElement> targetLink = driver.findElements(By.tagName("a")).stream()
        .filter(link -> link.getText().contains("Подробнее"))
        .findFirst();
```

---

## Связь с тестированием

- **Фильтрация результатов тестов** — отбор упавших, пропущенных, долгих тестов
- **Обработка тестовых данных** — трансформация CSV/JSON в объекты для data-driven тестов
- **Работа с WebElement** — фильтрация и маппинг элементов на странице
- **Генерация отчётов** — агрегация результатов по модулям, статусам, длительности
- **Утверждения (assertions)** — проверка условий для всех элементов коллекции
- **Подготовка тестовых данных** — генерация наборов данных через `Stream.generate()`

---

## Типичные ошибки

1. **Повторное использование stream** — stream одноразовый, после терминальной операции нельзя вызвать ещё одну
```java
Stream<String> stream = names.stream();
stream.forEach(System.out::println);
stream.count(); // IllegalStateException: stream has already been operated upon
```

2. **Забытая терминальная операция** — без неё конвейер не запустится
```java
// Ничего не произойдёт! Нет терминальной операции
allResults.stream()
        .filter(r -> r.getStatus() == Status.FAILED)
        .peek(r -> System.out.println(r)); // peek — промежуточная, не терминальная
```

3. **Модификация исходной коллекции внутри stream** — приводит к `ConcurrentModificationException`
```java
// НЕПРАВИЛЬНО
List<String> list = new ArrayList<>(List.of("a", "b", "c"));
list.stream().forEach(item -> list.remove(item)); // ConcurrentModificationException
```

4. **Использование `toMap` без обработки дубликатов ключей** — `IllegalStateException`

5. **Бесконечный stream без `limit()`** — программа зависнет

6. **Игнорирование порядка операций** — `filter` до `map` эффективнее, чем наоборот

---

## Вопросы на интервью

### Уровень Junior

- Что такое Stream API и чем stream отличается от коллекции?
- Какие способы создания stream вы знаете?
- Чем отличаются промежуточные и терминальные операции?
- Что делают `filter`, `map`, `collect`?
- Можно ли использовать stream повторно?

### Уровень Middle

- Чем `flatMap` отличается от `map`? Приведите пример.
- Что такое ленивость (laziness) в контексте stream?
- Как работает `reduce`? Какие есть перегрузки?
- Чем `findFirst` отличается от `findAny`?
- Как сгруппировать элементы с помощью `Collectors.groupingBy`?
- Как обработать дубликаты ключей в `Collectors.toMap`?

### Уровень Senior

- Как работает `parallelStream`? В каких случаях он даёт выигрыш?
- Как написать собственный `Collector`?
- Объясните разницу между `Stream.of(array)` и `Arrays.stream(array)` для примитивных массивов.
- Как stream обрабатывает `null`-элементы?
- В каких случаях stream менее эффективен, чем цикл?
- Как работают short-circuiting операции (`findFirst`, `anyMatch`, `limit`)?

---

## Практические задания

1. **Фильтрация тестов:** Дан список `TestResult` с полями `name`, `status`, `duration`, `module`. Получите имена всех упавших тестов модуля "payment", отсортированные по длительности выполнения.

2. **Статистика:** Из списка результатов тестов посчитайте процент прохождения для каждого модуля. Результат — `Map<String, Double>`, где ключ — имя модуля, значение — процент `PASSED`.

3. **Генератор данных:** С помощью `Stream.generate()` создайте список из 50 уникальных email-адресов для тестовых пользователей в формате `user_N@test.com`.

4. **Flat mapping:** Дан `List<TestSuite>`, каждый содержит `List<TestCase>`. Получите все уникальные теги (tags) из всех тест-кейсов всех сьютов одним списком.

5. **Отчёт:** Сгенерируйте текстовый отчёт: для каждого статуса выведите количество тестов и их имена, разделённые запятой. Используйте `groupingBy` и `joining`.

---

## Дополнительные ресурсы

- [Java Stream API Documentation](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/package-summary.html)
- [Baeldung — Java 8 Streams](https://www.baeldung.com/java-streams)
- [Baeldung — Guide to Java 8 Collectors](https://www.baeldung.com/java-8-collectors)
- [Java Stream API — Шпаргалка (Habr)](https://habr.com/ru/articles/348536/)
- Книга: "Modern Java in Action" — Manning Publications (главы 4-6)
