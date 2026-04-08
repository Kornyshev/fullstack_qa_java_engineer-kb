# Kafka для QA

## Обзор

Apache Kafka — распределённая платформа потоковой передачи сообщений, которая используется
для асинхронного обмена данными между сервисами. В микросервисной архитектуре Kafka часто является
«нервной системой» всей системы: через неё проходят события о заказах, платежах, уведомлениях,
обновлениях данных. Для QA-инженера умение тестировать Kafka — это навык, который отличает
среднего специалиста от сильного, поскольку баги в асинхронных системах — одни из самых сложных
для обнаружения и воспроизведения.

---

## Концепция Message Queue

### Синхронная vs Асинхронная коммуникация

**Синхронная (HTTP/REST):**
```
Service A → POST /api/orders → Service B
Service A ← 201 Created      ← Service B
Время ожидания: Service A блокирован, пока Service B не ответит
```

**Асинхронная (Message Queue):**
```
Service A → [отправил сообщение в очередь] → продолжает работу
                        ↓
               [Message Queue (Kafka)]
                        ↓
            Service B ← [забрал сообщение] → обработал
```

### Зачем нужны очереди сообщений

1. **Декаплинг** — сервисы не знают друг о друге напрямую
2. **Отказоустойчивость** — если consumer упал, сообщения сохраняются в очереди
3. **Масштабируемость** — можно добавить больше consumers для обработки нагрузки
4. **Пиковые нагрузки** — очередь сглаживает всплески (буферизация)
5. **Event-driven architecture** — события, а не прямые вызовы

### Kafka vs другие Message Brokers

| Аспект | Kafka | RabbitMQ | AWS SQS |
|--------|-------|----------|---------|
| Модель | Publish/Subscribe (log) | Queue/Exchange | Queue |
| Хранение | На диске (долгосрочное) | В памяти (кратковременное) | В облаке |
| Throughput | Очень высокий (миллионы msg/sec) | Средний | Средний |
| Ordering | Гарантия в рамках partition | Нет гарантии | FIFO-очереди (опция) |
| Replay | Да (можно перечитать) | Нет (сообщение удаляется) | Нет |

---

## Архитектура Kafka

### Основные компоненты

```
┌──────────────────────────────────────────────────────────┐
│                     Kafka Cluster                        │
│                                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │  Broker 1   │  │  Broker 2   │  │  Broker 3   │      │
│  │             │  │             │  │             │      │
│  │ Partition 0 │  │ Partition 1 │  │ Partition 2 │      │
│  │ (leader)    │  │ (leader)    │  │ (leader)    │      │
│  │             │  │             │  │             │      │
│  │ Partition 1 │  │ Partition 2 │  │ Partition 0 │      │
│  │ (replica)   │  │ (replica)   │  │ (replica)   │      │
│  └─────────────┘  └─────────────┘  └─────────────┘      │
│                                                          │
└──────────────────────────────────────────────────────────┘
         ▲                                    │
         │                                    ▼
   ┌───────────┐                       ┌───────────┐
   │ Producer  │                       │ Consumer  │
   │ (Service A)│                      │ (Service B)│
   └───────────┘                       └───────────┘
```

### Broker

Broker — это сервер Kafka. Кластер состоит из нескольких brokers для отказоустойчивости
и распределения нагрузки. Каждый broker хранит часть данных (partitions).

### Topic

Topic — именованный канал (лог) для сообщений. Аналогия — таблица в базе данных.

```
Topic: "order-events"
├── Partition 0: [msg0, msg3, msg6, msg9,  ...]
├── Partition 1: [msg1, msg4, msg7, msg10, ...]
└── Partition 2: [msg2, msg5, msg8, msg11, ...]
```

### Partition

Partition — единица параллелизма в Kafka. Topic разбивается на partitions, каждая хранится
на отдельном broker.

**Ключевые свойства:**
- Сообщения **упорядочены внутри partition** (но НЕ между partitions)
- Каждое сообщение имеет **offset** — порядковый номер внутри partition
- Partition — это append-only log (запись только в конец)
- Количество partitions определяет максимальный параллелизм consumers

### Consumer Group

Consumer Group — группа consumers, которые совместно читают topic. Каждая partition
назначается ровно одному consumer в группе.

```
Topic: "order-events" (3 partitions)

Consumer Group "order-processor":
├── Consumer 1 ← читает Partition 0
├── Consumer 2 ← читает Partition 1
└── Consumer 3 ← читает Partition 2

Consumer Group "analytics":
├── Consumer A ← читает Partition 0, 1
└── Consumer B ← читает Partition 2
```

