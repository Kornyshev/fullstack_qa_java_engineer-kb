# Пример задачи: регрессионное тестирование

## Обзор

Регрессионное тестирование — одна из ключевых повседневных задач QA Automation Engineer. Каждое утро (или после
каждого мержа в develop-ветку) запускается набор автоматизированных тестов, и QA должен проанализировать результаты,
отделить реальные баги от flaky-тестов и проблем окружения, и оперативно сообщить команде о найденных проблемах.

В этом разделе разберём полный цикл работы с регрессионным прогоном: от получения уведомления CI до финального
отчёта команде. Особое внимание уделим анализу Allure Report и работе с flaky-тестами.

---

## Сценарий

```
Ситуация:
Утро понедельника, 9:00. Ночной CI-прогон (scheduled pipeline) завершился.
В Slack пришло уведомление:

🔴 Nightly Regression — FAILED
Branch: develop
Duration: 42 min
Tests: 487 total | 451 passed | 28 failed | 8 broken
Allure Report: https://ci.example.com/allure/nightly-487

Задача QA: проанализировать результаты и принять меры.
```

---

## Шаг 1: Открыть Allure Report

### Обзорная страница (Overview)

При открытии Allure Report первое, что видит QA — обзорная страница.

```
Allure Report — Nightly Regression #487

Summary:
┌──────────────────────────────────────┐
│  Total: 487                          │
│  ██████████████████░░░░  Passed: 451 │
│  ██░                     Failed: 28  │
│  ░                       Broken: 8   │
│                                      │
│  Pass Rate: 92.6%                    │
│  Duration: 42m 17s                   │
└──────────────────────────────────────┘

Trend (последние 5 прогонов):
#483: 98.2% ✅
#484: 97.5% ✅
#485: 97.9% ✅
#486: 96.1% ⚠️
#487: 92.6% ❌  ← текущий (значительное падение)
```

**Первый вывод:** pass rate упал с ~97-98% до 92.6% — это значительное падение. Вероятно, есть реальные баги
или серьёзная проблема с окружением.

---

## Шаг 2: Категоризация Failures

### Разница между Failed и Broken

В Allure Report есть два типа неуспешных тестов:

| Тип | Описание | Причина |
|---|---|---|
| **Failed** (оранжевый) | Тест упал из-за assertion error | Вероятно, баг в продукте |
| **Broken** (фиолетовый) | Тест упал из-за infrastructure error | Проблема в тесте или окружении |

### Анализ Broken-тестов (8 штук)

```
Broken Tests (8):

1. testGetUserProfile — java.net.ConnectException: Connection refused
2. testUpdateUserEmail — java.net.ConnectException: Connection refused
3. testDeleteUser — java.net.ConnectException: Connection refused
4. testGetUserOrders — java.net.ConnectException: Connection refused
5. testGetOrderDetails — java.net.ConnectException: Connection refused
6. testSearchProducts — java.net.ConnectException: Connection refused
7. testGetProductReviews — java.net.ConnectException: Connection refused
8. testCreateReview — java.net.ConnectException: Connection refused
```

**Анализ:** Все 8 broken-тестов имеют одну и ту же ошибку — `ConnectException: Connection refused`.
Это типичная проблема окружения: сервис был недоступен в момент прогона.

**Действия:**
```
1. Проверить, работает ли сервис сейчас:
   curl -s http://staging:8080/actuator/health
   → {"status":"UP"} — сейчас работает

2. Проверить логи CI-прогона:
   → Сервис User Service рестартовался в 02:15 (прогон шёл с 02:00 до 02:42)
   → Тесты, зависящие от User Service, упали между 02:15 и 02:20

3. Категория: Environment Issue
4. Действие: сообщить DevOps о нестабильности сервиса на staging
5. Повторный запуск: перезапустить только эти 8 тестов
```

### Анализ Failed-тестов (28 штук)

Группируем failed-тесты по паттернам:

