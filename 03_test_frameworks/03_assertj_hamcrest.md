# AssertJ и Hamcrest

## Обзор

AssertJ и Hamcrest — это библиотеки расширенных проверок (assertions), которые значительно улучшают читаемость и выразительность тестов по сравнению со встроенными assertions в JUnit 5 или TestNG. QA-инженеру критически важно уметь писать понятные, информативные проверки — от качества assertion-сообщений напрямую зависит скорость диагностики падений.

**Зависимости (Maven):**

```xml
<!-- AssertJ -->
<dependency>
    <groupId>org.assertj</groupId>
    <artifactId>assertj-core</artifactId>
    <version>3.25.3</version>
    <scope>test</scope>
</dependency>

<!-- Hamcrest -->
<dependency>
    <groupId>org.hamcrest</groupId>
    <artifactId>hamcrest</artifactId>
    <version>2.2</version>
    <scope>test</scope>
</dependency>
```

---

## AssertJ — Fluent Assertions

AssertJ использует fluent API (цепочку вызовов), что делает проверки читаемыми как предложения на английском языке.

### Проверка строк

```java
import static org.assertj.core.api.Assertions.*;

@Test
void stringAssertions() {
    String email = "user@example.com";

    assertThat(email)
        .isNotNull()
        .isNotEmpty()
        .contains("@")
        .startsWith("user")
        .endsWith(".com")
        .doesNotContain(" ")
        .hasSize(16)
        .matches("^[\\w.]+@[\\w.]+\\.[a-z]{2,}$");
}

@Test
void stringComparisonAssertions() {
    String actual = "  Hello World  ";

    assertThat(actual)
        .containsIgnoringCase("hello")
        .isEqualToIgnoringWhitespace("Hello World")
        .isNotBlank();

    // Проверка подстроки
    assertThat("Error: file not found")
        .startsWith("Error")
        .containsAnyOf("not found", "missing", "absent");
}
```

### Проверка чисел

```java
@Test
void numberAssertions() {
    int statusCode = 200;
    double price = 19.99;

    assertThat(statusCode)
        .isPositive()
        .isEqualTo(200)
        .isBetween(200, 299)
        .isGreaterThanOrEqualTo(200)
        .isLessThan(300);

    assertThat(price)
        .isCloseTo(20.0, within(0.01))  // Допуск для double
        .isGreaterThan(0.0)
        .isNotZero();
}
```

### Проверка коллекций

```java
@Test
void collectionAssertions() {
    List<String> roles = List.of("ADMIN", "USER", "MODERATOR");

    assertThat(roles)
        .isNotEmpty()
        .hasSize(3)
        .contains("ADMIN", "USER")
        .containsExactly("ADMIN", "USER", "MODERATOR")  // Точный порядок
        .containsExactlyInAnyOrder("USER", "MODERATOR", "ADMIN")
        .doesNotContain("GUEST")
        .doesNotHaveDuplicates()
        .startsWith("ADMIN")
        .endsWith("MODERATOR");
}

@Test
void collectionObjectAssertions() {
    List<User> users = List.of(
        new User("Иван", 25, "ivan@mail.com"),
        new User("Мария", 30, "maria@mail.com"),
        new User("Пётр", 22, "petr@mail.com")
    );

    assertThat(users)
        .hasSize(3)
        .extracting(User::getName)
        .containsExactly("Иван", "Мария", "Пётр");

    assertThat(users)
        .extracting(User::getName, User::getAge)
        .containsExactly(
            tuple("Иван", 25),
            tuple("Мария", 30),
            tuple("Пётр", 22)
        );

    // Фильтрация и проверка
    assertThat(users)
        .filteredOn(user -> user.getAge() > 24)
        .hasSize(2)
        .extracting(User::getName)
        .containsExactly("Иван", "Мария");

    // Проверка, что все элементы удовлетворяют условию
    assertThat(users)
        .allMatch(user -> user.getEmail().contains("@"))
        .anyMatch(user -> user.getAge() > 28)
        .noneMatch(user -> user.getName().isEmpty());
}
```

### Проверка Map

