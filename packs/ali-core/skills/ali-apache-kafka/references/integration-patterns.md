# ali-apache-kafka - Integration Patterns Reference

## Overview

Architectural integration patterns using Apache Kafka for event-driven systems, CDC, CQRS, and distributed transactions.

---

## Event-Driven Architecture

### Pattern Overview

```
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│  Collibra    │─────▶│    Kafka     │─────▶│   Iceberg    │
│    Edge      │      │              │      │   Bronze     │
└──────────────┘      └──────────────┘      └──────────────┘
                             │
                             ├────▶ Analytics Service
                             │
                             └────▶ Notification Service
```

### Characteristics

- **Loose coupling:** Services don't know about each other
- **Asynchronous:** Non-blocking communication
- **Scalable:** Add consumers without affecting producers
- **Replay:** Consumers can re-process historical events

### Implementation Example

**Event schema:**
```json
{
  "event_type": "AssetCreated",
  "event_id": "uuid-123",
  "timestamp": "2026-02-15T10:30:00Z",
  "source": "collibra-edge",
  "data": {
    "asset_id": "a-12345",
    "asset_type": "Table",
    "domain": "Finance",
    "steward": "john.doe@example.com"
  }
}
```

**Producer (Collibra Edge):**
```java
ProducerRecord<String, Event> record = new ProducerRecord<>(
    "governance.events",
    event.getAssetId(),  // Partition by asset ID
    event
);
producer.send(record);
```

**Consumer (Analytics Service):**
```java
consumer.subscribe(Arrays.asList("governance.events"));

while (true) {
    ConsumerRecords<String, Event> records = consumer.poll(Duration.ofMillis(100));

    for (ConsumerRecord<String, Event> record : records) {
        Event event = record.value();

        if ("AssetCreated".equals(event.getEventType())) {
            analytics.trackAssetCreation(event.getData());
        }
    }

    consumer.commitAsync();
}
```

---

## Change Data Capture (CDC)

### Pattern Overview

```
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│  SAP MDG     │      │   Debezium   │      │    Kafka     │
│  (Oracle)    │─────▶│  Connector   │─────▶│              │
└──────────────┘      └──────────────┘      └──────────────┘
                                                    │
                                                    ▼
                                            ┌──────────────┐
                                            │  Medallion   │
                                            │  Architecture│
                                            │  (Bronze)    │
                                            └──────────────┘
```

### Benefits

- **Near-real-time:** Capture changes as they happen
- **No code changes:** Database-agnostic (works with existing apps)
- **Full history:** Captures inserts, updates, deletes
- **Schema evolution:** Tracks DDL changes

### Debezium Event Structure

**Insert event:**
```json
{
  "before": null,
  "after": {
    "material_number": "MAT-12345",
    "description": "High-strength steel",
    "base_unit": "KG"
  },
  "source": {
    "version": "2.1.0",
    "connector": "oracle",
    "name": "sap.mdg",
    "table": "MDG.MATERIAL",
    "ts_ms": 1673019234567
  },
  "op": "c",  // c=create, u=update, d=delete, r=read (snapshot)
  "ts_ms": 1673019234567
}
```

**Update event:**
```json
{
  "before": {
    "material_number": "MAT-12345",
    "description": "High-strength steel",
    "base_unit": "KG"
  },
  "after": {
    "material_number": "MAT-12345",
    "description": "Ultra high-strength steel",  // Changed
    "base_unit": "KG"
  },
  "op": "u"
}
```

**Delete event (tombstone):**
```json
{
  "before": {
    "material_number": "MAT-12345",
    "description": "Ultra high-strength steel",
    "base_unit": "KG"
  },
  "after": null,
  "op": "d"
}
// Followed by null value (tombstone for log compaction)
```

### Processing CDC Events

```java
for (ConsumerRecord<String, GenericRecord> record : records) {
    GenericRecord value = record.value();
    String op = value.get("op").toString();

    switch (op) {
        case "c":
        case "r":  // Snapshot reads treated as inserts
            GenericRecord after = (GenericRecord) value.get("after");
            insertIntoIceberg(after);
            break;

        case "u":
            GenericRecord updated = (GenericRecord) value.get("after");
            updateIceberg(updated);
            break;

        case "d":
            GenericRecord before = (GenericRecord) value.get("before");
            deleteFromIceberg(before);
            break;
    }
}
```

---

## CQRS (Command Query Responsibility Segregation)

### Pattern Overview

```
Write Side:                              Read Side:
┌──────────────┐                        ┌──────────────┐
│  Command     │                        │  Materialized│
│  Handler     │                        │    Views     │
└──────┬───────┘                        └──────▲───────┘
       │                                       │
       ▼                                       │
┌──────────────┐      ┌──────────────┐        │
│  Event Store │─────▶│    Kafka     │────────┘
│  (Source DB) │      │              │
└──────────────┘      └──────────────┘
```

### Write Model (Commands)

