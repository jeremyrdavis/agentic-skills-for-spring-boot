# Spring Boot Test Tooling — Doubles, MockMvc, Testcontainers, Black-Box Flow Tests

Load this file when writing a context test, choosing a mocking mechanism,
hitting an endpoint with MockMvc, or adding a black-box flow test. The
mechanics are numbered in `spring-boot-tdd/SKILL.md` (6–11); this file is the
worked form. The loop itself is `tdd-cycle`; the test's place in it is
unchanged by anything here.

**The principle behind every example:** if you mock the system under test, you
are not testing — you are scaffolding. A class named `XTest` that mocks `X`
asserts only what Mockito returned; the rules inside `X` go untested. Mock
*collaborators* (another application service, an external client, a clock);
never the class the test is named after, never a plain domain object.

## 1. Controller test — HTTP tier, `@WebMvcTest` + `@MockitoBean` on the service behind it

The controller's job is request parsing, status codes, and the
`@RestControllerAdvice`. The use case behind it has its own tests; here it is
a collaborator, so it is mocked (mechanic 6), and the mock variable is named
after the type. `@WebMvcTest` boots only the web slice — this controller, the
advices, Jackson — not the full context.

```java
package com.example.orders.api;

import com.example.orders.application.CancelOrderApplicationService;
import com.example.orders.application.OrderDTO;
import com.example.orders.domain.OrderId;
import com.example.orders.domain.OrderNotFoundException;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.web.servlet.MockMvc;

import java.math.BigDecimal;
import java.util.UUID;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@WebMvcTest(OrdersController.class)
class OrdersControllerTest {

    static final UUID KNOWN = UUID.fromString("0b5f6a3e-8d1c-4b7a-9c2e-1f3d5a7b9c0d");
    static final UUID UNKNOWN = UUID.fromString("ffffffff-ffff-ffff-ffff-ffffffffffff");

    @Autowired
    MockMvc mvc;

    @MockitoBean
    CancelOrderApplicationService cancelOrder;                      // collaborator — mocked

    @BeforeEach
    void stubUseCase() {
        Mockito.when(cancelOrder.cancel(new OrderId(KNOWN)))
               .thenReturn(new OrderDTO(KNOWN, UUID.randomUUID(), "CANCELLED", new BigDecimal("19.98"), "USD"));
        Mockito.when(cancelOrder.cancel(new OrderId(UNKNOWN)))
               .thenThrow(new OrderNotFoundException(new OrderId(UNKNOWN)));
    }

    @Test
    void cancelling_a_known_order_returns_the_cancelled_order() throws Exception {
        mvc.perform(post("/orders/{id}/cancel", KNOWN))             // path matches the @PostMapping on OrdersController
           .andExpect(status().isOk())                              // status first
           .andExpect(jsonPath("$.status").value("CANCELLED"));     // then JSON paths, never whole-body strings
    }

    @Test
    void cancelling_an_unknown_order_returns_404() throws Exception {
        mvc.perform(post("/orders/{id}/cancel", UNKNOWN))
           .andExpect(status().isNotFound());                       // the @RestControllerAdvice's job
    }
}
```

Per-argument behavior (`KNOWN` succeeds, `UNKNOWN` throws) is ordinary
`Mockito.when(...)` stubbing in `@BeforeEach`. `@MockitoBean` is the current
annotation — `@MockBean` was deprecated in Spring Boot 3.4 and is gone in
Boot 4; a codebase showing `@MockBean` is on the old one, not a different
mechanism.

## 2. Black-box flow test — the real HTTP stack, nothing mocked

Spring Boot has no packaged-artifact test annotation and no `*IT extends
*Test` companion idiom: there is no second process, so there is nothing a
companion would re-run differently. The black-box tier is instead a
`@SpringBootTest` on a random port, driven purely through HTTP —
state set up through the API itself, Testcontainers behind it, no `@Autowired`
application beans, no mocks, no test-managed transaction:

```java
package com.example.orders.api;

import com.example.TestcontainersConfiguration;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.context.annotation.Import;
import org.springframework.http.ResponseEntity;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.boot.test.context.SpringBootTest.WebEnvironment.RANDOM_PORT;

@SpringBootTest(webEnvironment = RANDOM_PORT)
@Import(TestcontainersConfiguration.class)
class OrdersFlowTest {                                              // black-box: no mocks, no repository access

    @Autowired
    TestRestTemplate rest;                                          // pre-wired against the random port

    static final String PLACE_ORDER = """
        {"customerId": "c-1", "lines": [{"sku": "A", "quantity": 1, "unitPrice": 10.00}]}
        """;

    @Test
    void an_order_placed_through_the_api_can_be_cancelled() {
        ResponseEntity<Map> placed = rest.postForEntity("/orders", json(PLACE_ORDER), Map.class);
        assertThat(placed.getStatusCode().value()).isEqualTo(201);
        String id = (String) placed.getBody().get("id");

        ResponseEntity<Map> cancelled = rest.postForEntity("/orders/{id}/cancel", null, Map.class, id);
        assertThat(cancelled.getStatusCode().value()).isEqualTo(200);
        assertThat(cancelled.getBody().get("status")).isEqualTo("CANCELLED");
    }
}
```