```java
@Test
void mapAssertions() {
    Map<String, Integer> scores = Map.of(
        "Alice", 95,
        "Bob", 87,
        "Charlie", 92
    );

    assertThat(scores)
        .isNotEmpty()
        .hasSize(3)
        .containsKey("Alice")
        .containsKeys("Alice", "Bob")
        .containsValue(95)
        .containsEntry("Alice", 95)
        .doesNotContainKey("Dave")
        .doesNotContainEntry("Bob", 100);
}
```

### Проверка исключений

```java
@Test
void exceptionAssertions() {
    // Способ 1: assertThatThrownBy
    assertThatThrownBy(() -> userService.findById(null))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessage("ID не может быть null")
        .hasMessageContaining("null")
        .hasNoCause();

    // Способ 2: assertThatExceptionOfType
    assertThatExceptionOfType(NotFoundException.class)
        .isThrownBy(() -> userService.findById(999L))
        .withMessage("Пользователь с ID 999 не найден")
        .withMessageStartingWith("Пользователь");

    // Способ 3: assertThatCode — проверка отсутствия исключения
    assertThatCode(() -> userService.findById(1L))
        .doesNotThrowAnyException();

    // Специальные утверждения для типичных исключений
    assertThatIllegalArgumentException()
        .isThrownBy(() -> new User("", -1, ""))
        .withMessageContaining("невалидные данные");

    assertThatNullPointerException()
        .isThrownBy(() -> userService.findById(null));
}
```

### Проверка Optional

```java
@Test
void optionalAssertions() {
    Optional<User> found = userService.findByEmail("ivan@mail.com");
    Optional<User> notFound = userService.findByEmail("unknown@mail.com");

    assertThat(found)
        .isPresent()
        .hasValueSatisfying(user -> {
            assertThat(user.getName()).isEqualTo("Иван");
            assertThat(user.getAge()).isGreaterThan(18);
        });

    assertThat(notFound)
        .isEmpty();

    // Проверка значения Optional напрямую
    assertThat(found)
        .get()  // Разворачивает Optional
        .extracting(User::getEmail)
        .isEqualTo("ivan@mail.com");
}
```

### Soft Assertions в AssertJ

Все проверки выполняются, и все ошибки собираются в один отчёт.

```java
@Test
void softAssertionsExample() {
    User user = userService.getById(1L);

    // Способ 1: SoftAssertions объект
    SoftAssertions softly = new SoftAssertions();
    softly.assertThat(user.getName()).isEqualTo("Иван");
    softly.assertThat(user.getAge()).isBetween(18, 65);
    softly.assertThat(user.getEmail()).contains("@");
    softly.assertThat(user.isActive()).isTrue();
    softly.assertAll();  // Выбрасывает все накопленные ошибки

    // Способ 2: assertSoftly — assertAll() вызывается автоматически
    assertSoftly(soft -> {
        soft.assertThat(user.getName()).isEqualTo("Иван");
        soft.assertThat(user.getAge()).isBetween(18, 65);
        soft.assertThat(user.getEmail()).contains("@");
        soft.assertThat(user.isActive()).isTrue();
    });
}
```

### Custom Assertions

Создание собственных assertion-классов для доменных объектов.

```java
// Кастомный assertion для User
public class UserAssert extends AbstractAssert<UserAssert, User> {

    protected UserAssert(User actual) {
        super(actual, UserAssert.class);
    }

    // Точка входа
    public static UserAssert assertThat(User actual) {
        return new UserAssert(actual);
    }

    public UserAssert hasName(String expectedName) {
        isNotNull();
        if (!actual.getName().equals(expectedName)) {
            failWithMessage("Ожидалось имя <%s>, но было <%s>",
                    expectedName, actual.getName());
        }
        return this;
    }

    public UserAssert isActive() {
        isNotNull();
        if (!actual.isActive()) {
            failWithMessage("Пользователь '%s' должен быть активен", actual.getName());
        }
        return this;
    }

    public UserAssert isAdult() {
        isNotNull();
        if (actual.getAge() < 18) {
            failWithMessage("Пользователь '%s' с возрастом %d — несовершеннолетний",
                    actual.getName(), actual.getAge());
        }
        return this;
    }
}

// Использование в тесте
@Test
void customAssertionExample() {
    User user = userService.getById(1L);

    UserAssert.assertThat(user)
        .hasName("Иван")
        .isActive()
        .isAdult();
}
```