```
Группа 1: Новые failures (не было в предыдущих прогонах) — 12 тестов
├── testCreateOrder_withPromoCode — AssertionError: expected 200, got 500
├── testApplyPromoCode — AssertionError: expected discount 10%, got 0%
├── testPromoCodeExpired — AssertionError: expected 400, got 500
├── testPromoCodeMaxUsage — AssertionError: expected 400, got 500
├── ... (ещё 8 тестов, связанных с промокодами)
→ Паттерн: все тесты связаны с промокодами
→ Вероятная причина: баг в новом коммите

Группа 2: Flaky tests (были в предыдущих прогонах периодически) — 10 тестов
├── testLoginUI — StaleElementReferenceException
├── testSearchAutocomplete — TimeoutException (ждали 10 сек)
├── testImageUpload — AssertionError: file size mismatch
├── ... (ещё 7 тестов с историей нестабильности)
→ Паттерн: известные flaky-тесты
→ Действие: отмечаем как known flaky, планируем исправление

Группа 3: Неопределённые (нужно расследование) — 6 тестов
├── testPaymentVisa — AssertionError: expected "SUCCESS", got "PENDING"
├── testPaymentMastercard — AssertionError: expected "SUCCESS", got "PENDING"
├── testRefund — AssertionError: expected 200, got 503
├── testPaymentHistory — AssertionError: expected 5 items, got 0
├── testRecurringPayment — TimeoutException
├── testPaymentWebhook — AssertionError: webhook not received
→ Паттерн: все связаны с платёжным модулем
→ Нужно расследование: баг или проблема с payment sandbox?
```

---

## Шаг 3: Расследование новых failures

### Группа 1: Промокоды (12 тестов)

**Метод расследования:**

```
1. Определить, когда начались failures:
   - Прогон #486 (вчера): 3 из 12 промокодных тестов failed
   - Прогон #485 (позавчера): все промокодные тесты passed
   → Начались в прогоне #486

2. Найти коммиты между прогонами #485 и #486:
   git log --oneline develop --since="2025-03-14" --until="2025-03-16"
   → abc1234 — "feat: refactor promo code validation logic" (Автор: Алексей)
   → def5678 — "fix: update promo code expiry check" (Автор: Алексей)

3. Проверить изменения в коммите abc1234:
   git diff abc1234^..abc1234
   → Изменён PromoCodeService.java — переписана логика валидации

4. Попробовать воспроизвести вручную:
   POST /api/orders с промокодом → 500 Internal Server Error
   → Подтверждено: баг в новом коде

5. Проверить логи сервера:
   → NullPointerException в PromoCodeService.validateCode()
   → promoCode.getDiscount() вызывается на null-объекте
```

**Результат:** реальный баг в продукте. Создаём bug report.

**Bug Report:**
```
ID: BUG-789
Severity: Critical
Priority: High
Компонент: Promo Codes
Вызвано коммитом: abc1234

Заголовок: NPE в PromoCodeService при применении промокода — 500 ошибка

Описание:
После рефакторинга PromoCodeService (коммит abc1234) все операции
с промокодами возвращают 500 Internal Server Error.

Причина: NullPointerException в PromoCodeService.validateCode(),
строка 45 — promoCode.getDiscount() вызывается на null-объекте,
когда промокод не найден в БД.

Затронуто: 12 автотестов, все API-операции с промокодами

Шаги воспроизведения:
1. POST /api/orders с телом: {"promoCode": "SUMMER2025", ...}
2. Ожидаемый ответ: 200 с applied discount
3. Фактический ответ: 500 Internal Server Error

Логи: [ссылка на Kibana]
Allure Report: [ссылка на конкретные тесты]
```

### Группа 3: Платёжный модуль (6 тестов)

```
Расследование:

1. Проверить payment sandbox:
   curl -s https://sandbox.payment-provider.com/health
   → {"status": "degraded", "message": "Scheduled maintenance until 06:00 UTC"}
   → Sandbox на обслуживании!

2. Категория: External Dependency Issue
3. Действие: перезапустить тесты после 06:00 UTC
4. Результат перезапуска в 07:00: все 6 тестов — Passed ✅
```

---

## Шаг 4: Работа с Flaky-тестами

### Что такое Flaky Test

Flaky test — тест, который проходит и падает без изменений в коде. Причины:
- Зависимость от timing (race conditions)
- Зависимость от порядка выполнения
- Зависимость от внешних сервисов
- Нестабильные локаторы в UI-тестах
- Проблемы с тестовыми данными

### Анализ flaky-теста