By default this class runs with the rest of the suite under `./mvnw test` /
`./gradlew test` — that is the mainstream Spring Boot setup, and `verify` /
`build` before a PR covers it. Teams that want a separate integration phase
name such classes `*IT` and add the Maven Failsafe plugin (which picks up
`*IT` at `verify`); Initializr does not configure Failsafe, so if `*IT`
classes exist, check the `pom.xml` for it — `*IT` without Failsafe is a suite
that silently never runs.

## 3. Repository test — container tier, real Testcontainers PostgreSQL

The mapping, the lock column, and the atomic update are what this tier is
for, so nothing is mocked. The test injects the repository **interface**; the
adapter is `@Import`ed because `@DataJpaTest` boots only the JPA slice
(Spring Data interfaces, entities, the `EntityManager`) and does not scan
`@Component` adapters:

```java
package com.example.orders.infrastructure;

import com.example.orders.domain.*;
import com.example.TestcontainersConfiguration;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.context.annotation.Import;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.Currency;
import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest                                                         // JPA slice; each test rolls back by default
@Import({TestcontainersConfiguration.class, OrderRepository.class})  // the real container + the adapter under test
class OrderRepositoryTest {

    @Autowired
    OrderRepository orders;                                          // com.example.orders.domain.OrderRepository

    @Test
    void persisting_then_loading_returns_an_equal_order() {
        Money unit = new Money(new BigDecimal("9.99"), Currency.getInstance("USD"));
        OrderId id = new OrderId(UUID.randomUUID());
        Order order = Order.place(id, new CustomerId(UUID.randomUUID()),
                                  List.of(new OrderLine("sku-1", 2, unit)),
                                  Instant.parse("2026-11-13T10:00:00Z"));

        orders.persist(order);
        Order found = orders.findById(id).orElseThrow();

        assertThat(found.status()).isEqualTo(Order.Status.PLACED);
        assertThat(found.total()).isEqualTo(new Money(new BigDecimal("19.98"), Currency.getInstance("USD")));
        assertThat(found.version()).isEqualTo(0L);                    // fresh row; the lock cursor starts at 0
    }
}
```

The container comes from the one shared `TestcontainersConfiguration`
(`@Bean @ServiceConnection PostgreSQLContainer<?>` — Initializr generates it
when `testcontainers` + a driver are selected); with a `@ServiceConnection`
datasource present, the slice's embedded-database replacement backs off, so
the test runs against real PostgreSQL — H2 would vacuously pass the lock and
atomic-update tests. A second, hand-rolled `@Container` with
`@DynamicPropertySource` beside it means two databases and flaky tests
(mechanic 9).

## 4. Constructed doubles, per-test variation, and what overriding costs

`@MockitoBean` is the default (mechanic 6). A `@TestConfiguration` `@Bean` is
the mechanism when you must construct the instance yourself — a fixed `Clock`,
a hand-rolled stateful fake of the repository interface. Both replace the bean
for *every* injection point in the test's context; there is no
field-only mock and no per-method bean swap in Spring.

Per-test *behavior* is re-stubbing; the `Clock` case shows the shape:

```java
package com.example.orders.api;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;
import org.springframework.test.context.bean.override.mockito.MockitoBean;

import java.time.Clock;
import java.time.Instant;
import java.time.ZoneOffset;

// inside a @WebMvcTest or @SpringBootTest class:

    @MockitoBean
    Clock clock;                                                     // replaces the app's Clock bean everywhere

    @BeforeEach
    void defaultTime() {
        Mockito.when(clock.instant()).thenReturn(Instant.parse("2026-11-13T10:00:00Z"));
        Mockito.when(clock.getZone()).thenReturn(ZoneOffset.UTC);
    }

    @Test
    void an_order_is_active_at_the_default_time() throws Exception {
        mvc.perform(get("/orders/{id}", KNOWN)).andExpect(status().isOk());
    }

    @Test
    void an_order_has_expired_a_year_later() throws Exception {
        Mockito.when(clock.instant()).thenReturn(Instant.parse("2027-11-13T10:00:00Z"));  // re-stub, don't reinstall

        mvc.perform(get("/orders/{id}", KNOWN)).andExpect(status().isGone());
    }
```

If the double should be a *real* fixed `Clock` rather than a mock, declare it
once where the whole suite can share it:

```java
@TestConfiguration(proxyBeanMethods = false)
public class FixedClockConfiguration {
    @Bean
    Clock clock() { return Clock.fixed(Instant.parse("2026-11-13T10:00:00Z"), ZoneOffset.UTC); }
}
```

and `@Import(FixedClockConfiguration.class)` — but then a different time means
a different context, so a test that needs to *vary* time wants the mock above,
not a second configuration.

