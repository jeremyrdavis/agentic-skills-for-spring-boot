# Canonical Example — ArchUnit Layer Test for a Bounded Context (Java / Spring Boot)

Load this file when adding the boundary test to a bounded context. The layering
rules are `ddd-foundations` Rules 1–8 and 12; this test makes them fail the
build instead of a review. One test class per bounded context.

## Dependency (test scope)

```xml
<dependency>
    <groupId>com.tngtech.archunit</groupId>
    <artifactId>archunit-junit5</artifactId>
    <version><!-- current release --></version>
    <scope>test</scope>
</dependency>
```

Plain JUnit 5 — no `@SpringBootTest`; it reads class files, not a running app.
(Not an Initializr dependency — add it to the build file by hand; ArchUnit
versions are not managed by the Spring Boot BOM, so this one does carry a
`<version>` tag.)

## The test

```java
package com.example.orders;

import com.tngtech.archunit.core.importer.ImportOption;
import com.tngtech.archunit.junit.AnalyzeClasses;
import com.tngtech.archunit.junit.ArchTest;
import com.tngtech.archunit.lang.ArchRule;
import org.springframework.transaction.annotation.Transactional;

import static com.tngtech.archunit.base.DescribedPredicate.not;
import static com.tngtech.archunit.core.domain.JavaClass.Predicates.resideInAPackage;
import static com.tngtech.archunit.core.domain.JavaClass.Predicates.resideInAnyPackage;
import static com.tngtech.archunit.core.domain.JavaClass.Predicates.simpleNameEndingWith;
import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.*;
import static com.tngtech.archunit.library.Architectures.layeredArchitecture;

@AnalyzeClasses(packages = "com.example.orders", importOptions = ImportOption.DoNotIncludeTests.class)
class OrdersArchitectureTest {

    // ddd-foundations Rules 1–6: four layers, arrows point inward only.
    // Infrastructure may call Application: inbound adapters (event listeners, Kafka consumers) invoke use cases.
    // The one tolerated api → domain edge: the @RestControllerAdvice names the domain exceptions it maps
    // (ddd-foundations placement table; ddd-persistence Rule 7's 409 mapping).
    @ArchTest
    static final ArchRule layers = layeredArchitecture().consideringOnlyDependenciesInLayers()
            .ignoreDependency(resideInAPackage("..orders.api.."), simpleNameEndingWith("Exception"))
            .layer("Domain").definedBy("..orders.domain..")
            .layer("Application").definedBy("..orders.application..")
            .layer("Infrastructure").definedBy("..orders.infrastructure..")
            .layer("Api").definedBy("..orders.api..")
            .whereLayer("Api").mayNotBeAccessedByAnyLayer()
            .whereLayer("Infrastructure").mayNotBeAccessedByAnyLayer()
            .whereLayer("Application").mayOnlyBeAccessedByLayers("Api", "Infrastructure")
            .whereLayer("Domain").mayOnlyBeAccessedByLayers("Application", "Infrastructure");

    // ddd-foundations Rule 2: domain depends on nothing in your code — and no framework.
    // org.springframework.stereotype is tolerated: domain services are @Service (ddd-services Rule 9).
    // Everything else Spring — tx, data, web, context, kafka — stays out, so the forbidden list names
    // the broad roots and the stereotype exception is carved out by NOT listing it.
    @ArchTest
    static final ArchRule domainIsPlain = noClasses().that().resideInAPackage("..orders.domain..")
            .should().dependOnClassesThat().resideInAnyPackage(
                    "jakarta.persistence..", "jakarta.transaction..",
                    "org.springframework.transaction..", "org.springframework.data..",
                    "org.springframework.web..", "org.springframework.context..",
                    "org.springframework.kafka..", "org.springframework.boot..",
                    "org.hibernate..");

    // ddd-foundations Rule 12: controllers see application only — the same advice carve-out as above.
    @ArchTest
    static final ArchRule apiSeesApplicationOnly = noClasses().that().resideInAPackage("..orders.api..")
            .should().dependOnClassesThat(resideInAnyPackage("..orders.domain..", "..orders.infrastructure..")
                    .and(not(simpleNameEndingWith("Exception"))));

    // spring-boot-ddd mechanic 1: the transaction boundary is the application service, nowhere else.
    // Spring's annotation can sit on methods or classes — check both.
    @ArchTest
    static final ArchRule transactionalMethodsLiveInApplication = methods().that().areAnnotatedWith(Transactional.class)
            .should().beDeclaredInClassesThat().resideInAPackage("..orders.application..")
            .orShould().beDeclaredInClassesThat().haveSimpleNameEndingWith("Relay");   // the outbox relay — see domain-events.md

    @ArchTest
    static final ArchRule transactionalClassesLiveInApplication = classes().that().areAnnotatedWith(Transactional.class)
            .should().resideInAPackage("..orders.application..")
            .allowEmptyShould(true);   // the convention is method-level, so this selection is usually empty

    // Only Spring's @Transactional, ever — mixing in the Jakarta one splits the semantics.
    @ArchTest
    static final ArchRule noJakartaTransactional = noClasses()
            .should().dependOnClassesThat().resideInAPackage("jakarta.transaction..");

    // ddd-foundations Rules 9–11: the suffix and the package agree; no bare *Service.
    @ArchTest
    static final ArchRule applicationServicesInApplication = classes().that().haveSimpleNameEndingWith("ApplicationService")
            .should().resideInAPackage("..orders.application..");

    // allowEmptyShould: ArchUnit fails a rule whose selection is empty, and a context with no
    // domain service — or no misnamed service — is a valid context.
    @ArchTest
    static final ArchRule domainServicesInDomain = classes().that().haveSimpleNameEndingWith("DomainService")
            .should().resideInAPackage("..orders.domain..")
            .allowEmptyShould(true);

    @ArchTest
    static final ArchRule noUnqualifiedService = noClasses().that().haveSimpleNameEndingWith("Service")
            .and().haveSimpleNameNotEndingWith("ApplicationService")
            .and().haveSimpleNameNotEndingWith("DomainService")
            .should().resideInAPackage("..orders..")
            .allowEmptyShould(true);

    // ddd-foundations Rule 7 / ddd-persistence Rule 3: the interface is domain, Spring Data is infrastructure.
    @ArchTest
    static final ArchRule springDataStaysInInfrastructure = noClasses().that().resideOutsideOfPackage("..orders.infrastructure..")
            .should().dependOnClassesThat().resideInAPackage("org.springframework.data..");
}
```

