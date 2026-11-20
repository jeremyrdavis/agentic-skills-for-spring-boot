# agentic-skills-for-spring-boot

Claude Code skills for building Spring Boot applications. A port of
[agentic-skills-for-quarkus](https://github.com/jeremyrdavis/agentic-skills-for-quarkus) to the Spring Boot
idiom — the same topics, realized the way regular Spring Boot users work, not Quarkus conventions transposed.

Three skills, three jobs:

- **`spring-boot-app`** — bootstrap and maintain a Spring Boot project: Spring Boot version, build tool, starters and BOMs, Spring Initializr, enterprise support.
- **`spring-boot-ddd`** — the Spring Boot mechanics tier under the [`ddd-*` skills](https://github.com/jeremyrdavis/agentic-skills-for-ddd): how Spring Boot realizes the decisions those skills make.
- **`spring-boot-tdd`** — the Spring Boot mechanics tier under the [`tdd-*` skills](https://github.com/jeremyrdavis/agentic-skills-for-tdd): how Spring Boot runs the red–green–refactor loop.

## Available skills

### `spring-boot-app`

Use when creating or maintaining a Spring Boot application. It covers generating a new project through Spring Initializr (curl, `spring init`, or the web UI), choosing the build tool and Java version, selecting official starters with BOM-managed versions (never a hand-pinned `<version>`), adding dependencies to an existing project, upgrading Spring Boot with OpenRewrite recipes, and Tanzu Spring enterprise support. It does not generate domain code.

### `spring-boot-ddd`

Use **alongside** the `ddd-*` skills when implementing DDD patterns in a Spring Boot project. The DDD skills decide the layers, the aggregate shape, the repository contract, who owns the transaction, and when events are delivered. `spring-boot-ddd` supplies the Spring Boot side of each decision:

- package layout mirroring the four `ddd-foundations` layers, enforced by an ArchUnit test;
- the Initializr dependency bundle for a DDD-shaped service;
- Spring's `@Transactional` on application-service methods and nowhere else;
- the repository adapter implementing the hand-written domain interface over a package-private Spring Data JPA repository; `@Version` compare-and-set (`Long`, null when new) and its mapping to 409 via `@RestControllerAdvice` + `ProblemDetail`;
- after-commit event delivery with `@TransactionalEventListener`, and a transactional outbox to Kafka for events that leave the service (Spring Modulith noted as the packaged alternative);
- test tiers: plain JUnit for `domain`, `@SpringBootTest`/`@DataJpaTest` with Testcontainers for `application` and `infrastructure`, `@WebMvcTest` for `api`.

It defines no DDD concepts of its own. Its `references/` directory holds the Spring Boot worked examples that the `ddd-*` skills point to: `foundations.md`, `services.md`, `persistence.md`, `domain-events.md`, `architecture-tests.md`.

### `spring-boot-tdd`

Use **alongside** the `tdd-*` skills when writing or running tests in a Spring Boot project. `tdd-test-list` decides what goes on the test list and in what order; `tdd-cycle` decides when a test is red, what green may change, when to refactor and commit. `spring-boot-tdd` supplies the Spring Boot side:

- which tests run on the plain JVM and which need a Spring context, what "all previous tests pass" costs per tier, and what context caching makes of that cost;
- the runner: focused `./mvnw test -Dtest=X` or `./gradlew test --tests X` during the loop (Gradle `-t` for rerun-on-save); `verify`/`build` before a PR;
- the first red as a `javac` error and the stub convention (`throw new UnsupportedOperationException()`);
- where the `[TEST]` list lives and how test classes and methods are named;
- `@MockitoBean` versus `@TestConfiguration` beans, MockMvc, Testcontainers `@ServiceConnection`, transactional-test rollback (and why it never observes `AFTER_COMMIT`), black-box flow tests on `RANDOM_PORT`, the `*IT`/Failsafe split, and what never to mock.

It defines no TDD rule of its own. Its `references/` directory holds `tdd-cycle.md` (one worked loop adding `cancel()` to an `Order`) and `testing.md` (the test tooling).

## Install

There is no `claude mcp add-skill` command. Use one of the following.

### Plugin, per project (recommended)

Install into each Spring Boot project rather than user-wide, so the descriptions are only in context where they apply. `spring-boot-ddd` needs the `ddd-skills` plugin installed in the same project; `spring-boot-tdd` needs `tdd-skills`.

```bash
claude plugin marketplace add jeremyrdavis/agentic-skills-for-ddd
claude plugin marketplace add jeremyrdavis/agentic-skills-for-tdd
claude plugin marketplace add jeremyrdavis/agentic-skills-for-spring-boot
cd <your-spring-boot-project>
claude plugin install ddd-skills@agentic-skills-for-ddd --scope project
claude plugin install tdd-skills@agentic-skills-for-tdd --scope project
claude plugin install spring-boot-skills@agentic-skills-for-spring-boot --scope project
```

One-step form from inside a Claude Code session:

```
/plugin install spring-boot-skills --marketplace jeremyrdavis/agentic-skills-for-spring-boot
```

A Spring Boot project that doesn't use DDD or TDD can install `spring-boot-skills` alone; it carries the unused `spring-boot-ddd` and `spring-boot-tdd` descriptions, which cost a few hundred tokens per session each and nothing else.

### Copy, per project (no manifests)

```bash
git clone https://github.com/jeremyrdavis/agentic-skills-for-spring-boot.git
mkdir -p <your-project>/.claude/skills
cp -r agentic-skills-for-spring-boot/skills/* <your-project>/.claude/skills/
```

## How the skills work together

For a new service: `spring-boot-app` generates the project through Spring Initializr with `web, data-jpa, postgresql, validation, testcontainers` (plus `kafka` if events leave the service). Then the `ddd-*` skill for the concern at hand loads (for example `ddd-aggregates` for a new aggregate), `spring-boot-ddd` loads with it, and the assistant reads the one Spring Boot reference file for that concern. With the `tdd-*` skills installed, `tdd-test-list` writes the `[TEST]` list first, `tdd-cycle` runs the loop, and `spring-boot-tdd` says which tests need a Spring context and what to run at each step — the same loop whatever architecture the project follows, DDD or not.

Example prompts:

- "Create a new Spring Boot REST API with PostgreSQL persistence." → `spring-boot-app`
- "Create an orders bounded context with an `Order` aggregate." → `ddd-foundations`, `ddd-aggregates`, `spring-boot-ddd`
- "Add a repository for `Order` with optimistic locking." → `ddd-persistence`, `spring-boot-ddd`
- "Publish `OrderCancelledEvent` to Kafka." → `ddd-domain-events`, `spring-boot-ddd`
- "Add `cancel` to `Order`, test-first." → `tdd-test-list`, `tdd-cycle`, `ddd-aggregates`, `spring-boot-ddd`, `spring-boot-tdd`
- "Write a MockMvc test for the orders controller." → `spring-boot-tdd` (and `tdd-cycle` if the controller is being built)

## Using the skills with Codex

Codex can use the same files as guidance. Point it at the repository or copy the relevant `skills/*/SKILL.md` into its context. For DDD work, give it the matching `ddd-*` skill as well, and for TDD the `tdd-*` skills; `spring-boot-ddd` and `spring-boot-tdd` on their own are deliberately incomplete.

## Repository structure

```text
.
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
└── skills/
    ├── spring-boot-app/
    │   ├── SKILL.md
    │   └── references/            (starters.md, enterprise-support.md)
    ├── spring-boot-ddd/
    │   ├── SKILL.md
    │   └── references/            (foundations, services, persistence, domain-events, architecture-tests)
    └── spring-boot-tdd/
        ├── SKILL.md
        └── references/            (tdd-cycle, testing)
```

## Prerequisites

- Java 25 or your preferred version
- Maven or Gradle
- Claude Code or Codex, depending on your workflow

## Contributing

Keep `spring-boot-app` about the project, `spring-boot-ddd` about DDD mechanics, and `spring-boot-tdd` about test mechanics. If a change to `spring-boot-ddd` would redefine a DDD concept, it belongs in the `ddd-*` repository; if a change to `spring-boot-tdd` would redefine a TDD rule, it belongs in the `tdd-*` repository. The `references/*.md` files cite `ddd-skills` and `tdd-skills` Core Rule numbers; they are written against the `1.x` versions.