---

## Hamcrest — Matcher-based Assertions

Hamcrest использует концепцию matchers — объектов, описывающих ожидания. Assertions строятся по шаблону `assertThat(actual, matcher)`.

### Базовые матчеры

```java
import static org.hamcrest.MatcherAssert.assertThat;
import static org.hamcrest.Matchers.*;

@Test
void basicMatchers() {
    String name = "Иван";
    int age = 25;

    assertThat(name, is("Иван"));                    // Равенство
    assertThat(name, equalTo("Иван"));                // Равенство (синоним)
    assertThat(name, is(not("Пётр")));                // Не равно
    assertThat(name, is(notNullValue()));              // Не null
    assertThat(age, is(greaterThan(18)));              // Больше
    assertThat(age, is(lessThanOrEqualTo(30)));        // Меньше или равно
    assertThat(age, is(both(greaterThan(18)).and(lessThan(65)))); // Диапазон
}
```

### Матчеры для строк

```java
@Test
void stringMatchers() {
    String message = "Error: Connection timed out";

    assertThat(message, containsString("Connection"));
    assertThat(message, startsWith("Error"));
    assertThat(message, endsWith("out"));
    assertThat(message, containsStringIgnoringCase("connection"));
    assertThat(message, matchesRegex("Error:.*timed out"));
    assertThat(message, not(emptyString()));
    assertThat(message, not(blankString()));
}
```

### Матчеры для коллекций

```java
@Test
void collectionMatchers() {
    List<String> tags = List.of("smoke", "regression", "api");

    assertThat(tags, hasSize(3));
    assertThat(tags, hasItem("smoke"));              // Содержит элемент
    assertThat(tags, hasItems("smoke", "api"));      // Содержит элементы
    assertThat(tags, contains("smoke", "regression", "api"));  // Точный порядок
    assertThat(tags, containsInAnyOrder("api", "smoke", "regression"));
    assertThat(tags, not(hasItem("ui")));
    assertThat(tags, everyItem(is(not(emptyString()))));  // Каждый элемент непуст
    assertThat(tags, not(empty()));
}

@Test
void mapMatchers() {
    Map<String, Integer> config = Map.of("timeout", 5000, "retries", 3);

    assertThat(config, hasKey("timeout"));
    assertThat(config, hasValue(5000));
    assertThat(config, hasEntry("retries", 3));
    assertThat(config, aMapWithSize(2));
}
```

### Комбинирование матчеров

```java
@Test
void combinedMatchers() {
    String response = "HTTP 200 OK";

    // allOf — все условия должны выполняться (AND)
    assertThat(response, allOf(
        containsString("HTTP"),
        containsString("200"),
        containsString("OK")
    ));

    // anyOf — хотя бы одно условие (OR)
    assertThat(response, anyOf(
        containsString("200"),
        containsString("201"),
        containsString("204")
    ));

    // not — отрицание
    assertThat(response, not(containsString("Error")));
}
```

### Custom Matcher

```java
// Кастомный матчер для проверки email
public class IsValidEmail extends TypeSafeMatcher<String> {

    @Override
    protected boolean matchesSafely(String item) {
        // Простая проверка формата email
        return item != null && item.matches("^[\\w.+]+@[\\w.]+\\.[a-zA-Z]{2,}$");
    }

    @Override
    public void describeTo(Description description) {
        // Описание ожидания при падении
        description.appendText("валидный email-адрес");
    }

    @Override
    protected void describeMismatchSafely(String item, Description mismatchDescription) {
        // Описание реального значения при падении
        mismatchDescription.appendText("получен ").appendValue(item);
    }

    // Фабричный метод для удобного использования
    public static Matcher<String> isValidEmail() {
        return new IsValidEmail();
    }
}

// Использование
@Test
void customMatcherExample() {
    assertThat("user@mail.com", isValidEmail());
    assertThat("invalid", not(isValidEmail()));
}
```

---

## Сравнительная таблица: AssertJ vs Hamcrest vs JUnit Assertions

