---
name: spring-boot-ddd
description: 'Spring Boot mechanics for the ddd-* skills — use alongside them (never instead of them) when implementing DDD patterns. Supplies the Spring realization of their decisions: package layout with ArchUnit enforcement, transaction placement (Spring @Transactional on application-service methods only), the repository adapter implementing the hand-written domain interface over an internal Spring Data JPA repository, @Version compare-and-set mapped to 409, after-commit delivery via @TransactionalEventListener or an outbox to Kafka, and layer-matched test tiers. Trigger on DDD vocabulary (aggregate, value object, repository, application service, domain event, bounded context) in a Spring Boot codebase, and on @Entity on a domain type, a JpaRepository as the domain repository, @Transactional outside <bc>/application, KafkaTemplate or ApplicationEventPublisher in <bc>/domain, or @SpringBootTest on a domain test. Defines no DDD concepts (the ddd-* skills do); project creation and dependencies are spring-boot-app.'
---

# Spring Boot DDD Mechanics

The Spring Boot tier under the `ddd-*` skills. Those skills decide **what** (layers, aggregate shape, repository contract, who owns the transaction, when events are delivered). This skill decides **how Spring Boot does it**. If a rule here appears to conflict with a `ddd-*` rule, the `ddd-*` rule wins and this file has a bug.

**Load order.** `spring-boot-app` first (the project exists, starters are in place), then the `ddd-*` skill for the concern at hand, then this file, then exactly one reference file from `references/` for the concern being implemented.

**Red Flags — STOP if you find yourself thinking:**