```
Тест: testLoginUI
Класс: LoginPageTest
История (последние 10 прогонов): ✅✅❌✅❌✅✅✅❌✅ (70% pass rate)

Ошибка: StaleElementReferenceException at LoginPage.clickLoginButton()

Причина: После ввода email и пароля страница обновляет DOM
(например, убирает tooltip), и ссылка на кнопку становится stale.

Исправление:
```

```java
// До (нестабильно):
public void clickLoginButton() {
    loginButton.click(); // элемент может быть stale
}

// После (стабильно):
public void clickLoginButton() {
    // Ожидаем, что кнопка кликабельна, и используем свежий элемент
    new WebDriverWait(driver, Duration.ofSeconds(10))
            .until(ExpectedConditions.elementToBeClickable(
                    By.id("login-button")))
            .click();
}
```

### Стратегия управления Flaky-тестами

```
1. Обнаружение:
   - Allure TestOps автоматически помечает flaky-тесты
   - Ручной анализ: тест падает без изменений в коде

2. Классификация:
   - Minor flaky (редко падает, < 10%) → пометить, исправить когда будет время
   - Major flaky (часто падает, > 30%) → исправить в текущем спринте
   - Critical flaky (блокирует CI) → исправить немедленно

3. Исправление:
   - Добавить explicit waits
   - Убрать зависимость от порядка тестов
   - Использовать retry (как временная мера)
   - Изолировать тестовые данные
   - Заменить UI-тест на API-тест (если возможно)

4. Мониторинг:
   - Отслеживать flaky rate как метрику команды
   - Целевой показатель: < 2% flaky от общего количества тестов
```

---

## Шаг 5: Отчёт команде

### Сообщение в Slack

```
📊 Nightly Regression #487 — Анализ завершён

Категории failures:

🔴 Реальный баг (Critical):
BUG-789 — NPE в PromoCodeService при применении промокода
Затронуто: 12 тестов. Вызвано коммитом abc1234 (@Алексей)
Jira: https://jira.example.com/browse/BUG-789

🟡 Проблема окружения:
8 тестов — User Service рестартовался во время прогона
Перезапуск: 8/8 passed ✅
DevOps уведомлен о нестабильности staging

🟡 Внешняя зависимость:
6 тестов — Payment Sandbox на обслуживании
Перезапуск после 07:00: 6/6 passed ✅

🟠 Flaky-тесты:
10 тестов — известные нестабильные тесты
QA-456 (задача на исправление) приоритизирована на этот спринт

Итого реальный pass rate: 475/487 = 97.5% (без учёта env и flaky)
Блокер: BUG-789 — нужен fix до конца дня
```

### Обновление в Jira

```
Действия в Jira:

1. BUG-789 создан → Priority: High, Assignee: Алексей
2. QA-456 (flaky tests) → обновлён комментарий с текущим списком
3. QA-345 (регрессия) → комментарий: "Прогон #487 проанализирован,
   1 critical баг найден, остальные failures — env/flaky"
4. Dashboard обновлён: текущий flaky rate = 2.05% (10/487)
```

---

## Шаг 6: Перезапуск после исправлений

```
14:00 — Алексей исправил BUG-789 (коммит ghi9012)

14:15 — QA запускает targeted regression:
mvn clean test -Dgroups="promo" -Dbase.url=http://staging:8080

14:25 — Результат:
- 12/12 промокодных тестов — Passed ✅
- BUG-789 верифицирован в автотестах

14:30 — QA запускает full regression:
Trigger: Jenkins → Nightly Regression (manual trigger)

15:15 — Результат:
487 total | 478 passed | 9 failed (all known flaky)
Pass Rate: 98.2% ✅ (без flaky: 100%)

15:20 — Сообщение в Slack:
"✅ Full regression после fix BUG-789: 98.2% passed.
Все новые failures resolved. Оставшиеся 9 — known flaky."
```

---

## Allure Report: полезные фичи для анализа

### Categories (Категории ошибок)

Allure автоматически группирует ошибки:
```
Product Defects:      12 (промокоды)
Test Defects:          2 (ошибки в тестах)
Known Flaky:          10 (помеченные как flaky)
Unknown:               6 (платёжный модуль — до расследования)
Infrastructure:        8 (ConnectException)
```

### Timeline (Временная шкала)

