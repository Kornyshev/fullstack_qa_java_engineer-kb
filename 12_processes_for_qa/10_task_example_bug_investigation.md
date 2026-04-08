# Пример задачи: расследование бага на проде

## Обзор

Расследование production-инцидента — одна из самых ответственных и стрессовых задач QA-инженера. В отличие от
тестирования в спринте, здесь на кону реальные пользователи и реальные потери бизнеса. QA должен быстро понять,
что происходит, собрать доказательства, помочь локализовать проблему и обеспечить качественную верификацию fix.

В этом разделе разберём полный цикл расследования бага на production: от получения алерта до post-mortem.
Рассмотрим работу с логами (ELK/Kibana), мониторингом (Grafana), базой данных и воспроизведение проблемы локально.

---

## Сценарий

```
Ситуация:
Четверг, 14:30. В Slack-канале #alerts появляется сообщение:

🚨 ALERT: Payment Service — Error Rate > 5%
Service: payment-service
Environment: production
Metric: error_rate = 8.3% (threshold: 5%)
Started: 14:25 UTC
Dashboard: https://grafana.example.com/d/payment

Одновременно в канале #support:
"Клиенты жалуются, что не могут оплатить заказ.
Получают ошибку 'Что-то пошло не так. Попробуйте позже.'"

Задача QA: расследовать инцидент, помочь локализовать проблему,
верифицировать fix.
```

---

## Шаг 1: Первичная оценка (Triage)

### Оценка severity и impact

**Первые 5 минут — критические.** QA должен быстро оценить масштаб проблемы.

```
Чеклист первичной оценки:

1. Что затронуто?
   → Оплата заказов — критический бизнес-процесс

2. Сколько пользователей затронуто?
   → Grafana: 8.3% error rate → примерно 1 из 12 платежей fail

3. Когда началось?
   → 14:25 UTC (5 минут назад)

4. Был ли деплой?
   → Проверяю deployment history: последний деплой payment-service
     в 14:20 UTC (версия 2.4.1 → 2.4.2)
   → СОВПАДЕНИЕ: проблема началась через 5 минут после деплоя!

5. Severity Assessment:
   → Severity: Critical (затронут основной revenue stream)
   → Impact: ~8% транзакций fail
   → Trend: стабильный (не растёт, не падает)
```

### Уведомление команды

```
Сообщение в #incidents (14:35):

🔴 INCIDENT: Оплата — error rate 8.3%

Начало: 14:25 UTC
Совпадение: деплой payment-service 2.4.2 в 14:20 UTC
Затронуто: ~8% платёжных транзакций
Trend: стабильный

Начинаю расследование. Прошу:
- @DevOps: быть готовым к rollback
- @dev-lead: подключиться к расследованию
- @support: собирать примеры затронутых заказов (order_id)
```

---

## Шаг 2: Проверка мониторинга (Grafana)

### Анализ метрик

```
Grafana Dashboard: Payment Service

1. Error Rate:
   ┌──────────────────────────────────────┐
   │ 10% ·                    ·····       │
   │  8% ·               ·····     ·     │
   │  6% ·                               │
   │  4% ·                               │
   │  2% ·                               │
   │  0% ···············                  │
   │    13:00  13:30  14:00  14:30  15:00 │
   └──────────────────────────────────────┘
   → Резкий скачок в 14:25

2. Response Time (p95):
   ┌──────────────────────────────────────┐
   │ 5s  ·                     ····       │
   │ 3s  ·                ····     ··     │
   │ 1s  ·                               │
   │ 0.5 ···············                  │
   └──────────────────────────────────────┘
   → P95 вырос с 500ms до 3-5 секунд

3. Throughput:
   → Стабильный (количество запросов не изменилось)

4. HTTP Status Codes:
   → 200: 91.7% (было 99.8%)
   → 500: 8.3% (было 0.2%)
   → Рост 500-ок совпадает с error rate

5. Downstream Dependencies:
   → Database: OK (response time стабильный)
   → Payment Gateway: OK (error rate 0.1%)
   → Redis Cache: OK
   → Notification Service: OK
```

**Выводы из мониторинга:**
- Проблема в самом payment-service, не в зависимостях
- Начало совпадает с деплоем 2.4.2
- Не все запросы fail (8.3%) → возможно, конкретный сценарий или конкретный тип карт

---

## Шаг 3: Анализ логов (ELK/Kibana)

### Поиск ошибок в Kibana

```
Kibana Query:
service: "payment-service" AND level: "ERROR" AND @timestamp >= "2025-03-20T14:25:00"

Результат: 247 записей за последние 10 минут
```