- "The aggregate is the `@Entity` — that's how Spring Boot apps are written." (The aggregate is a plain object; JPA belongs on the infrastructure mirror.)
- "An `OrderJpaRepository extends JpaRepository<OrderRecord, UUID>` *alone* is the repository." (The domain interface is hand-written and speaks domain types; Spring Data is an internal detail of the infrastructure adapter that implements it.)
- "Wrap just the `save` call in `TransactionTemplate` and publish the event afterwards." (The transaction is the application-service method; delivery is after commit by mechanism, not by call ordering.)
- "`@Transactional` on the repository, to be safe." (Repositories never demarcate — `ddd-persistence` Rule 12. Spring Data's own `SimpleJpaRepository` transactions are an implementation detail, not a boundary.)
- "A domain unit test needs `@SpringBootTest`." (If it does, the domain type has a framework dependency; fix the type.)
- "The value object gets its own `@Entity` with an `@Id`." (A value object has no identity — `@Embeddable` or columns on the mirror.)

---

## Stack defaults

| Concern | Default | Alternative (say why) |
| --- | --- | --- |
| Java | 25 (per `spring-boot-app`) | — |
| HTTP | Spring Web MVC + Jackson (`web`) | WebFlux |
| Persistence | Spring Data JPA (`data-jpa`, `postgresql`): the infrastructure adapter class implements the domain interface and delegates to an internal `JpaRepository<Record, UUID>` | `JdbcClient`/`EntityManager` for a query Spring Data cannot express; never a bare `JpaRepository` as the domain contract |
| Events in-process | `ApplicationEventPublisher` published by the application service; listeners `@TransactionalEventListener(phase = AFTER_COMMIT)` | Spring Modulith `@ApplicationModuleListener` (adds async + own transaction + publication log) |
| Events cross-service | Transactional outbox row → `@Scheduled` relay → `KafkaTemplate` (`kafka` dependency) | Spring Modulith event externalization; Debezium outbox |
| Tests | JUnit 5; slices (`@DataJpaTest`, `@WebMvcTest`) and `@SpringBootTest`; Testcontainers with `@ServiceConnection`; MockMvc | Docker Compose support for dev-time services |
| Boundary enforcement | ArchUnit test per bounded context (`archunit-junit5`, test scope) | Spring Modulith `ApplicationModules.verify()`; Maven multi-module |

Dependency bundle for a DDD-shaped service — select at generation time via Initializr (`spring-boot-app`):
`web, data-jpa, postgresql, validation, testcontainers`, plus `kafka` when events leave the service. ArchUnit (`com.tngtech.archunit:archunit-junit5`, test scope) is added to the build file by hand — it is not an Initializr dependency. `validation` is for api-layer request types only; value objects validate themselves (`ddd-value-objects`).

---

## Layout

Single Maven module; packages mirror the `ddd-foundations` layers exactly:

```
{base}.{bc}.domain          # aggregates, value objects, domain services, repository interfaces, events
{base}.{bc}.application     # application services, commands, DTOs
{base}.{bc}.infrastructure  # JPA mirrors, Spring Data-backed repository adapters, outbox, Kafka adapters
{base}.{bc}.api             # @RestController classes, request/response types, @RestControllerAdvice
```

No `persistence/` package and no `domain/services/` package: `ddd-foundations` has four layers and this skill does not add a fifth. Boundaries are enforced by an ArchUnit test (`references/architecture-tests.md`), not by review. Keep the `@SpringBootApplication` class in `{base}` so component scanning covers every bounded context without a `scanBasePackages` list.

---

## Core mechanics

1. **Transaction = application-service method.** `@org.springframework.transaction.annotation.Transactional` on the public method of a `*ApplicationService` in `application`. Nowhere else: not on controllers, repositories, domain services, or aggregates. Prefer the annotation over `TransactionTemplate`; use the programmatic API only for a boundary the annotation cannot express, and say why in a comment. (Always the Spring annotation, never `jakarta.transaction.Transactional` — one import, one semantics.)
2. **The Spring container is the composition root.** `@Service` on application services, `@Component` on repository adapters and other infrastructure beans; constructor injection everywhere (single constructor, no `@Autowired` needed, final fields). Domain types other than domain services carry no Spring annotations.
3. **The mirror carries the framework.** `@Entity`, `@Table`, `@Id UUID`, `@Version Long`, column mappings go on `OrderRecord` in `infrastructure`; fields private, with `static fromDomain(Order)` and `toDomain()` beside them. The `@Version` field is the `Long` wrapper, not `long`: Spring Data treats a null version as "new" (INSERT) and a non-null version as "existing" (merge), which is what makes `save` an upsert for assigned IDs. Value objects map to `@Embeddable` types or to columns — never to their own `@Entity` with an `@Id` (`ddd-value-objects` Rule 7).
4. **The infrastructure repository is an adapter over Spring Data, behind the domain interface.** `class OrderRepository implements com.example.orders.domain.OrderRepository` (same simple name as the interface — `ddd-persistence` Rule 3) delegating to a package-private `interface OrderJpaRepository extends JpaRepository<OrderRecord, UUID>` in the same package. Spring Data's derived queries (`findByCustomerId`), `@Query`, `Pageable`/`Sort` live on the internal interface; only domain-typed methods are on the domain interface. `toDomain()` calls the aggregate's `rehydrate(...)`.
5. **Upsert is `saveAndFlush`.** `persist(Order)` maps to a detached mirror carrying the aggregate's `version` (null when the aggregate is new), then `jpa.saveAndFlush(record)`. Flushing inside the adapter makes the compare-and-set fail *here*, in the application service's call, rather than wrapped at commit.
6. **Lock conflict → 409.** Catch `org.springframework.dao.OptimisticLockingFailureException` in the repository adapter (Spring's exception translation has already wrapped JPA's `OptimisticLockException`) and rethrow the bounded context's domain `ConcurrencyConflictException` (`ddd-persistence` Rule 7); an `@ExceptionHandler` in the context's `@RestControllerAdvice` maps it to `409 Conflict` as a `ProblemDetail`. The domain exception is the only thing that crosses layers, and the advice is the one `api` type allowed to name it (the ArchUnit test ignores exactly that dependency).
7. **Events deliver after commit, by mechanism.** The application service drains `pullPendingEvents()` and publishes each through `ApplicationEventPublisher` inside its transaction. In-process handlers listen with `@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)` and delegate to an application service whose method is `@Transactional(propagation = REQUIRES_NEW)` — after commit the completed publisher transaction's resources are still bound to the thread, so plain `REQUIRED` would join them and its writes would never commit. Right timing, but in-memory: a crash after commit loses the event, so use it only where that is tolerable. Cross-service delivery, and any event that must not be lost, writes an outbox row in the same transaction; a relay publishes via `KafkaTemplate`. Never `kafkaTemplate.send(...)` from `application`. (Spring Modulith's event publication registry is the packaged fix for the lost-event gap — note it as the alternative when the project already uses Modulith.)
8. **Kafka topic per bounded context**, broker config under `spring.kafka.*` in `application.properties`, topic names as application constants in `infrastructure`. Topic payloads are integration events (`ddd-domain-events` Rule 9), mapped from domain events in `infrastructure` — the domain event type is not the wire format.
9. **Test tiers follow the layers.** `domain`: plain JUnit 5, no Spring. `application`: `@SpringBootTest` with Testcontainers PostgreSQL (`@ServiceConnection`) and the real repository adapter; the mapping/locking detail lives in a `@DataJpaTest`. `api`: `@WebMvcTest` + MockMvc with the application service mocked. A domain type never needs `@SpringBootTest`; if it does, it has a framework dependency — fix the type.
10. **Logging lives in `api` and `infrastructure`.** SLF4J (`LoggerFactory.getLogger`) at DEBUG at adapter boundaries; nothing in `domain` or `application`.

