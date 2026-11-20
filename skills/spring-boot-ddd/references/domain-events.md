# Canonical Example — After-Commit Delivery, Outbox, and Integration Events (Java / Spring Boot)

Load this file when wiring event delivery in a Spring Boot project. The rules —
what an event is, what it carries, when it is delivered — are in
`ddd-domain-events/SKILL.md`; comments cite its rule numbers. The aggregate
raises events (`ddd-aggregates` Rule 12) and the application service drains
them (`ddd-services` Rule 11); this file starts where the drain ends.

## 1. The hand-off — application service, inside the transaction

```java
// com.example.orders.application.CancelOrderApplicationService — the relevant lines
@Transactional
public OrderDTO cancel(OrderId id) {
    Order order = orders.findById(id).orElseThrow(() -> new OrderNotFoundException(id));
    order.cancel();                                              // raises OrderCancelledEvent inside the aggregate
    orders.persist(order);
    order.pullPendingEvents().forEach(events::publishEvent);     // ApplicationEventPublisher — drain and hand off
    return OrderDTO.of(order);
}
```

`events.publishEvent(...)` runs inside the transaction. Nothing is delivered
yet: Spring holds `@TransactionalEventListener` listeners until the phase they
declared (Rule 6).

## 2. In-process listener — `AFTER_COMMIT`, own transaction, same bounded context

A domain event is observed only inside the context that raised it (Rule 9);
here a second aggregate of the orders context, `Reservation`, reacts. Another
context never imports `orders.domain` — it consumes the integration event in
section 4.

```java
package com.example.orders.infrastructure;

import com.example.orders.application.ReleaseReservationApplicationService;
import com.example.orders.domain.OrderCancelledEvent;             // same bounded context
import org.springframework.stereotype.Component;
import org.springframework.transaction.event.TransactionPhase;
import org.springframework.transaction.event.TransactionalEventListener;

@Component
public class OrderCancelledListener {                             // inbound adapter — infrastructure

    private final ReleaseReservationApplicationService release;

    OrderCancelledListener(ReleaseReservationApplicationService release) { this.release = release; }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    void on(OrderCancelledEvent event) {
        release.forOrder(event.orderId(), event.eventId());       // its own transaction (Rule 7)
    }
}
```

```java
// com.example.orders.application.ReleaseReservationApplicationService — the relevant lines
@Transactional(propagation = Propagation.REQUIRES_NEW)            // not REQUIRED — see below
public void forOrder(OrderId orderId, UUID eventId) {
    if (handled.contains(eventId)) return;                        // idempotent (Rule 8)
    Reservation reservation = reservations.findByOrder(orderId).orElseThrow(...);
    reservation.release();                                        // the second aggregate, its own transaction
    reservations.persist(reservation);
    handled.record(eventId);
}
```

The listener carries no `@Transactional` of its own: it is delivered after the
publisher's commit and *calls an application service*, whose method opens the
handler's transaction with `REQUIRES_NEW`. During `AFTER_COMMIT` the completed
publisher transaction's resources are still bound to the thread, so a plain
`@Transactional` (`REQUIRED`) would join that finished transaction and its
writes would never commit — Spring's own documentation says to use
`REQUIRES_NEW` here. The handler dedupes on `eventId` (Rule 8) —
`AFTER_COMMIT` delivery is in-memory, so a crash between commit and listener
loses the event unless the handler can be re-driven; use the outbox below when
that matters. (Spring Modulith's `@ApplicationModuleListener` packages this
pattern — async + `REQUIRES_NEW` + `AFTER_COMMIT` — and adds an event
publication log that closes the lost-event gap; prefer it when the project
already uses Modulith.)

## 3. Cross-service — transactional outbox to Kafka

**Outbox row, written in the publisher's transaction** (a plain `@EventListener`
runs synchronously inside the publishing transaction):

```java
package com.example.orders.infrastructure;

@Entity @Table(name = "orders_outbox")
public class OutboxRecord {
    @Id UUID id;                         // = eventId — the consumer's idempotency key
    String type;                         // "OrderCancelled", version in the payload
    @Column(columnDefinition = "text")
    String payload;                      // integration-event JSON — never the default varchar(255)
    Instant occurredAt;                  // when it happened — metadata, not the relay's ordering key
    long seq;                            // DB identity/sequence column — the relay orders on this
    Instant publishedAt;                 // null until the relay sends it
    // constructors/factory elided
}

@Component
public class OutboxWriter {
    private final OutboxJpaRepository outbox;              // JpaRepository<OutboxRecord, UUID>, package-private
    private final IntegrationEventMapper mapper;           // domain event → integration event (Rule 9)
    OutboxWriter(OutboxJpaRepository outbox, IntegrationEventMapper mapper) { this.outbox = outbox; this.mapper = mapper; }

    @EventListener                                         // synchronous — runs inside the firing transaction
    void on(DomainEvent event) {
        outbox.save(OutboxRecord.of(mapper.toIntegrationEvent(event)));   // same transaction as the aggregate
    }
}
```

