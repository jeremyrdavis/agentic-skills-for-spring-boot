# Canonical Example — Repository, Mirror, and Use-Case-Optimal Query (Java / Spring Boot, Spring Data JPA)

Load this file when implementing a repository or adding a non-trivial query in
a Spring Boot project. For placement and review decisions, the Core Rules and
Quick Reference tables in `ddd-persistence/SKILL.md` are sufficient on their
own. Comments cite `ddd-persistence` rule numbers.

## 1. The interface — `<bc>/domain`, ubiquitous language only

```java
package com.example.orders.domain;

import java.util.Optional;

public interface OrderRepository {
    Optional<Order> findById(OrderId id);                    // aggregate out (Rule 2)
    void persist(Order order);                                // whole aggregate, upsert (Rule 6)
    void remove(OrderId id);
    OrderPage findRecent(int page, int size);                 // use-case-optimal VO (Rules 9, 10)
    void incrementThumbsUp(OrderId id);                       // atomic op (Rule 8) — section 5
}
```

No `org.springframework`, no `jakarta.persistence`, no Spring Data in the
interface. Its vocabulary is the aggregate's vocabulary.

## 2. The purpose-built query VOs — `<bc>/domain`, per `ddd-value-objects`

Domain-specific, immutable, equality by value — explicitly **not** DTOs (Rule 9).

```java
package com.example.orders.domain;

public record OrderSummary(OrderId id, CustomerId customerId,
                           Money total, Order.Status status) {}

public record OrderPage(java.util.List<OrderSummary> items,
                        long totalCount, int page, int size) {
    public OrderPage { items = java.util.List.copyOf(items); }   // defensive immutability
}
```

## 3. The mirror — `<bc>/infrastructure`, carries every framework concern (Rule 4)

```java
package com.example.orders.infrastructure;

import com.example.orders.domain.*;
import jakarta.persistence.*;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

// Mirrors the canonical Order in ddd-aggregates/references/canonical-example-java.md:
// same fields, and toDomain() calls rehydrate(...) with its exact parameter list.
@Entity
@Table(name = "orders")
public class OrderRecord {

    @Id private UUID id;                              // assigned by the domain, not generated
    @Version private Long version;                    // the lock column (Rule 7) — wrapper, not primitive:
                                                      // Spring Data reads null as "new" (INSERT) and
                                                      // non-null as "existing" (merge) — the upsert hinge
    private UUID customerId;
    @Embedded private MoneyColumns total;             // value object → @Embeddable, no identity
    @Enumerated(EnumType.STRING) private Order.Status status;
    private Instant placedAt;
    @ElementCollection(fetch = FetchType.EAGER)       // lines are value objects inside the aggregate: no identity,
    @CollectionTable(name = "order_lines",            // loaded and saved with the root, covered by its @Version
                     joinColumns = @JoinColumn(name = "order_id"))
    @OrderColumn(name = "position")
    private List<OrderLineColumns> lines = new ArrayList<>();

    protected OrderRecord() {}                        // JPA's constructor — the mirror's problem, not the aggregate's

    static OrderRecord fromDomain(Order order) {      // outbound translation (Rule 5)
        var r = new OrderRecord();
        r.id = order.id().value();
        r.version = order.version() == 0 ? null : order.version();   // 0 = never persisted → null → INSERT
        r.customerId = order.customerId().value();
        r.total = MoneyColumns.of(order.total());
        r.status = order.status();
        r.placedAt = order.placedAt();
        r.lines = new ArrayList<>(order.lines().stream().map(OrderLineColumns::of).toList());
        return r;
    }

    Order toDomain() {                                // inbound translation — the only rehydrate caller (Rule 5)
        return Order.rehydrate(id, new CustomerId(customerId),
                               lines.stream().map(OrderLineColumns::toDomain).toList(),
                               status, total.toDomain(), placedAt, version);
    }

    OrderSummary toSummary() {
        return new OrderSummary(new OrderId(id), new CustomerId(customerId), total.toDomain(), status);
    }
}
```

`MoneyColumns` is an `@Embeddable` with `amount` and `currency` columns and a
`toDomain()` that calls the `Money` constructor, so value-object validation
re-runs on the way in. `OrderLineColumns` is the same idea for `OrderLine`
(`productSku`, `quantity`, an embedded `MoneyColumns unitPrice`; `of(OrderLine)`
and `toDomain()`): an order line is a value object inside the aggregate, so it
gets a collection table, never an `@Entity` of its own (`ddd-value-objects` Rule 7).

## 4. The implementation — `<bc>/infrastructure`, same name, Spring Data behind the interface (Rule 3)

The Spring Data interface is package-private machinery. Derived queries,
`@Query`, `Pageable` — all of it stays here:

```java
package com.example.orders.infrastructure;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import java.util.UUID;

interface OrderJpaRepository extends JpaRepository<OrderRecord, UUID> {

    Page<OrderRecord> findAllByOrderByPlacedAtDesc(Pageable pageable);

    @Modifying
    @Query("update OrderRecord o set o.thumbsUp = o.thumbsUp + 1, o.version = o.version + 1 where o.id = :id")
    void incrementThumbsUp(@Param("id") UUID id);     // see section 5
}
```

The adapter implements the domain contract and owns the translation:

```java
package com.example.orders.infrastructure;

import com.example.orders.domain.*;
import org.springframework.dao.OptimisticLockingFailureException;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.stereotype.Component;
import java.util.Optional;

@Component
// Same simple name as the domain interface — the package is the distinguisher,
// so the implements clause is fully qualified. No technology in the name (Rule 3).
// Spring Data's verbs arrive through the package-private OrderJpaRepository;
// they are NOT on the domain interface and the Spring Data interface never
// leaves this package.
public class OrderRepository implements com.example.orders.domain.OrderRepository {

    private final OrderJpaRepository jpa;

    OrderRepository(OrderJpaRepository jpa) { this.jpa = jpa; }

    @Override
    public Optional<Order> findById(OrderId id) {
        return jpa.findById(id.value())                      // Spring Data
                .map(OrderRecord::toDomain);                 // rehydrate inside (Rule 5)
    }

    @Override
    public void persist(Order order) {                       // whole aggregate, upsert (Rule 6)
        OrderRecord record = OrderRecord.fromDomain(order);  // detached mirror carrying the loaded version
        try {
            jpa.saveAndFlush(record);                        // null version → INSERT, non-null → merge;
        } catch (OptimisticLockingFailureException stale) {  // the flush makes the compare-and-set fail in
            throw new ConcurrencyConflictException(order.id(), stale);   // this call, not at commit —
        }                                                    // domain-neutral conflict (Rule 7)
    }

    @Override
    public void remove(OrderId id) {
        jpa.deleteById(id.value());                          // Spring Data
    }

    @Override
    public OrderPage findRecent(int page, int size) {        // Vernon query (Rules 9, 10)
        Page<OrderRecord> result = jpa.findAllByOrderByPlacedAtDesc(PageRequest.of(page, size));
        var items = result.getContent().stream().map(OrderRecord::toSummary).toList();
        return new OrderPage(items, result.getTotalElements(), page, size);
    }

    @Override
    public void incrementThumbsUp(OrderId id) {              // atomic op — section 5
        jpa.incrementThumbsUp(id.value());
    }
}
```

**Conflict surfacing (Rule 7):** Spring's exception translation has already
turned Hibernate's `OptimisticLockException` into
`OptimisticLockingFailureException` by the time it leaves the Spring Data call;
the adapter rethrows the domain `ConcurrencyConflictException`. The api layer
maps it, and nothing else does. The advice is the one `api` type that names a
domain exception type (`ddd-foundations` placement table); the ArchUnit test in
`references/architecture-tests.md` ignores exactly that dependency and no other
api → domain edge:

```java
// com.example.orders.api
import com.example.orders.domain.ConcurrencyConflictException;   // the one domain import api is allowed

@RestControllerAdvice
public class OrdersExceptionAdvice {

    @ExceptionHandler(ConcurrencyConflictException.class)
    public ProblemDetail onConflict(ConcurrencyConflictException e) {
        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.CONFLICT);
        problem.setDetail(e.getMessage());
        return problem;                                       // RFC 9457 response body
    }
}
```

The repository never retries — the use case decides what a conflict means.

## 5. An atomic operation (Rule 8)

`thumbsUp` has no invariant, so load-mutate-persist would be a lost-update
hazard. The domain interface gets an intention-revealing method
(`incrementThumbsUp(OrderId)` in section 1); the implementation is the one
`@Modifying @Query` shown on `OrderJpaRepository` above.

One UPDATE, atomic under concurrency. It advances the lock column itself,
because a JPQL bulk update bypasses `@Version`: without
`o.version = o.version + 1` a transaction that loaded the aggregate earlier
could still merge successfully and overwrite `thumbsUp`. The `@Modifying`
query executes inside the calling application service's transaction — no
`@Transactional` on the repository (Rule 12).

## What this demonstrates

- **Interface in domain vocabulary (Rule 2):** typed IDs in, aggregates and
  domain VOs out; zero framework imports in `<bc>/domain`.
- **Same-name implementation (Rule 3):** package is the distinguisher;
  fully-qualified `implements` is Java's spelling of it. Spring Data is a
  detail of the adapter, not of the contract.
- **Translation is the whole job (Rule 5):** `toDomain` calls
  `Order.rehydrate(...)`; VO constructors re-validate on the way in.
- **Upsert (Rule 6):** `saveAndFlush` saves the whole aggregate in one call;
  the `Long` version is the new-vs-existing hinge.
- **Locking (Rule 7):** the mirror's `@Version` does the compare-and-set; the
  boundary surfaces a domain-neutral conflict → 409 `ProblemDetail`.
- **Atomic op (Rule 8):** one `@Modifying @Query` behind a named method,
  bumping `version` so the whole-aggregate path still rejects stale writers.
- **Vernon query (Rules 9, 10):** `OrderPage`/`OrderSummary` are immutable
  domain records — not DTOs; `Pageable`/`Page`/derived-query names never cross
  the interface.

## Spring Boot idioms

- `@Version` on the mirror gives compare-and-set for free; keep the aggregate's
  own `version` a plain `long` (per `ddd-aggregates` Rule 10) and translate
  `0 ↔ null` at the mirror boundary in `fromDomain`.
- Assigned UUID ids + the `Long` version column mean `save` inserts a new
  aggregate and merges an existing one — one code path for Rule 6's upsert.
  (Without the wrapper version, Spring Data would try to INSERT every detached
  mirror and fail on existing rows.)
- Prefer Spring Data derived queries and `@Query` on the package-private
  interface; drop to `EntityManager` (injected into the adapter) only for a
  query Spring Data cannot express.
- Repository tests split: contract tests against an in-memory fake for callers;
  `@DataJpaTest` with Testcontainers PostgreSQL (a `@ServiceConnection`
  container — Spring Boot detects the container-sourced database, so no
  `@AutoConfigureTestDatabase(replace = NONE)` is needed) for the mapping, the
  lock, and the atomic update — H2 is not PostgreSQL, so the slice runs against
  the real engine.
