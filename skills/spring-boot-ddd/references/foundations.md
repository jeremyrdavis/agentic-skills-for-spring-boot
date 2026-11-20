# Canonical Example — `payments` Bounded Context (Java / Spring Boot)

Load this file when scaffolding a new bounded context in a Spring Boot project.
For placement and review decisions, the Core Rules and Quick Reference tables in
`ddd-foundations/SKILL.md` are sufficient on their own.

Use case: *authorize a payment* — load the `Payment` aggregate, run a
domain-service fraud check, advance the aggregate's state machine, persist via an
infrastructure repository adapter, return a `PaymentDTO`.

## Module layout

Layers are Java packages under one bounded-context package:

```
com.example.payments
├── domain
│   ├── PaymentMethod.java              (value object — record)
│   ├── Payment.java                    (aggregate root — plain class, no framework imports)
│   ├── PaymentRepository.java          (interface — uses only domain types)
│   ├── FraudDetectionDomainService.java
│   └── PaymentAuthorizedEvent.java     (domain event — record, past tense, *Event suffix)
├── application
│   ├── AuthorizePaymentCommand.java    (input DTO — record)
│   ├── PaymentDTO.java                 (output DTO — record)
│   └── AuthorizePaymentApplicationService.java
├── infrastructure
│   ├── PaymentRecord.java              (JPA-mapped mirror entity — jakarta.persistence lives here)
│   ├── PaymentJpaRepository.java       (package-private Spring Data interface — never leaves this package)
│   └── PaymentRepository.java          (same name as the domain interface; adapter over Spring Data, translates)
└── api
    ├── PaymentsController.java         (@RestController)
    └── PaymentsExceptionAdvice.java    (@RestControllerAdvice)
```

Allowed dependency directions (one way only):

```
api ──▶ application ──▶ domain
                           ▲
                infrastructure
```

## The key files

### 1. Repository interface — `domain`, speaks only domain language

```java
package com.example.payments.domain;

import java.util.Optional;

public interface PaymentRepository {
    Optional<Payment> findById(PaymentId id);
    Optional<Payment> findByOrderId(OrderId orderId);
    void persist(Payment payment);
}
```

No `jakarta.persistence`, no Spring Data, no Hibernate *in the interface*. Its
vocabulary is the aggregate's vocabulary.

### 2. Repository implementation — `infrastructure`, brings the framework (Spring Data JPA)

Two types, same package. The Spring Data interface is machinery — package-private,
so nothing outside `infrastructure` can inject it:

```java
package com.example.payments.infrastructure;

import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;
import java.util.UUID;

interface PaymentJpaRepository extends JpaRepository<PaymentRecord, UUID> {
    Optional<PaymentRecord> findByOrderId(UUID orderId);     // derived query — record types only
}
```

The adapter is what the rest of the application sees as *the* `PaymentRepository`:

```java
package com.example.payments.infrastructure;

import com.example.payments.domain.OrderId;
import com.example.payments.domain.Payment;
import com.example.payments.domain.PaymentId;
import org.springframework.stereotype.Component;
import java.util.Optional;

@Component
// Same simple name as the domain interface (ddd-persistence Rule 3); the package
// distinguishes, so the implements clause is fully qualified. Spring Data's verbs
// live on the package-private PaymentJpaRepository and never appear on the domain interface.
public class PaymentRepository implements com.example.payments.domain.PaymentRepository {

    private final PaymentJpaRepository jpa;

    PaymentRepository(PaymentJpaRepository jpa) { this.jpa = jpa; }

    @Override
    public Optional<Payment> findById(PaymentId id) {
        return jpa.findById(id.value())
                .map(PaymentRecord::toDomain);                   // persistence -> domain
    }

    @Override
    public Optional<Payment> findByOrderId(OrderId orderId) {
        return jpa.findByOrderId(orderId.value())
                .map(PaymentRecord::toDomain);
    }

    @Override
    public void persist(Payment payment) {
        jpa.saveAndFlush(PaymentRecord.fromDomain(payment));     // domain -> persistence, upsert
        // flush surfaces @Version conflicts here — see references/persistence.md
    }
}
```