```java
public class OrderCommandHandler {
    private final OrderRepository orderRepo;
    private final KafkaProducer<String, OrderEvent> eventProducer;

    public void createOrder(CreateOrderCommand cmd) {
        // Validate command
        validateOrder(cmd);

        // Create order entity
        Order order = new Order(
            UUID.randomUUID(),
            cmd.getCustomerId(),
            cmd.getItems()
        );

        // Persist to write store
        orderRepo.save(order);

        // Publish event
        OrderCreatedEvent event = new OrderCreatedEvent(
            order.getId(),
            order.getCustomerId(),
            order.getItems(),
            Instant.now()
        );

        eventProducer.send(new ProducerRecord<>(
            "order.events",
            order.getId().toString(),
            event
        ));
    }
}
```

### Read Model (Queries)

```java
public class OrderQueryHandler {
    private final KafkaConsumer<String, OrderEvent> eventConsumer;
    private final ElasticsearchClient searchClient;

    public void start() {
        eventConsumer.subscribe(Arrays.asList("order.events"));

        while (true) {
            ConsumerRecords<String, OrderEvent> records =
                eventConsumer.poll(Duration.ofMillis(100));

            for (ConsumerRecord<String, OrderEvent> record : records) {
                OrderEvent event = record.value();

                // Build materialized view optimized for queries
                if (event instanceof OrderCreatedEvent) {
                    OrderView view = new OrderView();
                    view.setOrderId(event.getOrderId());
                    view.setCustomerId(event.getCustomerId());
                    view.setTotalAmount(calculateTotal(event.getItems()));
                    view.setCreatedAt(event.getTimestamp());

                    // Index in Elasticsearch for fast search
                    searchClient.index("orders", view);
                }
            }

            eventConsumer.commitAsync();
        }
    }

    public List<OrderView> findOrdersByCustomer(String customerId) {
        // Query read-optimized view
        return searchClient.search("orders", "customerId", customerId);
    }
}
```

### Benefits

- **Scalability:** Read and write models scale independently
- **Optimization:** Each model optimized for its use case
- **Flexibility:** Multiple read models from same events
- **Audit trail:** Full event history

---

## Saga Pattern (Distributed Transactions)

### Pattern Overview

```
Order Service ──▶ Kafka ──▶ Inventory Service
                    │
                    ├────▶ Payment Service
                    │
                    └────▶ Shipping Service

Each service:
1. Consumes event
2. Performs local transaction
3. Publishes success/failure event
4. Compensating transaction on failure
```

### Choreography-Based Saga

**Order Service:**
```java
public void createOrder(Order order) {
    // Local transaction
    orderRepo.save(order);

    // Publish event
    publishEvent(new OrderCreatedEvent(order));
}

@KafkaListener(topics = "payment.events")
public void handlePaymentResult(PaymentEvent event) {
    if (event instanceof PaymentSucceededEvent) {
        // Continue saga
        orderRepo.updateStatus(event.getOrderId(), OrderStatus.PAID);
    } else if (event instanceof PaymentFailedEvent) {
        // Compensating transaction
        orderRepo.updateStatus(event.getOrderId(), OrderStatus.CANCELLED);
        publishEvent(new OrderCancelledEvent(event.getOrderId()));
    }
}
```

**Inventory Service:**
```java
@KafkaListener(topics = "order.events")
public void handleOrderCreated(OrderCreatedEvent event) {
    try {
        // Reserve inventory
        inventoryRepo.reserve(event.getOrderId(), event.getItems());

        // Publish success
        publishEvent(new InventoryReservedEvent(event.getOrderId()));
    } catch (InsufficientInventoryException e) {
        // Publish failure
        publishEvent(new InventoryReservationFailedEvent(
            event.getOrderId(), e.getMessage()
        ));
    }
}

@KafkaListener(topics = "order.events")
public void handleOrderCancelled(OrderCancelledEvent event) {
    // Compensating transaction: release inventory
    inventoryRepo.release(event.getOrderId());
}
```

**Payment Service:**
```java
@KafkaListener(topics = "inventory.events")
public void handleInventoryReserved(InventoryReservedEvent event) {
    try {
        // Charge payment
        paymentGateway.charge(event.getOrderId(), event.getAmount());

        // Publish success
        publishEvent(new PaymentSucceededEvent(event.getOrderId()));
    } catch (PaymentException e) {
        // Publish failure (triggers compensating transactions)
        publishEvent(new PaymentFailedEvent(event.getOrderId(), e.getMessage()));
    }
}
```

### Orchestration-Based Saga