**Правила:**
- Один consumer может читать несколько partitions
- Одну partition читает только один consumer в группе
- Если consumers > partitions, лишние consumers простаивают
- Разные consumer groups читают topic независимо (каждая группа получает все сообщения)

### Offset

Offset — позиция (порядковый номер) сообщения в partition. Consumer отслеживает свой offset,
чтобы знать, какие сообщения уже обработаны.

```
Partition 0: [0] [1] [2] [3] [4] [5] [6] [7] [8]
                              ↑                 ↑
                        committed offset    latest offset
                     (обработано до сюда)  (последнее сообщение)
```

**Стратегии чтения:**
- `earliest` — читать с самого начала (все сообщения)
- `latest` — читать только новые сообщения (после подключения)

---

## Producer и Consumer

### Producer (отправитель сообщений)

```java
// Пример Kafka Producer на Java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

KafkaProducer<String, String> producer = new KafkaProducer<>(props);

// Отправляем сообщение в topic "order-events"
// Ключ определяет, в какую partition попадёт сообщение
ProducerRecord<String, String> record = new ProducerRecord<>(
    "order-events",           // topic
    "order-123",              // key (используется для partitioning)
    "{\"orderId\": \"123\", \"status\": \"CREATED\"}"  // value (тело сообщения)
);

// Асинхронная отправка с callback
producer.send(record, (metadata, exception) -> {
    if (exception == null) {
        System.out.println("Отправлено в partition " + metadata.partition()
            + " с offset " + metadata.offset());
    } else {
        exception.printStackTrace();
    }
});
```

**Гарантии доставки Producer (acks):**
- `acks=0` — не ждать подтверждения (быстро, но ненадёжно)
- `acks=1` — ждать подтверждения от leader (баланс)
- `acks=all` — ждать подтверждения от всех replicas (надёжно, но медленнее)

### Consumer (получатель сообщений)

```java
// Пример Kafka Consumer на Java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("group.id", "order-processor");
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("auto.offset.reset", "earliest"); // С какого места начать читать

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Collections.singletonList("order-events"));

// Бесконечный цикл чтения сообщений
while (true) {
    // poll — забрать пачку сообщений (timeout — максимальное ожидание)
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));

    for (ConsumerRecord<String, String> record : records) {
        System.out.println("Получено сообщение: " + record.value()
            + " из partition " + record.partition()
            + " с offset " + record.offset());

        // Обработка сообщения...
    }

    // Ручной commit offset (подтверждаем обработку)
    consumer.commitSync();
}
```

---

## Тестирование Kafka

### Что тестировать

1. **Producer tests** — сообщение отправляется в правильный topic с правильным форматом
2. **Consumer tests** — сообщение обрабатывается корректно, offset commit работает
3. **Message format validation** — schema соответствует контракту (Avro, JSON Schema)
4. **Error handling** — обработка невалидных сообщений (poison pill)
5. **Ordering** — порядок обработки сообщений (в рамках partition)
6. **Idempotency** — повторная обработка одного сообщения не ломает систему
7. **End-to-end flow** — полный путь: событие → Kafka → обработка → результат

### Unit-тесты Producer

```java
// Тестируем, что сервис отправляет правильное сообщение в Kafka
@ExtendWith(MockitoExtension.class)
class OrderEventPublisherTest {

    @Mock
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;

    @InjectMocks
    private OrderEventPublisher publisher;

    @Test
    void shouldPublishOrderCreatedEvent() {
        // Подготавливаем тестовые данные
        Order order = new Order("ORD-123", BigDecimal.valueOf(1500));

        // Вызываем метод, который должен отправить событие
        publisher.publishOrderCreated(order);

        // Проверяем, что сообщение отправлено в правильный topic с правильным содержимым
        ArgumentCaptor<OrderEvent> eventCaptor = ArgumentCaptor.forClass(OrderEvent.class);
        verify(kafkaTemplate).send(eq("order-events"), eq("ORD-123"), eventCaptor.capture());

        OrderEvent event = eventCaptor.getValue();
        assertEquals("ORDER_CREATED", event.getType());
        assertEquals("ORD-123", event.getOrderId());
        assertEquals(BigDecimal.valueOf(1500), event.getAmount());
    }
}
```

### Unit-тесты Consumer