| Критерий | JUnit 5 Assertions | Hamcrest | AssertJ |
|---|---|---|---|
| Стиль API | Статические методы | Matcher-based | Fluent (цепочка) |
| Автодополнение IDE | Среднее | Слабое | **Отличное** |
| Читаемость | Базовая | Хорошая | **Отличная** |
| Сообщения об ошибках | Ручные | Автоматические | **Автоматические** |
| Расширяемость | Нет | Custom Matchers | Custom Assertions |
| Soft Assertions | Нет (assertAll) | Нет | **Да** |
| Коллекции | Базово | Хорошо | **Богатый API** |
| Optional | Нет | Нет | **Да** |
| Исключения | `assertThrows` | Нет | **assertThatThrownBy** |
| Зависимости | Встроен | Отдельная | Отдельная |

### Пример — одна и та же проверка в трёх стилях

```java
// Проверка: список содержит "admin" и имеет размер 3

// JUnit 5 Assertions
assertEquals(3, roles.size(), "Размер списка должен быть 3");
assertTrue(roles.contains("admin"), "Список должен содержать admin");

// Hamcrest
assertThat(roles, allOf(hasSize(3), hasItem("admin")));

// AssertJ
assertThat(roles).hasSize(3).contains("admin");
```

### Пример — сообщения об ошибках

```java
List<String> actual = List.of("A", "B", "C");

// JUnit 5: "Expected [D] to be in the list" — только если вы сами написали сообщение
assertTrue(actual.contains("D"), "Expected [D] to be in the list");

// Hamcrest: автоматическое сообщение
// "Expected: a collection containing 'D'
//  but: was <[A, B, C]>"
assertThat(actual, hasItem("D"));

// AssertJ: самое подробное автоматическое сообщение
// "Expecting ArrayList: [A, B, C] to contain: [D]
//  but could not find the following element(s): [D]"
assertThat(actual).contains("D");
```

---

## Практические примеры в тестовом коде

### Тестирование REST API ответа

```java
@Test
void shouldReturnUserListFromApi() {
    // Act — вызов API
    Response response = apiClient.get("/api/users");
    List<UserDto> users = response.jsonPath().getList(".", UserDto.class);

    // Assert с AssertJ — проверка ответа
    assertThat(response.getStatusCode()).isEqualTo(200);

    assertThat(users)
        .isNotEmpty()
        .hasSizeGreaterThanOrEqualTo(1)
        .allSatisfy(user -> {
            assertThat(user.getId()).isPositive();
            assertThat(user.getEmail()).contains("@");
            assertThat(user.getName()).isNotBlank();
        });

    // Проверка конкретного пользователя
    assertThat(users)
        .filteredOn(UserDto::getRole, "ADMIN")
        .singleElement()
        .satisfies(admin -> {
            assertThat(admin.getName()).isEqualTo("Admin User");
            assertThat(admin.isActive()).isTrue();
        });
}
```

### Тестирование бизнес-логики

```java
@Test
void shouldCalculateOrderTotalCorrectly() {
    Order order = Order.builder()
        .addItem("Laptop", 999.99, 1)
        .addItem("Mouse", 29.99, 2)
        .withDiscount(10) // 10% скидка
        .build();

    // AssertJ: проверка расчёта
    assertSoftly(soft -> {
        soft.assertThat(order.getSubtotal())
            .as("Промежуточная сумма")
            .isCloseTo(1059.97, within(0.01));
        soft.assertThat(order.getDiscount())
            .as("Сумма скидки")
            .isCloseTo(105.997, within(0.01));
        soft.assertThat(order.getTotal())
            .as("Итого")
            .isCloseTo(953.973, within(0.01));
        soft.assertThat(order.getItems())
            .as("Количество позиций")
            .hasSize(2);
    });
}
```

### Тестирование с Hamcrest и REST Assured

```java
@Test
void shouldReturnUserById() {
    given()
        .pathParam("id", 1)
    .when()
        .get("/api/users/{id}")
    .then()
        .statusCode(200)
        .body("name", equalTo("Иван"))
        .body("age", greaterThan(18))
        .body("roles", hasItem("USER"))
        .body("email", matchesRegex("^[\\w.]+@[\\w.]+$"));
}
```

---

## Связь с тестированием