`PaymentRecord` is the JPA-mapped mirror entity (`@Entity`, `@Table`, `@Version`,
column mappings) and it lives in this package. The `toDomain`/`fromDomain`
translation keeps ORM concerns out of `domain`. Spring wires the adapter as *the*
domain `PaymentRepository`; nothing outside `infrastructure` ever names the
adapter or the Spring Data interface. The lock-conflict translation and 409
mapping are in `references/persistence.md`.

### 3. Application service — `application`, orchestrates the use case

```java
package com.example.payments.application;

import com.example.payments.domain.*;   // domain types and interfaces — never the impls
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class AuthorizePaymentApplicationService {

    private final PaymentRepository payments;            // domain interface
    private final FraudDetectionDomainService fraud;     // domain service

    AuthorizePaymentApplicationService(PaymentRepository payments,
                                       FraudDetectionDomainService fraud) {
        this.payments = payments;
        this.fraud = fraud;
    }

    @Transactional                                       // owns the transaction boundary
    public PaymentDTO authorize(AuthorizePaymentCommand cmd) {
        Payment payment = payments.findById(cmd.paymentId())
                .orElseThrow(PaymentNotFoundException::new);
        FraudVerdict verdict = fraud.check(payment, cmd.method());
        payment.authorize(verdict);                      // aggregate enforces invariants
        payments.persist(payment);
        return PaymentDTO.of(payment);                   // DTO out — never the entity
    }
}
```

Load → call domain service → mutate aggregate → persist → map to DTO.
Transactions and DTO mapping live here and only here.

### 4. HTTP controller — `api`, depends only on `application`

```java
package com.example.payments.api;

import com.example.payments.application.AuthorizePaymentApplicationService;
import com.example.payments.application.PaymentDTO;
// never an import from ..payments.domain or ..payments.infrastructure
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/payments")
public class PaymentsController {

    private final AuthorizePaymentApplicationService authorizePayment;

    PaymentsController(AuthorizePaymentApplicationService authorizePayment) {
        this.authorizePayment = authorizePayment;
    }

    @PostMapping("/{id}/authorize")
    public ResponseEntity<PaymentDTO> authorize(@PathVariable String id,
                                                @RequestBody AuthorizeRequest body) {
        PaymentDTO dto = authorizePayment.authorize(body.toCommand(id));
        return ResponseEntity.ok(dto);
    }
}
```

The controller never sees `Payment`, the `PaymentRepository` interface, or its
infrastructure adapter. Request parsing, status codes, and the
`@RestControllerAdvice` in this package are its whole job.

## What this demonstrates

- The repository **interface** lives in `domain`; its signatures use only domain types.
- The **implementation** lives in `infrastructure` and carries all framework
  concerns — the JPA mirror entity `PaymentRecord`, the package-private Spring
  Data interface, and the translation between the mirror and the domain aggregate.
- The application service orchestrates, owns `@Transactional`, depends on the
  domain *interface*, and returns DTOs.
- The fraud check is a `DomainService`: pure domain logic, no I/O.
- The domain event is `PaymentAuthorizedEvent` — past tense, `*Event` suffix, in `domain`.
- The controller lives in `api` and imports only from `application`.

## Spring Boot idioms

- Value objects, events, and DTOs are `record`s. Aggregates are plain classes
  with state transitions as methods.
- The Spring container is the composition root: the constructor-injected domain
  `PaymentRepository` resolves to the Spring Data-backed infrastructure
  `PaymentRepository` at runtime. Single constructor — no `@Autowired`.
- Domain purity: no `jakarta.persistence.*`, `org.springframework.*` (except
  `org.springframework.stereotype` on domain services — see
  `references/services.md`), or `org.hibernate.*` imports under `<bc>.domain`.
- Enforce with **ArchUnit**: the layered-architecture rule and the domain-purity
  rule in `references/architecture-tests.md`, one test class per bounded context.
- Keep the `@SpringBootApplication` class in the base package above the bounded
  contexts; component scanning then finds every layer without configuration.
- Scaffolding order when generating a context from scratch: value objects →
  domain events → aggregate → repository interface → domain services →
  application service, commands, DTOs → mirror + Spring Data interface +
  repository adapter → controller + advice → ArchUnit test.