### Анализ ошибок

```
Топ ошибок (агрегация):

1. NullPointerException in PaymentProcessor.processPayment() — 198 записей
   at com.example.payment.PaymentProcessor.processPayment(PaymentProcessor.java:87)
   at com.example.payment.PaymentController.createPayment(PaymentController.java:42)

2. TimeoutException in PaymentGatewayClient.charge() — 34 записи
   at com.example.payment.client.PaymentGatewayClient.charge(PaymentGatewayClient.java:56)

3. IllegalArgumentException: Currency code cannot be null — 15 записей
   at com.example.payment.model.Money.<init>(Money.java:23)
```

### Детальный анализ главной ошибки

```
Kibana — детали ошибки #1 (NullPointerException):

{
  "timestamp": "2025-03-20T14:26:15.234Z",
  "level": "ERROR",
  "service": "payment-service",
  "version": "2.4.2",
  "trace_id": "abc123def456",
  "message": "Failed to process payment",
  "exception": {
    "class": "java.lang.NullPointerException",
    "message": "Cannot invoke method getDiscount() on null reference",
    "stacktrace": [
      "at com.example.payment.PaymentProcessor.processPayment(PaymentProcessor.java:87)",
      "at com.example.payment.PaymentController.createPayment(PaymentController.java:42)"
    ]
  },
  "context": {
    "order_id": "ORD-2025-78234",
    "user_id": 45678,
    "amount": 15990,
    "currency": "RUB",
    "payment_method": "CARD",
    "promo_code": "SPRING2025"
  }
}
```

**Ключевое наблюдение:** ошибка `getDiscount() on null reference` возникает при наличии `promo_code`.

```
Проверка гипотезы:

Kibana Query: service: "payment-service" AND level: "ERROR"
  AND context.promo_code: *

→ 198 из 198 ошибок NPE содержат promo_code!

Kibana Query: service: "payment-service" AND level: "INFO"
  AND message: "Payment successful" AND context.promo_code: *

→ 0 записей! Ни одна оплата с промокодом не прошла успешно.

Вывод: 100% оплат с промокодом fail. Оплаты без промокода работают нормально.
```

---

## Шаг 4: Проверка в базе данных

### SQL-запросы для верификации

```sql
-- Количество неуспешных платежей за последний час
SELECT status, COUNT(*) as cnt
FROM payments
WHERE created_at >= NOW() - INTERVAL '1 hour'
GROUP BY status;

-- Результат:
-- SUCCESS: 1847
-- FAILED:  167
-- PENDING: 12

-- Проверка: все ли failed платежи связаны с промокодом
SELECT
    p.status,
    p.promo_code,
    COUNT(*) as cnt
FROM payments p
WHERE p.created_at >= NOW() - INTERVAL '1 hour'
  AND p.status = 'FAILED'
GROUP BY p.status, p.promo_code;

-- Результат:
-- FAILED | SPRING2025    | 89
-- FAILED | WELCOME10     | 45
-- FAILED | MARCH2025     | 28
-- FAILED | NULL          |  5  ← эти 5 — другие ошибки (timeout и т.д.)

-- Вывод: 162 из 167 failed платежей связаны с промокодами
```

---

## Шаг 5: Воспроизведение локально

### Локальное воспроизведение

```bash
# Скачиваем версию 2.4.2
git checkout payment-service-2.4.2

# Запускаем локально
docker-compose up payment-service

# Воспроизводим
curl -X POST http://localhost:8080/api/payments \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "orderId": "TEST-001",
    "amount": 1000,
    "currency": "RUB",
    "paymentMethod": "CARD",
    "promoCode": "SPRING2025"
  }'

# Результат: 500 Internal Server Error
# → Воспроизведено!

# Проверяем без промокода
curl -X POST http://localhost:8080/api/payments \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "orderId": "TEST-002",
    "amount": 1000,
    "currency": "RUB",
    "paymentMethod": "CARD"
  }'

# Результат: 201 Created
# → Без промокода работает
```

### Анализ кода

```java
// PaymentProcessor.java (версия 2.4.2)
// Строка 87 — место ошибки:

public PaymentResult processPayment(PaymentRequest request) {
    BigDecimal amount = request.getAmount();

    // Применяем промокод (если есть)
    if (request.getPromoCode() != null) {
        // Ищем промокод в новой таблице (миграция в 2.4.2)
        PromoCode promo = promoCodeRepository.findByCodeV2(request.getPromoCode());
        // BUG: promo может быть null, если промокод не мигрирован в новую таблицу!
        amount = amount.subtract(promo.getDiscount()); // ← NPE здесь!
    }

    return paymentGateway.charge(amount, request.getCurrency());
}
```