- **AssertJ** — стандарт для Java-проектов. Fluent API ускоряет написание тестов, а автоматические сообщения ускоряют диагностику
- **Hamcrest** — широко используется в REST Assured, Spring MockMvc, Espresso (Android). Матчеры являются его основой
- **Soft Assertions** — незаменимы при проверке объектов с множеством полей (DTO, API-ответы)
- **Custom Assertions/Matchers** — повышают переиспользуемость и читаемость в крупных тестовых проектах
- **Выбор библиотеки** часто определяется проектом: AssertJ — для новых проектов, Hamcrest — при использовании REST Assured

---

## Типичные ошибки

1. **Импорт `assertThat` не из того пакета** — у AssertJ, Hamcrest и JUnit разные `assertThat`, IDE может подставить не тот
2. **Забыть `assertAll()` в SoftAssertions** — все проверки пройдут «молча», ошибки не зафиксируются
3. **Использовать `isEqualTo` для double** — сравнение с плавающей точкой требует `isCloseTo` с допуском
4. **Слишком длинные цепочки AssertJ** — лучше разбить на логические группы с `as("описание")` для каждой
5. **Не использовать `as()` / описания** — при падении неясно, какая именно проверка провалилась
6. **Hamcrest `is()` без статического импорта** — конфликт с Mockito `is()`, нужно явно импортировать `org.hamcrest.Matchers.is`
7. **Забыть `extracting()` при проверке списков объектов** — вместо этого пишут циклы с `get(i)`, что нечитаемо
8. **Сравнение `contains` vs `containsExactly`** — `contains` не проверяет порядок и полноту, `containsExactly` проверяет оба

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Чем AssertJ отличается от стандартных JUnit assertions?
2. Что такое fluent API и как он применяется в AssertJ?
3. Как проверить, что список содержит определённый элемент, в AssertJ и Hamcrest?
4. Как проверить выбрасывание исключения в AssertJ?
5. Что такое Hamcrest matcher? Приведите примеры.

### 🟡 Средний уровень
6. Чем `contains` отличается от `containsExactly` и `containsExactlyInAnyOrder` в AssertJ?
7. Как работают Soft Assertions? Какая типичная ошибка при их использовании?
8. Как в AssertJ проверить содержимое Optional?
9. Как скомбинировать несколько Hamcrest-матчеров через `allOf` / `anyOf`?
10. Когда использовать `extracting()` и `filteredOn()`?

### 🔴 Продвинутый уровень
11. Как создать Custom Assertion в AssertJ? Зачем это нужно?
12. Как создать Custom Matcher в Hamcrest? В чём отличие от Custom Assertion?
13. Сравните подходы к расширяемости AssertJ и Hamcrest.
14. Как интегрируются Hamcrest matchers с REST Assured и Spring MockMvc?
15. Когда стоит выбрать Hamcrest вместо AssertJ и наоборот?

---

## Практические задания

### Задание 1: Fluent Assertions
Напишите тест для API-ответа, содержащего список товаров. Используя AssertJ, проверьте: размер списка, наличие товара по имени, диапазон цен, отсутствие дубликатов по ID.

### Задание 2: Soft Assertions
Напишите тест для `UserProfileDto`, проверяющий 8+ полей. Используйте `assertSoftly` и `as()` для каждой проверки.

### Задание 3: Custom Assertion
Создайте `OrderAssert` для класса `Order` с методами: `hasTotalGreaterThan()`, `containsProduct()`, `isPaid()`, `hasItemCount()`.

### Задание 4: Hamcrest Custom Matcher
Создайте матчер `isValidPhoneNumber()` для проверки формата телефона. Обеспечьте информативные сообщения об ошибках.

### Задание 5: Сравнение подходов
Напишите один и тот же набор из 10 проверок тремя способами (JUnit, Hamcrest, AssertJ). Сравните читаемость, количество кода и информативность сообщений при падении.

---

## Дополнительные ресурсы

- [AssertJ Official Documentation](https://assertj.github.io/doc/)
- [AssertJ GitHub](https://github.com/assertj/assertj)
- [Hamcrest Tutorial](http://hamcrest.org/JavaHamcrest/tutorial)
- [Hamcrest GitHub](https://github.com/hamcrest/JavaHamcrest)
- [Baeldung — AssertJ](https://www.baeldung.com/introduction-to-assertj)
- [Baeldung — Hamcrest](https://www.baeldung.com/java-junit-hamcrest-guide)