```java
// Тестируем логику обработки сообщения (без реального Kafka)
@ExtendWith(MockitoExtension.class)
class PaymentEventListenerTest {

    @Mock
    private OrderService orderService;

    @InjectMocks
    private PaymentEventListener listener;

    @Test
    void shouldUpdateOrderStatusOnPaymentSuccess() {
        // Создаём событие об успешной оплате
        PaymentEvent event = new PaymentEvent("ORD-123", "PAYMENT_SUCCESS");

        // Вызываем обработчик события
        listener.handlePaymentEvent(event);

        // Проверяем, что статус заказа обновлён
        verify(orderService).updateStatus("ORD-123", OrderStatus.PAID);
    }

    @Test
    void shouldHandlePaymentFailure() {
        PaymentEvent event = new PaymentEvent("ORD-123", "PAYMENT_FAILED");

        listener.handlePaymentEvent(event);

        verify(orderService).updateStatus("ORD-123", OrderStatus.PAYMENT_FAILED);
    }

    @Test
    void shouldHandleUnknownEventType() {
        // Тестируем обработку неизвестного типа события (poison pill)
        PaymentEvent event = new PaymentEvent("ORD-123", "UNKNOWN_TYPE");

        // Не должно выбрасывать исключение, а залогировать предупреждение
        assertDoesNotThrow(() -> listener.handlePaymentEvent(event));
        verifyNoInteractions(orderService);
    }
}
```

### Интеграционные тесты с Testcontainers

```java
// Полноценный интеграционный тест с реальным Kafka через Testcontainers
@SpringBootTest
@Testcontainers
class OrderKafkaIntegrationTest {

    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.5.0")
    );

    @DynamicPropertySource
    static void kafkaProperties(DynamicPropertyRegistry registry) {
        // Подставляем адрес тестового Kafka
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void shouldProcessOrderEventEndToEnd() throws Exception {
        // 1. Отправляем событие в Kafka
        String event = """
            {
                "orderId": "ORD-TEST-001",
                "type": "ORDER_CREATED",
                "amount": 2500.00
            }
            """;

        kafkaTemplate.send("order-events", "ORD-TEST-001", event).get();

        // 2. Ждём, пока consumer обработает сообщение
        // Используем polling вместо Thread.sleep!
        await().atMost(Duration.ofSeconds(30))
               .pollInterval(Duration.ofSeconds(1))
               .until(() -> orderRepository.findByOrderId("ORD-TEST-001").isPresent());

        // 3. Проверяем результат обработки
        Order order = orderRepository.findByOrderId("ORD-TEST-001").orElseThrow();
        assertEquals(OrderStatus.CREATED, order.getStatus());
        assertEquals(new BigDecimal("2500.00"), order.getAmount());
    }

    @Test
    void shouldHandleInvalidMessage() throws Exception {
        // Отправляем невалидное сообщение (невалидный JSON)
        kafkaTemplate.send("order-events", "bad-key", "not a json").get();

        // Проверяем, что система не упала и продолжает обрабатывать следующие сообщения
        String validEvent = """
            {"orderId": "ORD-AFTER-BAD", "type": "ORDER_CREATED", "amount": 100.00}
            """;
        kafkaTemplate.send("order-events", "ORD-AFTER-BAD", validEvent).get();

        await().atMost(Duration.ofSeconds(30))
               .until(() -> orderRepository.findByOrderId("ORD-AFTER-BAD").isPresent());
    }
}
```

### Тестирование Message Format (Schema Validation)

```java
// Тестирование соответствия сообщения JSON Schema
@Test
void shouldProduceMessageMatchingSchema() throws Exception {
    // Загружаем JSON Schema
    JsonSchemaFactory factory = JsonSchemaFactory.getInstance(SpecVersion.VersionFlag.V7);
    JsonSchema schema = factory.getSchema(
        getClass().getResourceAsStream("/schemas/order-event.json")
    );

    // Создаём событие через production-код
    OrderEvent event = orderEventPublisher.createEvent(testOrder);
    String json = objectMapper.writeValueAsString(event);

    // Валидируем против схемы
    Set<ValidationMessage> errors = schema.validate(objectMapper.readTree(json));
    assertTrue(errors.isEmpty(),
        "Сообщение не соответствует схеме: " + errors);
}
```

---

## Docker-compose для тестирования Kafka

```yaml
# docker-compose.kafka-test.yml — среда для ручного тестирования Kafka
version: '3.8'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    depends_on:
      - kafka
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
```