Показывает, когда упали тесты — помогает определить timing-проблемы:
```
02:00 ███████████████████ tests running
02:15      ████ broken (User Service restart)
02:20 ███████████████████ tests continue
02:30          ██████ failed (promo code tests)
02:42 end
```

### Retries (Повторные запуски)

Если настроен retry, Allure показывает историю:
```
testLoginUI:
  Attempt 1: Failed (StaleElementException)
  Attempt 2: Passed ✅
  → Помечен как Flaky (прошёл со второй попытки)
```

### History Trend

```
Прогон  | Passed | Failed | Broken | Flaky
#483    | 479    | 3      | 0      | 5
#484    | 476    | 4      | 2      | 5
#485    | 478    | 2      | 0      | 7
#486    | 468    | 10     | 2      | 7
#487    | 451    | 28     | 8      | 10    ← аномалия
```

---

## Связь с тестированием

- Регрессионное тестирование — основной механизм обнаружения регрессий до релиза
- Умение анализировать Allure Report — ежедневный навык QA Automation Engineer
- Быстрая категоризация failures экономит время всей команды
- Управление flaky-тестами поддерживает доверие к автотестам
- Чёткий отчёт команде обеспечивает прозрачность процесса тестирования

---

## Типичные ошибки

1. **Игнорирование failures** — «вчера тоже падали, наверное flaky» (без расследования)
2. **Нет категоризации** — все failures свалены в одну кучу
3. **Нет отчёта команде** — QA проанализировал, но не сообщил
4. **Retry как решение** — вместо исправления flaky-теста добавляем retry=3
5. **Нет мониторинга тренда** — не замечаем постепенную деградацию pass rate
6. **Все failures = баги** — каждый failure превращается в bug report без расследования
7. **Нет root cause analysis** — «тест упал» без понимания «почему»
8. **Flaky-тесты не исправляются** — список растёт, доверие к автотестам падает

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Что такое регрессионное тестирование? Когда оно проводится?
2. Что такое Allure Report? Какие разделы он содержит?
3. Чем отличается Failed-тест от Broken-теста в Allure?
4. Что такое flaky test?
5. Как вы анализируете результаты CI-прогона?

### 🟡 Средний уровень
6. Как вы определяете, что failure — это реальный баг, а не проблема тестового окружения?
7. Как категоризировать 30 упавших тестов за 15 минут?
8. Какие метрики вы используете для оценки стабильности регрессионного прогона?
9. Как управлять flaky-тестами? Какие стратегии вы применяете?
10. Как организовать регрессионное тестирование для микросервисной архитектуры?

### 🔴 Продвинутый уровень
11. Как оптимизировать время регрессионного прогона (сейчас 2 часа, нужно 30 минут)?
12. Как определить минимальный набор тестов для regression (test impact analysis)?
13. Как построить quality gate в CI/CD на основе результатов регрессии?
14. Как организовать мониторинг flaky rate и автоматическую карантинизацию flaky-тестов?
15. Как реализовать selective regression (запускать только тесты, затронутые изменениями)?

---

## Практические задания

### Задание 1: Анализ Allure Report
Откройте Allure Report вашего проекта (или demo report на allurereport.org).
Проанализируйте: сколько failures, какие категории, есть ли тренд деградации.

### Задание 2: Flaky Test Hunt
Найдите 3 flaky-теста в вашем проекте. Для каждого:
- Определите root cause
- Предложите fix
- Оцените время на исправление

### Задание 3: Regression Report
Напишите отчёт команде по результатам регрессионного прогона:
- 500 тестов, 40 failures, 15 broken
- Категоризируйте, предложите действия

### Задание 4: Quality Gate
Определите критерии quality gate для вашего CI/CD:
- При каком pass rate блокировать мерж?
- Как обрабатывать known flaky?
- Как обрабатывать новые failures?

---

## Дополнительные ресурсы

- **Allure Report Documentation** (allurereport.org) — официальная документация
- **Allure TestOps** (docs.qameta.io) — платформа для управления тестами с flaky detection
- **Google Testing Blog** — статьи о flaky tests от Google
- **«Software Testing Automation Tips» by Gennadiy Alpaev** — практики автоматизации
- **Martin Fowler — «Eradicating Non-Determinism in Tests»** — классическая статья о flaky-тестах