## Notes

- `consideringOnlyDependenciesInLayers()` ignores JDK and library dependencies
  in the layer rule; the separate `domainIsPlain` rule is what keeps the
  framework out of `domain`. Forbidding `org.springframework.transaction..`,
  `..data..`, `..web..`, `..context..`, `..kafka..`, and `..boot..` while
  leaving `org.springframework.stereotype` unlisted is the carve-out for
  `@Service` on domain services.
- The package-private Spring Data interface (`OrderJpaRepository`) satisfies
  `springDataStaysInInfrastructure` by construction — Java's access control and
  the ArchUnit rule enforce the same boundary twice, cheaply.
- `*Exception` is the only domain name `api` may touch, and only so the
  `@RestControllerAdvice` can map it; the carve-out is by suffix, so keep
  domain exceptions named `*Exception`.
- If the api layer needs a typed ID in a signature, convert at the edge
  (`new OrderId(UUID.fromString(id))` happens in the application service's
  command constructor, not in the controller) — keep `apiSeesApplicationOnly`
  green rather than weakening it.
- One class per bounded context; for several contexts in one module, add a
  rule that `..orders..` and `..payments..` do not depend on each other's
  `domain` packages — cross-context data flows through integration events.
  (Spring Modulith's `ApplicationModules.verify()` enforces exactly this
  cross-context rule out of the box if the project adopts Modulith; the
  per-layer rules above stay either way.)
