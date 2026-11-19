# Spring Boot Dependency Reference

Quick lookup for commonly requested dependencies. Use the **Initializr ID** in `curl -d dependencies=` or
`spring init --dependencies=`. In an existing build file, use the Maven artifact (groupId
`org.springframework.boot` unless noted) with **no version tag** — the BOM manages it.

IDs verified against start.spring.io metadata for Spring Boot 4.1. When in doubt, the Initializr metadata is
the source of truth, not this table:

```bash
curl -s https://start.spring.io/metadata/client -H 'Accept: application/json'
```

**(SC)** = has Service Connection support: Spring Boot auto-configures the connection from a container
started by Docker Compose support (`compose.yaml` on run) or Testcontainers (`@ServiceConnection` in tests).

---

## REST / Web

Spring Boot 4 modularized the starters — one starter per technology. The old names (e.g.
`spring-boot-starter-web`) still resolve for migration purposes, but new projects get the modular names below.

| Initializr ID | Artifact | Description |
|---|---|---|
| `web` | `spring-boot-starter-webmvc` | Spring MVC + embedded Tomcat + Jackson (preferred for new apps; was `spring-boot-starter-web` before Boot 4) |
| `webflux` | `spring-boot-starter-webflux` | Reactive web stack (Netty + Reactor) |
| `spring-restclient` | `spring-boot-starter-restclient` | Blocking HTTP clients (`RestClient`, HTTP service clients) without a server stack |
| `graphql` | `spring-boot-starter-graphql` | Spring for GraphQL |
| `data-rest` | `spring-boot-starter-data-rest` | Expose Spring Data repositories as REST |
| `hateoas` | `spring-boot-starter-hateoas` | Hypermedia-driven REST |
| `websocket` | `spring-boot-starter-websocket` | WebSocket + STOMP messaging |
| `springdoc-openapi` | `org.springdoc:springdoc-openapi-starter-webmvc-ui` | OpenAPI 3 / Swagger UI (`/swagger-ui.html`) |
| `validation` | `spring-boot-starter-validation` | Bean Validation (Hibernate Validator) |

---

## Data / Persistence

| Initializr ID | Artifact | Description |
|---|---|---|
| `data-jpa` | `spring-boot-starter-data-jpa` | Spring Data JPA + Hibernate (standard choice) |
| `data-jdbc` | `spring-boot-starter-data-jdbc` | Spring Data JDBC (simpler, aggregate-oriented — good DDD fit) |
| `jdbc` | `spring-boot-starter-jdbc` | Plain `JdbcTemplate`/`JdbcClient` |
| `data-r2dbc` | `spring-boot-starter-data-r2dbc` | Reactive relational access |
| `postgresql` | `org.postgresql:postgresql` (runtime) | PostgreSQL driver **(SC)** |
| `mysql` | `com.mysql:mysql-connector-j` (runtime) | MySQL driver **(SC)** |
| `mariadb` | `org.mariadb.jdbc:mariadb-java-client` (runtime) | MariaDB driver **(SC)** |
| `h2` | `com.h2database:h2` (runtime) | H2 in-memory DB (test/dev only) |
| `data-mongodb` | `spring-boot-starter-data-mongodb` | Spring Data MongoDB **(SC)** |
| `data-redis` | `spring-boot-starter-data-redis` | Redis client (Lettuce) + Spring Data Redis **(SC)** |
| `data-elasticsearch` | `spring-boot-starter-data-elasticsearch` | Spring Data Elasticsearch **(SC)** |
| `flyway` | `org.flywaydb:flyway-core` | Flyway database migrations |
| `liquibase` | `org.liquibase:liquibase-core` | Liquibase database migrations |

---

## Messaging / Events

| Initializr ID | Artifact | Description |
|---|---|---|
| `kafka` | `org.springframework.kafka:spring-kafka` | Spring for Apache Kafka (`KafkaTemplate`, `@KafkaListener`) **(SC)** |
| `kafka-streams` | `org.apache.kafka:kafka-streams` | Kafka Streams topology support |
| `rabbitmq` | `spring-boot-starter-amqp` | Spring AMQP / RabbitMQ **(SC)** |
| `artemis` | `spring-boot-starter-artemis` | JMS via ActiveMQ Artemis **(SC)** |
| `pulsar` | `spring-boot-starter-pulsar` | Spring for Apache Pulsar **(SC)** |
| `cloud-stream` | `org.springframework.cloud:spring-cloud-stream` | Binder-based messaging abstraction over Kafka/RabbitMQ (needs Spring Cloud BOM) |
| `integration` | `spring-boot-starter-integration` | Spring Integration (EIP-style flows) |

---

## Security