**Relay** — the only class that touches the broker:

```java
package com.example.orders.infrastructure;

import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

@Component
public class OutboxRelay {
    static final String TOPIC = "orders";                  // topic = bounded context (spring-boot-ddd mechanic 8)

    private final KafkaTemplate<String, String> kafka;
    private final OutboxJpaRepository outbox;
    OutboxRelay(KafkaTemplate<String, String> kafka, OutboxJpaRepository outbox) { this.kafka = kafka; this.outbox = outbox; }

    @Scheduled(fixedDelay = 1000)
    @Transactional                                         // infrastructure job, not a use case — the one tolerated exception
    void relay() {
        for (OutboxRecord r : outbox.unpublished(100)) {   // "publishedAt is null order by seq"
            kafka.send(TOPIC, r.getPayload()).join();      // wait for the broker's ack: a failure throws,
            r.markPublished(Instant.now());                //   publishedAt rolls back, the next tick retries
            // at-least-once: a crash after the ack and before commit re-sends the row — consumers dedupe on eventId
        }
    }
}
```

Scheduling needs `@EnableScheduling` once, on a `@Configuration` class in
`infrastructure`. `application.properties` (broker config is all under
`spring.kafka.*`):

```properties
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.apache.kafka.common.serialization.StringSerializer
```

Alternatives: Spring Modulith's event externalization
(`@Externalized("orders")` on the integration event + the
`spring-modulith-events-kafka` starter) writes a publication log and publishes
after commit, removing the hand-rolled outbox and relay. Debezium can tail the
same outbox table. Same contract either way.

## 4. The integration event — a published, versioned contract (Rule 9)

```java
package com.example.orders.infrastructure.contracts;              // what other services depend on

public record OrderCancelledV1(UUID eventId, Instant occurredAt,
                               UUID orderId, UUID customerId, String reason) {}
```

Mapped from `OrderCancelledEvent` by `IntegrationEventMapper` in
`infrastructure`. The domain event is free to change with the model; `V1` is
not. The consumer in another bounded context deserializes `OrderCancelledV1`,
dedupes on `eventId`, and calls *its* application service — it never imports
`com.example.orders.domain`:

```java
package com.example.inventory.infrastructure;                     // the other context's inbound adapter

import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

@Component
public class OrdersConsumer {
    private final ReleaseStockApplicationService release;         // inventory's own use case, own @Transactional
    OrdersConsumer(ReleaseStockApplicationService release) { this.release = release; }

    @KafkaListener(topics = "orders", groupId = "inventory")      // spring.kafka.consumer.* in this service
    public void on(OrderCancelledV1 event) {                      // JsonDeserializer for the contract type
        release.forOrder(event.orderId(), event.eventId());       // dedupes on eventId inside
    }
}
```

## What this demonstrates

- **After-commit by mechanism (Rule 6):** `AFTER_COMMIT` listeners in-process
  (right timing, best-effort — in-memory); outbox + relay wherever the event
  must not be lost, and always across services. Neither relies on "publish
  after the call returns".
- **Own transaction per handler (Rule 7):** the listener delegates to an
  application service running `REQUIRES_NEW`; the relay's `@Transactional` is
  infrastructure plumbing.
- **Idempotent handlers (Rule 8):** `eventId` is the outbox key and the
  consumer's dedupe key; at-least-once is the assumption.
- **Domain vs integration events (Rule 9):** the Kafka payload is `V1`, mapped
  in `infrastructure`; the domain type never leaves the service, and the other
  context's consumer depends on the `V1` contract only.
- **No broker in domain or application (Rule 10):** `KafkaTemplate` appears in
  one infrastructure class.

## Spring Boot idioms

- `@TransactionalEventListener` fires only when a transaction is active at
  publish time; outside one, the event is silently dropped by default
  (`fallbackExecution = false`) — one more reason the `@Transactional` stays on
  the application service.
- Publish the integration-event JSON as a `String` payload, or a typed
  `KafkaTemplate<String, OrderCancelledV1>` with Spring Kafka's
  `JsonSerializer`, never a `KafkaTemplate<…, OrderCancelledEvent>`.
- `KafkaTemplate.send` returns a `CompletableFuture<SendResult>` that completes
  on the broker's ack; the relay joins it before marking the row published, so
  a send failure rolls `publishedAt` back and the row is retried.
- Testcontainers starts Kafka for integration tests
  (`@ServiceConnection KafkaContainer`); test the relay against it, and test
  listeners by publishing domain events inside a transaction with
  `ApplicationEventPublisher` and asserting on the delegated application
  service — `@RecordApplicationEvents` captures what was published when the
  broker is noise.
