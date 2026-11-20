---
name: spring-boot-tdd
description: 'Spring Boot test mechanics for the tdd-* skills — use alongside them (never instead of them) when writing or running tests. Supplies: the plain-JVM vs Spring-context tiers and what "all previous tests pass" costs per tier, the runner (focused ./mvnw or ./gradlew test runs in the loop; verify/build at the end), the first red as a javac error and the stub convention, where the [TEST] list lives, @SpringBootTest with Testcontainers @ServiceConnection, @WebMvcTest + MockMvc, @MockitoBean versus @TestConfiguration beans, @DataJpaTest, test rollback, and the *IT/Failsafe split. Trigger on @SpringBootTest, @WebMvcTest, @DataJpaTest, @MockitoBean/@MockBean, MockMvc, Testcontainers, [TEST] comments in a Java test, running the tests in a Spring Boot project, @SpringBootTest on a class with no container dependency, a mock of the class the test is named after, or an assertion on Hibernate, Jackson, or Spring-injection behavior. Defines no TDD rule (tdd-test-list, tdd-cycle); project creation is spring-boot-app.'
---

# Spring Boot TDD Mechanics

The Spring Boot tier under the `tdd-*` skills. `tdd-test-list` decides what goes on the list and in what order; `tdd-cycle` decides when a test is red, what green may change, when to refactor and commit. This skill decides **how Spring Boot runs that loop**: which tier a test belongs to, what to run at each step, how the first red looks in Java, and which test tooling to use. If a rule here appears to conflict with a `tdd-*` rule, the `tdd-*` rule wins and this file has a bug.

**Load order.** `tdd-test-list` (the list exists), `tdd-cycle` (the loop), then this file, then exactly one reference: `references/tdd-cycle.md` for the loop in Java/Spring Boot, `references/testing.md` for test tooling.

**Red Flags — STOP if you find yourself thinking:**

- "`@SpringBootTest` on a plain Java class's test." (If a class with no container dependency needs the context to be tested, it has acquired one — fix the class, not the test.)
- "Run the whole suite, context tests included, after every red." (The plain-JVM tier runs every step; the context tiers run at green and after refactoring — context caching makes the second run cheap, not the first.)
- "Mock the application service in `CancelOrderApplicationServiceTest`." (Never the class the test is named after.)
- "`@MockitoBean` *and* a `@TestConfiguration` `@Bean` for the same type, to be safe." (One mechanism per double.)
- "The pure-JUnit rule tests come later." (Later is never; the rules are the next list item on the plain JVM.)
- "I'll test the repository through the controller." (Three tiers, three classes; a failure must name its tier.)
- "`return null;` to make it compile." (Stubs throw `UnsupportedOperationException` so a stub reached by accident is a red, not a silent pass.)

---

## Stack defaults

| Concern | Default | Alternative (say why) |
| --- | --- | --- |
| Runner during the loop | One class on demand: `./mvnw test -Dtest=OrderTest` or `./gradlew test --tests OrderTest`; Gradle's `-t` (continuous) reruns on save — Spring Boot has no `quarkus test` equivalent | The IDE's auto-rerun for the focused class |
| Runner at the end | Maven: `./mvnw verify`; Gradle: `./gradlew build` — before a PR. If the project splits `*IT` onto Failsafe, `verify` is what runs them | — |
| Assertions | AssertJ (`assertThat`, `assertThatThrownBy`) — included in `spring-boot-starter-test`; Hamcrest only inside MockMvc matchers (`jsonPath(..., equalTo(...))`) | — |
| Plain-JVM tier (no container dependency) | Plain JUnit 5, no Spring annotations, no context | — |
| Container tier (beans, repositories) | `@SpringBootTest` + Testcontainers PostgreSQL via `@ServiceConnection` (the Initializr-generated `TestcontainersConfiguration`) with the **real** repository adapter; `@Transactional` on the test for per-test rollback. Repository mapping/locking detail: `@DataJpaTest` + `@Import` of the adapter | A hand-rolled in-memory fake of the repository interface when the test is about orchestration only |
| HTTP tier (controllers) | `@WebMvcTest(TheController.class)` + MockMvc; `@MockitoBean` on the application service the controller calls | — |
| Doubles | `@MockitoBean` for collaborators (never `@MockBean` — removed in Boot 4); a `@TestConfiguration` `@Bean` for a real instance you construct (a fixed `Clock`, a stateful fake); never a domain type, never the class under test | — |
| Black-box flow tests | `@SpringBootTest(webEnvironment = RANDOM_PORT)` + `TestRestTemplate`, Testcontainers behind it, state set up through the API itself | Name them `*IT` on Failsafe when the team splits test phases |
| Test dependencies | `spring-boot-starter-test` (JUnit 5, Mockito, AssertJ, MockMvc, JsonPath — automatic), `spring-boot-testcontainers` + `postgresql` (the `testcontainers` Initializr selection), `archunit-junit5` by hand — per `spring-boot-app` | — |

---

## Core mechanics