| Initializr ID | Artifact | Description |
|---|---|---|
| `security` | `spring-boot-starter-security` | Spring Security core (form login, basic auth, method security) |
| `oauth2-resource-server` | `spring-boot-starter-oauth2-resource-server` | JWT/opaque-token verification — Keycloak, Entra ID, etc. |
| `oauth2-client` | `spring-boot-starter-oauth2-client` | OIDC/OAuth2 login + client credentials for calling secured services |
| `oauth2-authorization-server` | `spring-boot-starter-oauth2-authorization-server` | Issue/sign tokens (Spring Authorization Server) |

---

## Observability

| Initializr ID | Artifact | Description |
|---|---|---|
| `actuator` | `spring-boot-starter-actuator` | Health, metrics, info endpoints (`/actuator/*`) — include in every service |
| `prometheus` | `io.micrometer:micrometer-registry-prometheus` | Prometheus metrics (`/actuator/prometheus`) |
| `distributed-tracing` | `io.micrometer:micrometer-tracing-bridge-otel` | Micrometer Tracing with OpenTelemetry bridge |
| `otlp-metrics` | `io.micrometer:micrometer-registry-otlp` | Ship metrics via OTLP |
| `zipkin` | `io.opentelemetry:opentelemetry-exporter-zipkin` | Export traces to Zipkin **(SC)** |

Health endpoints: `/actuator/health`, `/actuator/health/liveness`, `/actuator/health/readiness`
(probes are enabled automatically on Kubernetes).

---

## Cloud / Deployment

| Initializr ID | Artifact | Description |
|---|---|---|
| `cloud-starter` | `org.springframework.cloud:spring-cloud-starter` | Spring Cloud foundations (needs Spring Cloud BOM) |
| `cloud-config-client` | `org.springframework.cloud:spring-cloud-starter-config` | Externalized config from Config Server |
| `cloud-gateway` | `org.springframework.cloud:spring-cloud-starter-gateway` | API gateway |
| `cloud-resilience4j` | `org.springframework.cloud:spring-cloud-starter-circuitbreaker-resilience4j` | Circuit breakers, retries |
| `native` | *(build plugin config)* | GraalVM native image support (`./mvnw -Pnative native:compile`) |

Container images need no extra dependency — Buildpacks support is built into the build plugins:
`./mvnw spring-boot:build-image` or `./gradlew bootBuildImage`. Kubernetes manifests are not generated by
Spring Boot; probes and graceful shutdown work out of the box with `actuator`.

---

## AI (Spring AI — Initializr adds the `spring-ai-bom`)

| Initializr ID | Description |
|---|---|
| `spring-ai-openai` | OpenAI chat/embeddings (`spring-ai-starter-model-openai`) |
| `spring-ai-anthropic` | Anthropic Claude |
| `spring-ai-ollama` | Ollama (local models) **(SC)** |
| `spring-ai-vectordb-pgvector` | PGvector vector store **(SC)** |
| `spring-ai-mcp-server` | Model Context Protocol server |
| `spring-ai-mcp-client` | Model Context Protocol client |

---

## Developer Tools & Testing

| Initializr ID | Artifact | Description |
|---|---|---|
| `devtools` | `spring-boot-devtools` (runtime) | Auto-restart, live reload — include in every project |
| `docker-compose` | `spring-boot-docker-compose` (runtime) | Starts `compose.yaml` services on run and wires connections |
| `testcontainers` | `spring-boot-testcontainers` (test) | Container-backed tests with `@ServiceConnection` |
| `configuration-processor` | `spring-boot-configuration-processor` | IDE metadata for `@ConfigurationProperties` |
| `lombok` | `org.projectlombok:lombok` | Boilerplate reduction (ask — teams are split; prefer records where possible) |

The test starter (JUnit 5, Mockito, AssertJ) is included in every generated project automatically — never
select or add it manually. Boot 4 also ships per-technology test starters (e.g.
`spring-boot-starter-webmvc-test` for MockMvc) that Initializr adds alongside the matching starter.

---

## Other Useful Dependencies

| Initializr ID | Artifact | Description |
|---|---|---|
| `quartz` | `spring-boot-starter-quartz` | Quartz scheduler (persistent/clustered jobs) — plain `@Scheduled` + `@EnableScheduling` needs no dependency |
| `cache` | `spring-boot-starter-cache` | Caching abstraction (`@Cacheable`) |
| `mail` | `spring-boot-starter-mail` | Email sending |
| `thymeleaf` | `spring-boot-starter-thymeleaf` | Server-side HTML templating |
| `batch` | `spring-boot-starter-batch` | Spring Batch jobs |
| `modulith` | `org.springframework.modulith:spring-modulith-starter-core` | Enforced module boundaries + event publication log |

---

## Finding More Dependencies

```bash
# Full catalog with IDs, grouped by category
curl -s https://start.spring.io/metadata/client -H 'Accept: application/json'

# Generate a reference pom.xml for any dependency combination
curl -s https://start.spring.io/pom.xml -d dependencies=web,data-jpa,postgresql

# Browse interactively
open https://start.spring.io
```