---

## Reference files

Load exactly one, for the concern being implemented. Each exercises the numbered Core Rules of the matching `ddd-*` skill and cites them.

| Implementing… | Read | Pairs with |
| --- | --- | --- |
| A new bounded context's package tree, controller, application service, repository wiring | `references/foundations.md` | `ddd-foundations` |
| An application-service / domain-service split, transaction and event hand-off | `references/services.md` | `ddd-services` |
| A repository, mirror, `@Version`, 409 mapping, use-case-optimal query | `references/persistence.md` | `ddd-persistence` |
| Event delivery: `AFTER_COMMIT` listeners, outbox, Kafka config, integration events | `references/domain-events.md` | `ddd-domain-events` |
| The ArchUnit layer test for a bounded context | `references/architecture-tests.md` | `ddd-foundations` |

Aggregate and value-object code is stack-neutral Java and stays in `ddd-aggregates/references/canonical-example-java.md` and `ddd-value-objects/references/canonical-example-java.md`.

---

## Quick Reference

- [ ] `@Transactional` (Spring's) appears only on `*ApplicationService` methods in `application` — never `jakarta.transaction.Transactional` anywhere.
- [ ] No `jakarta.persistence`, `org.springframework` (except `org.springframework.stereotype` on domain services), or `org.hibernate` import under `domain` (ArchUnit enforces).
- [ ] The domain repository interface is hand-written; `JpaRepository` appears only on the package-private Spring Data interface inside `infrastructure`, behind the same-named adapter class.
- [ ] Value objects persist as `@Embeddable` or columns; no `@Entity` with its own `@Id` for a value object.
- [ ] `persist` = `saveAndFlush`; mirror `@Version` is `Long`, null when new; `OptimisticLockingFailureException` is translated in `infrastructure`; `api` maps the domain exception to 409.
- [ ] Events go through `ApplicationEventPublisher` with `AFTER_COMMIT` listeners or an outbox; no `KafkaTemplate.send` in `application`.
- [ ] Domain tests have no `@SpringBootTest`.

### Companion skills

- **`ddd-foundations`, `ddd-aggregates`, `ddd-value-objects`, `ddd-services`, `ddd-persistence`, `ddd-domain-events`** — the design rules this skill implements. Written against `ddd-skills` 1.x; Core Rule numbers cited in `references/` refer to that version.
- **`spring-boot-app`** — creating the project, choosing the Spring Boot version, adding dependencies.