**Saga Orchestrator:**
```java
public class OrderSagaOrchestrator {
    @KafkaListener(topics = "order.commands")
    public void createOrder(CreateOrderCommand cmd) {
        SagaState state = new SagaState(cmd.getOrderId());

        // Step 1: Reserve inventory
        publishCommand(new ReserveInventoryCommand(cmd));
        state.setStep(SagaStep.INVENTORY_RESERVATION);
        saveSagaState(state);
    }

    @KafkaListener(topics = "inventory.events")
    public void handleInventoryResult(InventoryEvent event) {
        SagaState state = loadSagaState(event.getOrderId());

        if (event instanceof InventoryReservedEvent) {
            // Step 2: Charge payment
            publishCommand(new ChargePaymentCommand(event.getOrderId()));
            state.setStep(SagaStep.PAYMENT);
            saveSagaState(state);
        } else {
            // Rollback
            publishCommand(new CancelOrderCommand(event.getOrderId()));
        }
    }

    @KafkaListener(topics = "payment.events")
    public void handlePaymentResult(PaymentEvent event) {
        SagaState state = loadSagaState(event.getOrderId());

        if (event instanceof PaymentSucceededEvent) {
            // Step 3: Ship order
            publishCommand(new ShipOrderCommand(event.getOrderId()));
            state.setStep(SagaStep.SHIPPING);
            saveSagaState(state);
        } else {
            // Rollback: refund payment, release inventory
            publishCommand(new ReleaseInventoryCommand(event.getOrderId()));
            publishCommand(new CancelOrderCommand(event.getOrderId()));
        }
    }
}
```

### Saga State Persistence

Use Kafka as saga state store with compacted topics:

```properties
# Saga state topic configuration
cleanup.policy=compact
min.cleanable.dirty.ratio=0.5
delete.retention.ms=86400000  # 1 day
```

```java
// Persist saga state as Kafka message
ProducerRecord<String, SagaState> stateRecord = new ProducerRecord<>(
    "saga.states",
    sagaId,  // Key for compaction
    sagaState
);
producer.send(stateRecord);
```

---

## Outbox Pattern

### Problem

How to atomically update database and publish Kafka event?

### Solution

**Use database transaction with outbox table:**

```sql
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    customer_id VARCHAR(50),
    total_amount DECIMAL(10,2),
    status VARCHAR(20)
);

CREATE TABLE outbox (
    id UUID PRIMARY KEY,
    aggregate_id UUID NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);
```

**Application code:**
```java
@Transactional
public void createOrder(Order order) {
    // 1. Insert order
    orderRepo.save(order);

    // 2. Insert event into outbox (same transaction)
    OutboxEvent event = new OutboxEvent();
    event.setAggregateId(order.getId());
    event.setEventType("OrderCreated");
    event.setPayload(toJson(order));
    outboxRepo.save(event);

    // Transaction commits atomically
}
```

**Outbox publisher (separate process):**
```java
while (true) {
    List<OutboxEvent> events = outboxRepo.findUnpublished(limit=100);

    for (OutboxEvent event : events) {
        // Publish to Kafka
        producer.send(new ProducerRecord<>(
            "order.events",
            event.getAggregateId().toString(),
            event.getPayload()
        ));

        // Mark as published
        outboxRepo.markPublished(event.getId());
    }

    Thread.sleep(100);
}
```

**Or use Debezium to stream outbox table directly to Kafka:**

```json
{
  "name": "outbox-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "table.include.list": "public.outbox",
    "transforms": "outbox",
    "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
    "transforms.outbox.table.field.event.type": "event_type",
    "transforms.outbox.table.field.event.key": "aggregate_id",
    "transforms.outbox.route.topic.replacement": "order.events"
  }
}
```

---

## Stream Processing Patterns

### Stateless Transformation

```java
// Kafka Streams: Filter and transform
StreamsBuilder builder = new StreamsBuilder();

KStream<String, Order> orders = builder.stream("orders");

KStream<String, OrderView> highValueOrders = orders
    .filter((key, order) -> order.getAmount() > 1000)
    .mapValues(order -> new OrderView(
        order.getId(),
        order.getCustomerId(),
        order.getAmount(),
        "HIGH_VALUE"
    ));

highValueOrders.to("high-value-orders");
```

### Stateful Aggregation

```java
// Count orders per customer
KTable<String, Long> orderCountsByCustomer = orders
    .groupBy((orderId, order) -> order.getCustomerId())
    .count();

orderCountsByCustomer.toStream().to("customer-order-counts");
```

### Join Patterns

```java
// Join orders with customers
KStream<String, Order> orders = builder.stream("orders");
KTable<String, Customer> customers = builder.table("customers");

KStream<String, EnrichedOrder> enrichedOrders = orders
    .selectKey((orderId, order) -> order.getCustomerId())
    .join(customers,
        (order, customer) -> new EnrichedOrder(
            order,
            customer.getName(),
            customer.getSegment()
        )
    );
```

---

## Best Practices

1. **Use event sourcing for audit trails** - Immutable event log
2. **Design events for evolution** - Include version, use Schema Registry
3. **Partition by entity ID** - Maintains ordering per entity
4. **Use compacted topics for state** - Saga state, materialized views
5. **Implement idempotency** - Consumers may see duplicates
6. **Handle out-of-order events** - Use timestamps, version vectors
7. **Monitor event lag** - Delays indicate problems
8. **Test failure scenarios** - Saga compensations, network partitions
9. **Document event schemas** - Centralized catalog
10. **Use Outbox pattern** - Atomic database + Kafka updates

---

**Note:** For CDC connector configuration, see references/kafka-connect.md
