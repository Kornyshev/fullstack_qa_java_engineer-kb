# Тестирование Kafka

## Обзор

Apache Kafka — де-факто стандарт для асинхронного обмена сообщениями в микросервисной архитектуре.
Сервисы публикуют события (events) в топики Kafka, другие сервисы подписываются и обрабатывают их.
Это порождает целый класс задач тестирования, с которыми QA-инженер сталкивается ежедневно:
корректность формата сообщений, обработка ошибок, идемпотентность, управление offset'ами,
работа с Dead Letter Queue (DLQ) и многое другое.

В отличие от тестирования синхронных REST API, тестирование Kafka требует особого подхода:
сообщения асинхронны, порядок обработки не гарантирован (между partition'ами),
а ошибки могут проявиться с задержкой.

---

## Архитектура Kafka: что нужно знать QA

```
Producer → [Topic: order-events] → Consumer
              ├── Partition 0
              ├── Partition 1        Каждый partition — упорядоченная последовательность
              └── Partition 2        сообщений (offset 0, 1, 2, ...)
```

### Ключевые концепции для тестирования

- **Producer** — отправляет сообщения в топик. Отвечает за формат, ключ партицирования, заголовки.
- **Consumer** — читает сообщения. Отвечает за обработку, коммит offset'ов, обработку ошибок.
- **Partition Key** — определяет, в какой partition попадёт сообщение. Сообщения с одним ключом
  всегда попадают в один partition (гарантия порядка).
- **Consumer Group** — группа потребителей, каждый partition обрабатывается одним consumer'ом в группе.
- **Offset** — позиция сообщения в partition. Consumer отслеживает, до какого offset он дочитал.
- **DLQ (Dead Letter Queue)** — топик для сообщений, которые не удалось обработать.

---

## Тестирование Producer

### Что проверяем у Producer

1. **Формат сообщения** — структура JSON/Avro соответствует схеме.
2. **Заголовки (headers)** — наличие обязательных заголовков (correlation-id, content-type, timestamp).
3. **Partition key** — правильный ключ для обеспечения порядка (например, orderId).
4. **Топик** — сообщение отправляется в правильный топик.
5. **Обработка ошибок отправки** — что происходит, если Kafka недоступна.

### Пример: unit-тест для Producer

```java
// Producer, отправляющий событие о создании заказа
@Service
@RequiredArgsConstructor
public class OrderEventProducer {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void sendOrderCreated(Order order) {
        var event = new OrderEvent(
                order.getId(),
                "ORDER_CREATED",
                order.getCustomerId(),
                order.getTotal(),
                Instant.now()
        );

        // Ключ партицирования — orderId (гарантирует порядок событий одного заказа)
        kafkaTemplate.send(
                new ProducerRecord<>(
                        "order-events",       // топик
                        order.getId(),         // ключ
                        event                  // значение
                )
        ).whenComplete((result, ex) -> {
            if (ex != null) {
                log.error("Ошибка отправки события для заказа {}: {}", order.getId(), ex.getMessage());
            }
        });
    }
}
```

```java
@ExtendWith(MockitoExtension.class)
class OrderEventProducerTest {

    @Mock
    KafkaTemplate<String, OrderEvent> kafkaTemplate;

    @InjectMocks
    OrderEventProducer producer;

    @Captor
    ArgumentCaptor<ProducerRecord<String, OrderEvent>> recordCaptor;

    @Test
    void shouldSendOrderCreatedEventWithCorrectPartitionKey() {
        // Подготовка: заказ с известными данными
        var order = Order.builder()
                .id("order-123")
                .customerId("customer-456")
                .total(BigDecimal.valueOf(5000))
                .build();

        when(kafkaTemplate.send(any(ProducerRecord.class)))
                .thenReturn(mock(CompletableFuture.class));

        // Действие
        producer.sendOrderCreated(order);

        // Проверка
        verify(kafkaTemplate).send(recordCaptor.capture());
        var record = recordCaptor.getValue();

        // Проверяем топик
        assertThat(record.topic()).isEqualTo("order-events");
        // Проверяем ключ партицирования
        assertThat(record.key()).isEqualTo("order-123");
        // Проверяем содержимое события
        assertThat(record.value().getEventType()).isEqualTo("ORDER_CREATED");
        assertThat(record.value().getCustomerId()).isEqualTo("customer-456");
        assertThat(record.value().getAmount()).isEqualByComparingTo(BigDecimal.valueOf(5000));
        // Проверяем, что timestamp заполнен
        assertThat(record.value().getTimestamp()).isNotNull();
    }

    @Test
    void shouldUseOrderIdAsPartitionKey() {
        // Проверяем, что все события одного заказа отправляются с одним ключом
        var order1 = Order.builder().id("order-1").customerId("c-1").total(BigDecimal.TEN).build();
        var order2 = Order.builder().id("order-1").customerId("c-1").total(BigDecimal.TEN).build();

        when(kafkaTemplate.send(any(ProducerRecord.class)))
                .thenReturn(mock(CompletableFuture.class));

        producer.sendOrderCreated(order1);
        producer.sendOrderCreated(order2);

        verify(kafkaTemplate, times(2)).send(recordCaptor.capture());
        var keys = recordCaptor.getAllValues().stream()
                .map(ProducerRecord::key)
                .toList();

        // Оба события должны иметь одинаковый ключ
        assertThat(keys).containsExactly("order-1", "order-1");
    }
}
```

---

## Тестирование Consumer

### Что проверяем у Consumer

1. **Логика обработки** — сообщение корректно десериализуется и обрабатывается.
2. **Обработка ошибок** — что происходит при невалидном сообщении, ошибке БД, таймауте.
3. **Идемпотентность** — повторная обработка одного и того же сообщения не создаёт дубликатов.
4. **Управление offset'ами** — offset коммитится только после успешной обработки.
5. **DLQ** — невалидные/необрабатываемые сообщения отправляются в Dead Letter Queue.

### Пример: unit-тест для Consumer

```java
@Service
@RequiredArgsConstructor
public class PaymentEventConsumer {

    private final OrderRepository orderRepository;
    private final OrderEventProducer eventProducer;

    @KafkaListener(topics = "payment-events", groupId = "order-service")
    public void handlePaymentEvent(PaymentEvent event) {
        var order = orderRepository.findById(event.getOrderId())
                .orElseThrow(() -> new OrderNotFoundException(event.getOrderId()));

        switch (event.getStatus()) {
            case "PAYMENT_APPROVED" -> {
                order.setStatus(OrderStatus.CONFIRMED);
                orderRepository.save(order);
                eventProducer.sendOrderConfirmed(order);
            }
            case "PAYMENT_REJECTED" -> {
                order.setStatus(OrderStatus.CANCELLED);
                order.setCancellationReason("Оплата отклонена");
                orderRepository.save(order);
            }
            default -> log.warn("Неизвестный статус оплаты: {}", event.getStatus());
        }
    }
}
```

```java
@ExtendWith(MockitoExtension.class)
class PaymentEventConsumerTest {

    @Mock
    OrderRepository orderRepository;

    @Mock
    OrderEventProducer eventProducer;

    @InjectMocks
    PaymentEventConsumer consumer;

    @Test
    void shouldConfirmOrderOnPaymentApproved() {
        // Подготовка
        var order = new Order("order-1", OrderStatus.PENDING);
        when(orderRepository.findById("order-1")).thenReturn(Optional.of(order));

        var event = new PaymentEvent("order-1", "PAYMENT_APPROVED", BigDecimal.valueOf(5000));

        // Действие
        consumer.handlePaymentEvent(event);

        // Проверка: статус заказа обновлён
        assertThat(order.getStatus()).isEqualTo(OrderStatus.CONFIRMED);
        verify(orderRepository).save(order);
        // Проверяем, что отправлено подтверждение
        verify(eventProducer).sendOrderConfirmed(order);
    }

    @Test
    void shouldCancelOrderOnPaymentRejected() {
        var order = new Order("order-1", OrderStatus.PENDING);
        when(orderRepository.findById("order-1")).thenReturn(Optional.of(order));

        var event = new PaymentEvent("order-1", "PAYMENT_REJECTED", BigDecimal.valueOf(5000));

        consumer.handlePaymentEvent(event);

        assertThat(order.getStatus()).isEqualTo(OrderStatus.CANCELLED);
        assertThat(order.getCancellationReason()).isEqualTo("Оплата отклонена");
        verify(orderRepository).save(order);
        // Не должно быть отправлено подтверждение
        verify(eventProducer, never()).sendOrderConfirmed(any());
    }

    @Test
    void shouldThrowExceptionWhenOrderNotFound() {
        when(orderRepository.findById("order-999")).thenReturn(Optional.empty());
        var event = new PaymentEvent("order-999", "PAYMENT_APPROVED", BigDecimal.valueOf(100));

        assertThatThrownBy(() -> consumer.handlePaymentEvent(event))
                .isInstanceOf(OrderNotFoundException.class);
    }
}
```

---

## Kafka Testcontainers

### Настройка

Testcontainers позволяет запустить реальный Kafka-брокер в Docker для интеграционных тестов.

```java
@SpringBootTest
@Testcontainers
class KafkaIntegrationTest {

    @Container
    static KafkaContainer kafka = new KafkaContainer(
            DockerImageName.parse("confluentinc/cp-kafka:7.5.0")
    );

    @DynamicPropertySource
    static void kafkaProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }

    @Autowired
    KafkaTemplate<String, String> kafkaTemplate;

    @Autowired
    KafkaListenerEndpointRegistry kafkaListenerRegistry;

    @Test
    void shouldProduceAndConsumeMessage() throws Exception {
        // Отправляем сообщение
        kafkaTemplate.send("test-topic", "key-1", "{\"orderId\": \"123\"}").get();

        // Ждём обработки (используем Awaitility)
        await().atMost(Duration.ofSeconds(10)).untilAsserted(() -> {
            // Проверяем результат обработки (например, запись в БД)
            var order = orderRepository.findById("123");
            assertThat(order).isPresent();
        });
    }
}
```

### Конфигурация для тестов с несколькими партициями

```java
@Container
static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.5.0"))
        .withEnv("KAFKA_NUM_PARTITIONS", "3")              // 3 партиции по умолчанию
        .withEnv("KAFKA_DEFAULT_REPLICATION_FACTOR", "1");  // 1 реплика (достаточно для тестов)
```

### Создание топика программно

```java
@BeforeAll
static void createTopics() {
    try (var admin = AdminClient.create(
            Map.of(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, kafka.getBootstrapServers()))) {
        var topic = new NewTopic("order-events", 3, (short) 1); // 3 партиции, 1 реплика
        admin.createTopics(List.of(topic)).all().get(10, TimeUnit.SECONDS);
    }
}
```

---

## Embedded Kafka (Spring)

### Альтернатива Testcontainers

Spring Kafka предоставляет `@EmbeddedKafka` — встроенный Kafka-брокер, который запускается
в том же JVM-процессе, что и тест. Не требует Docker.

```java
@SpringBootTest
@EmbeddedKafka(
        partitions = 3,
        topics = {"order-events", "payment-events", "order-events-dlq"},
        brokerProperties = {
                "listeners=PLAINTEXT://localhost:0",  // случайный порт
                "auto.create.topics.enable=false"
        }
)
class OrderServiceEmbeddedKafkaTest {

    @Autowired
    EmbeddedKafkaBroker embeddedKafka;

    @Autowired
    KafkaTemplate<String, OrderEvent> kafkaTemplate;

    @Test
    void shouldProcessOrderEvent() {
        // Отправляем событие
        var event = new OrderEvent("order-1", "ORDER_CREATED", "customer-1",
                BigDecimal.valueOf(1000), Instant.now());
        kafkaTemplate.send("order-events", "order-1", event);

        // Проверяем обработку
        await().atMost(Duration.ofSeconds(10)).untilAsserted(() -> {
            var order = orderRepository.findById("order-1");
            assertThat(order).isPresent();
            assertThat(order.get().getStatus()).isEqualTo(OrderStatus.CREATED);
        });
    }
}
```

### Testcontainers vs Embedded Kafka

| Критерий             | Testcontainers                  | Embedded Kafka                    |
|----------------------|---------------------------------|-----------------------------------|
| Требует Docker       | Да                              | Нет                               |
| Реалистичность       | Высокая (реальный Kafka)        | Средняя (упрощённая реализация)   |
| Скорость запуска     | 10-30 секунд                    | 3-5 секунд                        |
| Совместимость с CI   | Нужен Docker в CI               | Работает везде                    |
| Kafka Connect, Schema Registry | Поддерживается        | Не поддерживается                 |
| Рекомендация         | Интеграционные/E2E тесты        | Unit/Component тесты              |

---

## Тестирование схем сообщений (Avro, JSON Schema)

### Зачем нужны схемы

Схема определяет структуру сообщения и обеспечивает совместимость между Producer и Consumer.
Без схемы Producer может отправить сообщение в произвольном формате, и Consumer сломается.

### JSON Schema: валидация формата сообщений

```java
// Тест: проверяем, что событие соответствует JSON Schema
@Test
void orderEventShouldMatchJsonSchema() {
    var event = new OrderEvent("order-1", "ORDER_CREATED", "customer-1",
            BigDecimal.valueOf(5000), Instant.now());

    // Сериализуем в JSON
    var json = objectMapper.writeValueAsString(event);

    // Валидируем по JSON Schema
    var schema = JsonSchemaFactory.getInstance(SpecVersion.VersionFlag.V7)
            .getSchema(getClass().getResourceAsStream("/schemas/order-event.json"));

    var errors = schema.validate(objectMapper.readTree(json));
    assertThat(errors).isEmpty();
}
```

### Avro: тестирование совместимости схем

```java
// Тест: проверяем backward-совместимость новой версии схемы
@Test
void newSchemaShouldBeBackwardCompatible() {
    // Старая схема (v1)
    var oldSchema = new Schema.Parser().parse(
            getClass().getResourceAsStream("/avro/order-event-v1.avsc"));
    // Новая схема (v2) — добавлено поле discount с default
    var newSchema = new Schema.Parser().parse(
            getClass().getResourceAsStream("/avro/order-event-v2.avsc"));

    // Проверяем backward-совместимость: новый consumer может читать старые сообщения
    var compatibility = SchemaCompatibility.checkReaderWriterCompatibility(
            newSchema,  // reader (новый consumer)
            oldSchema   // writer (старый producer)
    );

    assertThat(compatibility.getType())
            .isEqualTo(SchemaCompatibility.SchemaCompatibilityType.COMPATIBLE);
}
```

### Тестирование Schema Registry

```java
@SpringBootTest
@Testcontainers
class SchemaRegistryTest {

    @Container
    static KafkaContainer kafka = new KafkaContainer(
            DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));

    @Container
    static GenericContainer<?> schemaRegistry = new GenericContainer<>(
            DockerImageName.parse("confluentinc/cp-schema-registry:7.5.0"))
            .withExposedPorts(8081)
            .withEnv("SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS", kafka.getBootstrapServers())
            .withEnv("SCHEMA_REGISTRY_HOST_NAME", "schema-registry")
            .dependsOn(kafka);

    @Test
    void shouldRegisterAndValidateSchema() {
        var client = new CachedSchemaRegistryClient(
                "http://localhost:" + schemaRegistry.getMappedPort(8081), 100);

        // Регистрируем схему
        var schema = new Schema.Parser().parse("""
                {"type": "record", "name": "OrderEvent", "fields": [
                    {"name": "orderId", "type": "string"},
                    {"name": "eventType", "type": "string"},
                    {"name": "amount", "type": "double"}
                ]}
                """);

        int schemaId = client.register("order-events-value", new AvroSchema(schema));
        assertThat(schemaId).isPositive();
    }
}
```

---

## Тестирование идемпотентности

### Почему идемпотентность критична

В Kafka сообщение может быть доставлено повторно (at-least-once delivery).
Consumer должен обрабатывать дубли корректно.

```java
@Test
void shouldBeIdempotent_processSameMessageTwice() {
    var event = new PaymentEvent("order-1", "PAYMENT_APPROVED", BigDecimal.valueOf(5000));

    // Первая обработка
    consumer.handlePaymentEvent(event);

    // Повторная обработка того же события (дубликат)
    consumer.handlePaymentEvent(event);

    // В БД должен быть только один заказ со статусом CONFIRMED
    var orders = orderRepository.findAllByOrderId("order-1");
    assertThat(orders).hasSize(1);
    assertThat(orders.get(0).getStatus()).isEqualTo(OrderStatus.CONFIRMED);
}
```

### Стратегии обеспечения идемпотентности

```java
// Подход 1: Проверка по уникальному идентификатору события
@KafkaListener(topics = "payment-events")
public void handlePaymentEvent(PaymentEvent event,
                                @Header(KafkaHeaders.RECEIVED_KEY) String key,
                                @Header("eventId") String eventId) {
    // Проверяем, было ли событие уже обработано
    if (processedEventRepository.existsByEventId(eventId)) {
        log.info("Событие {} уже обработано, пропускаем", eventId);
        return;
    }

    // Обрабатываем событие
    processPayment(event);

    // Сохраняем идентификатор обработанного события
    processedEventRepository.save(new ProcessedEvent(eventId, Instant.now()));
}
```

```java
// Тест идемпотентности с реальной БД
@SpringBootTest
@Testcontainers
class IdempotencyIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @Container
    static KafkaContainer kafka = new KafkaContainer(
            DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));

    @Autowired
    KafkaTemplate<String, String> kafkaTemplate;

    @Autowired
    OrderRepository orderRepository;

    @Test
    void shouldNotCreateDuplicateOrders() throws Exception {
        var eventJson = """
                {"orderId": "order-1", "status": "PAYMENT_APPROVED", "amount": 5000}
                """;

        // Отправляем одно и то же сообщение дважды с одинаковым eventId
        var headers = new RecordHeaders();
        headers.add("eventId", "evt-123".getBytes());

        kafkaTemplate.send(new ProducerRecord<>("payment-events", null, "order-1", eventJson, headers)).get();
        kafkaTemplate.send(new ProducerRecord<>("payment-events", null, "order-1", eventJson, headers)).get();

        // Ждём обработки обоих сообщений
        await().atMost(Duration.ofSeconds(15)).untilAsserted(() -> {
            var orders = orderRepository.findAllByStatus(OrderStatus.CONFIRMED);
            // Должен быть ровно один подтверждённый заказ, не два
            assertThat(orders).hasSize(1);
        });
    }
}
```

---

## Тестирование Dead Letter Queue (DLQ)

### Что такое DLQ

DLQ — специальный топик, куда отправляются сообщения, которые не удалось обработать
после заданного числа попыток. Это критический механизм отказоустойчивости.

### Конфигурация DLQ в Spring Kafka

```java
@Bean
public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory(
        ConsumerFactory<String, String> consumerFactory,
        KafkaTemplate<String, String> kafkaTemplate) {

    var factory = new ConcurrentKafkaListenerContainerFactory<String, String>();
    factory.setConsumerFactory(consumerFactory);

    // Настройка retry: 3 попытки с интервалом 1 секунда
    factory.setCommonErrorHandler(new DefaultErrorHandler(
            new DeadLetterPublishingRecoverer(kafkaTemplate),  // Отправка в DLQ
            new FixedBackOff(1000L, 3)                         // 3 retry, 1 сек интервал
    ));

    return factory;
}
```

### Тест: проверка отправки в DLQ

```java
@SpringBootTest
@EmbeddedKafka(topics = {"order-events", "order-events.DLT"})
class DlqTest {

    @Autowired
    KafkaTemplate<String, String> kafkaTemplate;

    // Consumer для чтения из DLQ
    @Autowired
    ConsumerFactory<String, String> consumerFactory;

    @Test
    void shouldSendToDlqAfterRetries() throws Exception {
        // Отправляем невалидное сообщение, которое вызовет ошибку при обработке
        kafkaTemplate.send("order-events", "bad-key", "invalid-json{{{").get();

        // Читаем из DLQ-топика
        try (var consumer = consumerFactory.createConsumer("dlq-test-group", "dlq-test")) {
            consumer.subscribe(List.of("order-events.DLT"));

            await().atMost(Duration.ofSeconds(30)).untilAsserted(() -> {
                var records = consumer.poll(Duration.ofMillis(500));
                assertThat(records.count()).isGreaterThan(0);

                var dlqRecord = records.iterator().next();
                assertThat(dlqRecord.value()).isEqualTo("invalid-json{{{");

                // Проверяем заголовки DLQ (причина ошибки)
                var exceptionHeader = new String(
                        dlqRecord.headers().lastHeader("kafka_dlt-exception-message").value());
                assertThat(exceptionHeader).contains("Deserialization");
            });
        }
    }
}
```

### Что проверяем при тестировании DLQ

- Сообщение попадает в DLQ после исчерпания retry-попыток.
- Оригинальное сообщение сохраняется в DLQ без изменений.
- В заголовках DLQ-сообщения содержится информация об ошибке.
- Обработка продолжается для следующих сообщений (DLQ не блокирует consumer).
- Количество retry-попыток соответствует конфигурации.

---

## Тестирование Offset Management

### Сценарии для тестирования

```java
@Test
void shouldCommitOffsetOnlyAfterSuccessfulProcessing() {
    // 1. Отправляем 3 сообщения
    kafkaTemplate.send("order-events", "key-1", event1Json).get();
    kafkaTemplate.send("order-events", "key-2", event2Json).get();
    kafkaTemplate.send("order-events", "key-3", event3Json).get();

    // 2. Настраиваем consumer так, чтобы обработка 2-го сообщения упала
    // (через мок сервиса, который кидает исключение для key-2)
    doThrow(new RuntimeException("Ошибка БД"))
            .when(orderService).processOrder(argThat(e -> e.getKey().equals("key-2")));

    // 3. Ждём обработки
    await().atMost(Duration.ofSeconds(15)).untilAsserted(() -> {
        // Первое сообщение обработано
        assertThat(orderRepository.findById("key-1")).isPresent();
        // Второе — в DLQ (после retry)
        // Третье — обработано
        assertThat(orderRepository.findById("key-3")).isPresent();
    });
}
```

---

## Полный интеграционный тест: Producer + Consumer + DLQ

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class FullKafkaFlowTest {

    @Container
    static KafkaContainer kafka = new KafkaContainer(
            DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
            .withDatabaseName("test");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry registry) {
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    TestRestTemplate restTemplate;

    @Autowired
    OrderRepository orderRepository;

    @Test
    void fullFlow_createOrder_processPayment_confirmOrder() {
        // 1. Создаём заказ через REST API (это отправит событие в Kafka)
        var response = restTemplate.postForEntity("/api/orders",
                new CreateOrderRequest("item-1", 2, BigDecimal.valueOf(5000)),
                OrderResponse.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        var orderId = response.getBody().getOrderId();

        // 2. Имитируем ответ от payment-service (отправляем событие в Kafka)
        kafkaTemplate.send("payment-events", orderId,
                "{\"orderId\": \"" + orderId + "\", \"status\": \"PAYMENT_APPROVED\"}");

        // 3. Ждём, пока order-service обработает событие и обновит статус
        await().atMost(Duration.ofSeconds(15)).untilAsserted(() -> {
            var order = orderRepository.findById(orderId).orElseThrow();
            assertThat(order.getStatus()).isEqualTo(OrderStatus.CONFIRMED);
        });
    }
}
```

---

## Связь с тестированием

QA-инженер при тестировании Kafka должен:
- **Проверять формат сообщений** — schema validation как часть CI.
- **Тестировать обработку ошибок** — невалидные сообщения, таймауты, недоступность БД.
- **Проверять идемпотентность** — повторная обработка не должна создавать дубли.
- **Тестировать DLQ** — «ядовитые» сообщения не должны блокировать обработку.
- **Тестировать порядок обработки** — если бизнес-логика зависит от порядка событий.
- **Мониторить consumer lag** — отставание consumer'а от producer'а.

---

## Типичные ошибки

1. **Не тестируют десериализацию** — producer отправляет корректный JSON, но consumer
   не может его десериализовать из-за несовпадения полей или типов.

2. **Забывают про DLQ** — «ядовитое» сообщение блокирует partition,
   и все последующие сообщения не обрабатываются.

3. **Не тестируют идемпотентность** — повторная обработка создаёт дубликаты в БД.

4. **Hardcoded таймауты в тестах** — `Thread.sleep(5000)` вместо Awaitility.
   Результат: медленные или flaky-тесты.

5. **Не проверяют partition key** — события одного заказа попадают в разные partition'ы,
   нарушается порядок обработки.

6. **Тестируют Producer и Consumer отдельно, но не вместе** — нет интеграционного теста
   всего потока (API → Producer → Kafka → Consumer → БД).

7. **Не учитывают consumer group rebalancing** — при добавлении/удалении consumer'ов
   происходит перераспределение partition'ов, которое может вызвать повторную обработку.

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Что такое Kafka и зачем она используется в микросервисах?
2. Что такое Producer и Consumer в Kafka?
3. Что такое topic и partition?
4. Как написать тест для Kafka Producer с помощью Mockito?
5. Что такое Embedded Kafka и для чего она используется в тестах?

### 🟡 Средний уровень
6. Как настроить Kafka Testcontainers для интеграционных тестов?
7. Что такое Dead Letter Queue (DLQ) и как протестировать его работу?
8. Как тестировать идемпотентность Consumer'а?
9. Зачем нужен partition key и как протестировать правильность его выбора?
10. Чем Embedded Kafka отличается от Kafka Testcontainers?
11. Как тестировать Avro-схемы на backward-совместимость?

### 🔴 Продвинутый уровень
12. Как тестировать exactly-once semantics в Kafka?
13. Как протестировать поведение consumer'а при rebalancing?
14. Как тестировать систему с Kafka Streams?
15. Как организовать тестирование Schema Registry в CI/CD?
16. Как протестировать, что consumer не теряет сообщения при перезапуске?
17. Как тестировать порядок обработки сообщений при нескольких partition'ах?

---

## Практические задания

### Задание 1: Producer тест
Напишите unit-тест для `NotificationEventProducer`, который отправляет событие `EMAIL_SENT`
в топик `notification-events`. Проверьте: формат сообщения, заголовки (correlationId, timestamp),
partition key (userId).

### Задание 2: Consumer с DLQ
Реализуйте и протестируйте Consumer, который:
- Читает из топика `order-events`.
- Сохраняет заказ в PostgreSQL.
- При ошибке десериализации отправляет в DLQ.
- При ошибке БД делает 3 retry с интервалом 1 секунда.
Используйте Testcontainers для Kafka и PostgreSQL.

### Задание 3: Тестирование идемпотентности
Напишите интеграционный тест, который отправляет одно и то же событие 5 раз
и проверяет, что в БД создана только одна запись.

### Задание 4: Schema evolution
Создайте две версии Avro-схемы для `OrderEvent` (v1 и v2, где v2 добавляет
необязательное поле `discount`). Напишите тест, проверяющий backward-совместимость.

---

## Дополнительные ресурсы

- **Документация**: [Spring for Apache Kafka](https://docs.spring.io/spring-kafka/reference/)
- **Документация**: [Testcontainers Kafka Module](https://testcontainers.com/modules/kafka/)
- **Статья**: "Testing Kafka Applications" (Confluent Blog)
- **Инструмент**: [Kafdrop](https://github.com/obsidiandynamics/kafdrop) — UI для просмотра Kafka
- **Книга**: "Kafka: The Definitive Guide" (O'Reilly) — глава о тестировании
- **Библиотека**: [Awaitility](https://github.com/awaitility/awaitility) — для асинхронных assertions
- **Видео**: "Testing Event-Driven Microservices" (Confluent, YouTube)