**Root Cause найден:**
В версии 2.4.2 была выполнена миграция промокодов в новую таблицу (`promo_codes_v2`), но метод
`findByCodeV2()` ищет только в новой таблице. Старые промокоды (`SPRING2025`, `WELCOME10`, `MARCH2025`)
не были мигрированы → `findByCodeV2()` возвращает `null` → NPE.

---

## Шаг 6: Решение о действиях

### Обсуждение с командой (14:55)

```
Варианты:

1. Rollback на версию 2.4.1
   + Быстро (5 минут)
   + Гарантированно работает
   - Откатываем все изменения 2.4.2 (включая полезные)

2. Hotfix
   + Точечное исправление
   - Нужно 30-60 минут на разработку, тестирование и деплой

3. Feature flag (выключить промокоды)
   + Быстро (если есть feature flag)
   - Промокоды перестанут работать для всех пользователей

Решение команды: Rollback на 2.4.1 (быстрое восстановление)
+ параллельно готовить hotfix 2.4.3 с migration fix
```

---

## Шаг 7: Верификация после rollback

```
15:05 — Rollback на payment-service 2.4.1 выполнен

QA Verification Checklist:

1. ☐ Health check: curl https://api.example.com/payment/health
   → {"status":"UP","version":"2.4.1"} ✅

2. ☐ Оплата без промокода:
   → Тестовый заказ TEST-003: 201 Created ✅

3. ☐ Оплата с промокодом:
   → Тестовый заказ TEST-004 с SPRING2025: 201 Created ✅

4. ☐ Мониторинг Grafana:
   → Error rate: упал с 8.3% до 0.3% за 2 минуты ✅
   → Response time p95: вернулся к 500ms ✅

5. ☐ Kibana:
   → Новые NPE в PaymentProcessor: 0 за последние 5 минут ✅

6. ☐ Проверка бизнес-метрик:
   → Успешные платежи: восстановились до нормы ✅

Результат: Rollback успешен. Инцидент resolved.
Время: 15:10 UTC (общая длительность: 45 минут)
```

---

## Шаг 8: Верификация Hotfix

```
17:00 — Hotfix 2.4.3 готов

Изменения в hotfix:
1. Добавлен fallback: если промокод не найден в v2 таблице,
   ищем в v1 таблице
2. Добавлена миграция старых промокодов в v2 таблицу
3. Добавлен null-check перед вызовом getDiscount()

QA Verification на staging:

1. ☐ Оплата со старым промокодом (SPRING2025): 201 ✅
2. ☐ Оплата с новым промокодом: 201 ✅
3. ☐ Оплата с невалидным промокодом: 400 "Промокод не найден" ✅
4. ☐ Оплата без промокода: 201 ✅
5. ☐ Миграция: проверить, что старые промокоды есть в v2 таблице ✅
6. ☐ Regression: запуск полного набора payment-тестов → 47/47 passed ✅

Рекомендация QA: Hotfix 2.4.3 готов к деплою ✅
```

---

## Шаг 9: Написание подробного Bug Report

```
ID: INC-2025-037
Severity: Critical
Type: Production Incident
Duration: 45 minutes (14:25 — 15:10 UTC)
Affected Users: ~8% платёжных транзакций

Заголовок: NPE при оплате с промокодом после деплоя payment-service 2.4.2

Root Cause:
В версии 2.4.2 промокоды мигрированы в новую таблицу (promo_codes_v2),
но старые промокоды не были перенесены. Метод findByCodeV2() возвращал
null для старых промокодов, что приводило к NullPointerException
при вызове getDiscount().

Impact:
- 162 неуспешных платежа за 45 минут
- Примерная потеря: ~487,000 RUB (средний чек × количество failures)
- Затронутые промокоды: SPRING2025 (89), WELCOME10 (45), MARCH2025 (28)

Timeline:
14:20 — Деплой payment-service 2.4.2
14:25 — Первые ошибки в логах
14:30 — Алерт в Slack (error rate > 5%)
14:35 — QA начал расследование
14:55 — Root cause определён
15:00 — Принято решение о rollback
15:05 — Rollback на 2.4.1 выполнен
15:10 — Верификация — инцидент resolved

Resolution:
- Immediate: rollback на 2.4.1
- Permanent: hotfix 2.4.3 (деплой запланирован на 17:30)

Lessons Learned:
- Автотесты не покрывали сценарий "оплата со старым промокодом после миграции"
- Нет интеграционного теста для миграции данных
- Canary deployment мог бы ограничить blast radius
```