**What overriding costs (mechanic 6):** Spring caches application contexts
across test classes keyed on their configuration — and every `@MockitoBean`
field and every `@Import`ed test configuration is part of that key. Two
`@SpringBootTest` classes with different override sets boot two contexts.
That is the Spring version of Quarkus's "mocks are global": the mock is *not*
global across classes (each context has its own), but each new combination is
another container boot. Keep override sets uniform — the same
`TestcontainersConfiguration` everywhere, mocks declared consistently — and
the suite runs on a handful of cached contexts.

**Events and rollback (mechanic 9):** a test-managed `@Transactional`
transaction is rolled back, so `@TransactionalEventListener(AFTER_COMMIT)`
listeners never fire inside such a test. Assert the *publish* with
`@RecordApplicationEvents` + an injected `ApplicationEvents`
(`events.stream(OrderCancelledEvent.class)`), and test listener side effects
in a non-transactional test that cleans up its own data.

## Anti-patterns

| Don't | Why it's wrong | Fix |
| --- | --- | --- |
| `@MockitoBean CancelOrderApplicationService cancelOrder;` *and* a `@TestConfiguration` `@Bean` for the same type | Two mechanisms define one bean; the override rules pick one and the other is dead code that misleads the next reader. | Pick `@MockitoBean`, stub in `@BeforeEach`, delete the `@Bean` (mechanic 6). |
| A fresh `@TestConfiguration` per test method's scenario | Each configuration forks another cached context; the suite boots N containers to vary one value. | Re-stub the one mock inside the test method; reserve `@TestConfiguration` for a constructed instance the whole class shares. |
| `@SpringBootTest class CancelOrderApplicationServiceTest` with `@MockitoBean CancelOrderApplicationService` | The test is named after the class it mocks; it asserts its own stubs. | Exercise the real service against Testcontainers or a hand-rolled in-memory repository fake (mechanic 7). |
| Mocking `Order`, `Money`, or `OrderLine` | The rules inside the object are the thing under test; a mock bypasses them. | Real instances through `Order.place(...)` (mechanic 7). |
| `CancelOrderApplicationService mockRepository = Mockito.mock(...)` | The variable name contradicts the type; the layer confusion the names exist to prevent. | Name the variable after the type: `cancelOrder`. |
| Test hits `/api/orders` while the controller maps `@RequestMapping("/orders")` | The test asserts a route that doesn't exist; its failure is uninformative. | Match the controller's mapping (mechanic 8). |
| A context test asserting that a field was persisted by Hibernate or serialized by Jackson | Tests the framework, not your code. | Assert the behavior your service exposes (mechanic 11). |
| `@SpringBootTest` where `@WebMvcTest` or `@DataJpaTest` answers the question | The full context boots everything to test one slice; slow for no information. | The matching slice; `@SpringBootTest` for wiring and flow tests. |
| `*IT`-named classes with no Failsafe plugin in the `pom.xml` | Surefire ignores `*IT`, so the suite exists and never runs. | Add Failsafe, or keep the classes `*Test`-named in the normal suite (mechanic 10). |
| A static `@Container` + `@DynamicPropertySource` beside the shared `TestcontainersConfiguration` | Two databases compete; slow and flaky. | The one `@ServiceConnection` container (mechanic 9). |
| Asserting `AFTER_COMMIT` listener effects inside a `@Transactional` test | The transaction rolls back; the commit phase never runs — the test is green or red for the wrong reason. | `@RecordApplicationEvents` for the publish; a non-transactional test for the listener (mechanic 9). |
| `content().string("{...}")` on a JSON response | Whole-body string compare; brittle to whitespace and field order. | JSON paths: `jsonPath("$.status").value("CANCELLED")` (mechanic 8). |

## Excuse / Reality

| Excuse | Reality |
| --- | --- |
| "A senior reviewer told me to declare the mock as both `@MockitoBean` and a `@TestConfiguration` bean — that's authority." | Cargo-culted defenses against an unspecified ordering problem aren't authority; they're folklore. If an override rule misbehaves, name it and file the bug. Otherwise: one source of truth per double. |
| "Belt-and-braces never hurts — define the double both ways." | They define the same bean through different mechanisms. The override rules pick one; the other is dead code the next author has to decode. Tests are read more than written. |
| "Each scenario needs its own test configuration, so my per-scenario `@Import` fits." | A configuration per scenario is a context per scenario. Different return or throw values per argument is `Mockito.when(...)` — in `@BeforeEach` for the default, in the test method for the exception. If the configuration exists to vary one stubbed value, it is the wrong tool. |
| "The `@SpringBootTest` just verifies the wiring; the real rules get pure-JUnit tests later." | "Later" codifies the absence. A context test named `XTest` that mocks `X` is a tautology, and the rules in `X` stay untested until someone notices in production. If the rules deserve unit tests, write them now, on the plain JVM, as the next list item. |

---

Ported from `quarkus-tdd`'s testing reference, re-expressed in the Spring Boot
test idiom (slices, bean overrides, Testcontainers service connections).