**Полезные команды для ручного тестирования:**
```bash
# Создать topic
docker exec kafka kafka-topics --create \
  --topic order-events --partitions 3 --replication-factor 1 \
  --bootstrap-server localhost:9092

# Отправить сообщение (producer)
docker exec -it kafka kafka-console-producer \
  --topic order-events --bootstrap-server localhost:9092
> {"orderId": "ORD-001", "type": "ORDER_CREATED"}

# Прочитать сообщения (consumer)
docker exec kafka kafka-console-consumer \
  --topic order-events --from-beginning \
  --bootstrap-server localhost:9092

# Посмотреть список topics
docker exec kafka kafka-topics --list --bootstrap-server localhost:9092

# Посмотреть consumer group offsets
docker exec kafka kafka-consumer-groups \
  --describe --group order-processor \
  --bootstrap-server localhost:9092
```

---

## Связь с тестированием

| Аспект Kafka | Что тестировать | Как тестировать |
|-------------|-----------------|-----------------|
| Producer | Формат сообщения, topic, key | Unit tests + MockKafka |
| Consumer | Логика обработки, error handling | Unit tests + Testcontainers |
| Ordering | Порядок обработки в partition | Integration tests с конкретным key |
| Idempotency | Повторная обработка | Отправить одно сообщение дважды |
| Dead Letter Queue | Обработка ошибочных сообщений | Отправить невалидное сообщение |
| Performance | Throughput, latency | Нагрузочные тесты |
| Schema | Совместимость схем | Schema Registry + contract tests |

---

## Типичные ошибки

1. **`Thread.sleep` вместо polling** — тесты становятся flaky или неоправданно медленными
2. **Не тестируют poison pill** — одно невалидное сообщение блокирует весь consumer
3. **Не тестируют idempotency** — повторная обработка сообщения создаёт дубли
4. **Игнорируют ordering** — тесты не учитывают, что порядок гарантирован только внутри partition
5. **Не очищают topic между тестами** — тесты влияют друг на друга
6. **Используют shared Kafka** — тестовые данные смешиваются с production
7. **Не тестируют Dead Letter Queue** — необработанные сообщения теряются бесследно

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Что такое Kafka и зачем она нужна?
2. В чём разница между синхронной и асинхронной коммуникацией?
3. Что такое topic, partition и offset?
4. Что такое consumer group?

### 🟡 Средний уровень
5. Как гарантируется порядок сообщений в Kafka?
6. Что произойдёт, если consumer упадёт? Как Kafka обеспечивает отказоустойчивость?
7. Как тестировать Kafka producer и consumer?
8. Что такое Dead Letter Queue и зачем она нужна?
9. Как настроить Kafka в тестах с Testcontainers?
10. В чём разница между `auto.offset.reset=earliest` и `latest`?

### 🔴 Продвинутый уровень
11. Как тестировать exactly-once semantics в Kafka?
12. Что происходит при rebalancing consumer group и как это влияет на тесты?
13. Как тестировать Kafka Streams приложения?
14. Как обеспечить schema evolution (совместимость схем) и как это тестировать?
15. Как протестировать производительность Kafka consumer при высокой нагрузке?

---

## Практические задания

### Задание 1: Поднять Kafka локально
Используя docker-compose из этого раздела:
1. Поднимите Kafka и Kafka UI
2. Создайте topic `test-events` с 3 partitions
3. Отправьте 10 сообщений через console-producer
4. Прочитайте их через console-consumer
5. Откройте Kafka UI и изучите интерфейс

### Задание 2: Написать интеграционный тест
Напишите Spring Boot тест с Testcontainers:
1. Поднимите Kafka container
2. Отправьте JSON-сообщение в topic
3. Проверьте, что consumer обработал сообщение и записал результат в БД

### Задание 3: Тестирование обработки ошибок
1. Отправьте невалидное сообщение (битый JSON) в topic
2. Убедитесь, что consumer не упал
3. Отправьте валидное сообщение после невалидного
4. Проверьте, что валидное сообщение обработано корректно

### Задание 4: Тестирование ordering
1. Отправьте 5 сообщений с одинаковым key (в одну partition)
2. Проверьте, что consumer получает их в правильном порядке
3. Отправьте 5 сообщений с разными keys
4. Проверьте, что они могут обработаться в произвольном порядке

---

## Дополнительные ресурсы

- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Confluent Kafka Tutorials](https://developer.confluent.io/tutorials/)
- [Testcontainers: Kafka Module](https://testcontainers.com/modules/kafka/)
- [Kafka UI](https://github.com/provectus/kafka-ui)
- [Designing Event-Driven Systems (Ben Stopford, бесплатная книга)](https://www.confluent.io/designing-event-driven-systems/)