---

## Шаг 10: Post-Mortem и предотвращение

### Action Items

```
Post-Mortem — INC-2025-037

Что пошло не так:
1. Миграция данных не была протестирована с реальными промокодами
2. Нет null-check для результата findByCodeV2()
3. Нет canary deployment для payment-service

Action Items:
1. [QA] Добавить автотест: оплата с промокодом после data migration
   → Assignee: QA Lead | Deadline: Sprint 16
2. [Dev] Добавить defensive coding (null checks) в PaymentProcessor
   → Assignee: Dev Lead | Deadline: Sprint 16
3. [DevOps] Внедрить canary deployment для критических сервисов
   → Assignee: DevOps Lead | Deadline: Q2 2025
4. [QA] Добавить smoke-тест для оплаты с промокодом в post-deploy suite
   → Assignee: QA | Deadline: Sprint 16
5. [Dev] Добавить integration test для миграции данных в CI
   → Assignee: Dev | Deadline: Sprint 16
```

---

## Связь с тестированием

- Расследование production-инцидента — это **разновидность тестирования** (failure analysis)
- Навыки работы с логами и мониторингом расширяют **инструментарий QA** за пределы test cases
- Каждый инцидент порождает **новые автотесты**, предотвращающие повторение
- Post-mortem улучшает **процесс тестирования** системно
- QA, умеющий расследовать production-инциденты, **незаменим** в команде

---

## Типичные ошибки

1. **Паника вместо системного анализа** — хаотичные действия без чеклиста
2. **Пропуск мониторинга** — сразу лезть в логи, не посмотрев общую картину в Grafana
3. **Нет корреляции с деплоем** — не проверяют, был ли недавний деплой
4. **Расследование в одиночку** — не привлекают Dev и DevOps
5. **Нет верификации после fix/rollback** — «DevOps сказал, что откатил, значит всё ок»
6. **Нет post-mortem** — «починили и забыли», проблема повторяется
7. **Нет timeline** — невозможно восстановить хронологию событий
8. **Не создаются предотвращающие тесты** — те же баги попадают в production снова

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Что вы делаете первым делом при получении алерта о баге на production?
2. Что такое ELK-стек? Для чего QA использует Kibana?
3. Что такое Grafana? Какие метрики QA мониторит?
4. Чем отличается hotfix от обычного релиза?
5. Что такое post-mortem?

### 🟡 Средний уровень
6. Как определить root cause production-инцидента? Опишите свой подход
7. Как вы решаете: rollback или hotfix?
8. Какие данные QA должен собрать для bug report по production-инциденту?
9. Как верифицировать hotfix на production без риска для пользователей?
10. Как предотвратить повторение production-инцидента (какие action items)?

### 🔴 Продвинутый уровень
11. Как организовать on-call для QA? Какие инструменты и процессы нужны?
12. Как построить систему мониторинга, которая обнаруживает проблемы до пользователей?
13. Как организовать chaos engineering для предотвращения production-инцидентов?
14. Как балансировать между скоростью расследования и полнотой анализа?
15. Как построить культуру blameless post-mortem в команде?

---

## Практические задания

### Задание 1: Investigation Playbook
Создайте пошаговый playbook для расследования production-инцидента:
- Чеклист первичной оценки (первые 5 минут)
- Список инструментов и запросов
- Шаблон timeline
- Шаблон коммуникации

### Задание 2: Kibana Queries
Напишите 10 Kibana-запросов, полезных для расследования инцидентов:
- Поиск ошибок по сервису и времени
- Агрегация по типам ошибок
- Поиск по trace_id
- Корреляция с деплоями

### Задание 3: Post-Mortem
Вспомните последний production-инцидент (или придумайте гипотетический).
Напишите полный post-mortem:
- Timeline, Root Cause, Impact, Action Items

### Задание 4: Предотвращающий тест
Для бага из этого раздела (NPE при оплате с промокодом) напишите автотест на Java + REST Assured, который бы поймал этот баг до деплоя.

---

## Дополнительные ресурсы

- **Google SRE Book** (sre.google) — глава про Incident Management
- **«The Phoenix Project» by Gene Kim** — культура управления инцидентами
- **PagerDuty Incident Response Guide** (response.pagerduty.com) — практический гайд
- **Atlassian Incident Management** (atlassian.com/incident-management) — шаблоны и процессы
- **ELK Stack Documentation** (elastic.co) — работа с Elasticsearch, Logstash, Kibana
- **Grafana Documentation** (grafana.com/docs) — мониторинг и алертинг