1. **Tier by dependency, not by package.** A test runs on the plain JVM — JUnit 5, no Spring annotations — when the class under test needs no injection, database, or HTTP: domain objects, pure logic, mappers. It needs a Spring context when the class needs the container: beans with injected collaborators and repositories under `@SpringBootTest`/`@DataJpaTest` against Testcontainers, controllers under `@WebMvcTest`. One test class per production class, in the matching test package.
2. **What "all previous tests pass" (`tdd-cycle` Rule 7) costs per tier.** The plain-JVM tier runs at every red and green — milliseconds. The context tiers run at every green commit and after each refactoring; Spring's context caching means the JVM boots the context once per unique configuration, so keep test-class configurations uniform (same `TestcontainersConfiguration`, same override set) and the suite stays one context. `verify`/`build` runs before the PR, not inside the loop.
3. **The first red is a compile error.** `tdd-cycle` Rule 5's two steps in Java: the test names a class or method that doesn't exist → `javac` fails → add the type or method with a body of `throw new UnsupportedOperationException();` (never `return null`) → the AssertJ failure is the second red. Constructors and records that must exist get their parameters and nothing else.
4. **The list lives at the bottom of the test class** as `// [TEST] ...` comments, in list order; the line is deleted at the green commit that completes it (`tdd-test-list` Rule 7), so during red the item is still on the list. Items discovered while going green are appended there (`tdd-test-list` Rule 6).
5. **Test names are snake_case sentences that state the behavior**: `cancelling_a_fulfilled_order_is_refused`. The class is `<ProductionClass>Test`; no `should`, no `test` prefix.
6. **`@MockitoBean` is the default double.** Declare the field, stub in `@BeforeEach` with `Mockito.when(...)`. A `@TestConfiguration` `@Bean` instead when you must construct the instance yourself (a fixed `Clock`, a stateful fake). Never both for one type. Per-test variation is re-stubbing the same mock inside the test method, not swapping instances — Spring has no per-method bean swap; a genuinely different wiring is a different test class. Every distinct override set (each combination of `@MockitoBean` fields and test configurations) is a distinct cached context: overriding freely is how a suite ends up booting ten contexts.
7. **Never mock the class the test is named after, nor a plain domain object.** Entities, value types, and pure-logic classes are exercised as real instances; mock collaborators only — another application service, an external client, a `Clock`.
8. **MockMvc: status first, then JSON paths.** `mvc.perform(post(...)).andExpect(status().isOk()).andExpect(jsonPath("$.status").value("CANCELLED"))`. The path in the test is the path the controller maps; fix the test, not the controller. Never whole-body string comparison.
9. **Repository tests use Testcontainers and rollback.** Real PostgreSQL via `@ServiceConnection`; `@DataJpaTest` rolls back per test by default, and a `@SpringBootTest` gets the same from `@Transactional` on the test. Inject the domain interface, not the Spring Data interface. One shared `TestcontainersConfiguration` — never a second container beside it. (A transactional test never observes `AFTER_COMMIT` events — there is no commit; test event delivery without test-managed transactions, per `references/testing.md`.)
10. **Black-box flow tests run the real HTTP stack.** `@SpringBootTest(webEnvironment = RANDOM_PORT)` + `TestRestTemplate`, nothing mocked, state set up through the API. There is no packaged-artifact test annotation in Spring Boot and no `*IT extends *Test` inheritance trick; if the team separates phases, these classes are named `*IT` and run by the Failsafe plugin at `verify` (configured by hand — Initializr does not add it), otherwise they live in the normal suite.
11. **Don't test the framework.** No test that Hibernate persists a field, Jackson serializes a record, or Spring injects a bean. The mapping, the lock, and the atomic update are yours and get tests; the ORM is not.
12. **Commits follow `tdd-cycle` Rule 12 with `git`**: at green, `feat:`/`fix:` naming the behavior; after refactoring, `refactor:` naming the move. Commit the cycle's files by path (`git commit -m "..." -- <paths>`), never `-a`, so unrelated changes in the tree are left alone. In human mode the red step is `test: <behavior>` with the failing test alone, so the history shows the test came first.

---

## Reference files

Load exactly one, for the concern at hand.

| Doing… | Read | Pairs with |
| --- | --- | --- |
| The loop in Java/Spring Boot: a `[TEST]` list, a plain-JVM cycle, a `@SpringBootTest` cycle, the commits, a "passed without red" callout | `references/tdd-cycle.md` | `tdd-cycle`, `tdd-test-list` |
| A context test: doubles, MockMvc, Testcontainers, rollback, the black-box flow test, context caching | `references/testing.md` | `tdd-cycle` Rule 13 |

---

## Quick Reference

- [ ] Each `[TEST]` item has a tier, and the test class sits in the matching package.
- [ ] The plain-JVM tier is green before every commit; context tiers are green at every green commit.
- [ ] A new class or method was first seen as a `javac` failure; stubs throw `UnsupportedOperationException`.
- [ ] The `[TEST]` list is at the bottom of the test class and shrinks at each green commit.
- [ ] Test names are snake_case sentences; classes are `<Production>Test`; black-box flow tests use `RANDOM_PORT` + `TestRestTemplate`.
- [ ] Doubles: `@MockitoBean` on collaborators only; no mock of the class under test or of a domain type; no type doubled two ways; override sets kept uniform so contexts stay cached.
- [ ] Repository tests: real Testcontainers PostgreSQL via the shared `TestcontainersConfiguration`, rollback per test, domain interface injected.
- [ ] MockMvc asserts status, then JSON paths, at the controller's declared mapping.
- [ ] Nothing asserts Hibernate, Jackson, or Spring-injection behavior.

### Companion skills

- **`tdd-test-list`, `tdd-cycle`** — the process this skill runs. Written against `tdd-skills` 1.x; Core Rule numbers cited here and in `references/` refer to that version.
- **`spring-boot-app`** — the project, the dependencies, the test starters.
- **`spring-boot-ddd`** — where each production class lives; its test tiers and this skill's are the same tiers.
