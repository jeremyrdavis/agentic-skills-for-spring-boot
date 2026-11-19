---
name: spring-boot-app
description: >
  Use this skill whenever a user wants to create, scaffold, bootstrap, initialize, generate, or set up a new
  Spring Boot application or microservice. Also trigger when a user asks how to add starters or dependencies to
  an existing Spring Boot project, upgrade a Spring Boot version, or maintain a Spring Boot app's dependencies.
  This skill ensures all projects are generated with Spring Initializr (start.spring.io) on the latest stable
  Spring Boot release, use official Spring Boot starters with BOM-managed versions, and follow standard Spring
  Boot conventions. Trigger even for casual phrasing like "spin up a Spring Boot app", "start a new Spring Boot
  service", "create a Spring Boot REST API", or "add a starter to my Spring Boot project".
  Does not cover DDD modeling (bounded contexts, aggregates, value objects, repositories, application services);
  use the ddd-* skills together with spring-boot-ddd for that. This skill only supplies the project, the Spring
  Boot version, and the dependencies.
---

# Spring Boot App Skill

This skill governs how to create and maintain Spring Boot applications. Always follow the decision tree and
conventions below.

---

## Step 0 — Check the Latest Spring Boot Version

Before generating any project, always verify the current latest stable Spring Boot release. The version baked
into this skill may be stale. The authoritative source is Spring Initializr's metadata:

```bash
curl -s https://start.spring.io/metadata/client -H 'Accept: application/json'
```

The `bootVersion.default` field is the current recommended release. Also useful:

- Releases: https://github.com/spring-projects/spring-boot/releases (e.g. currently `4.1.x`)
- Support timelines (OSS vs. enterprise): https://spring.io/projects/spring-boot#support
- Spring Boot ships two minor releases per year; each minor gets ~12 months of OSS support. There is no
  separate community LTS stream — longer lifecycles come from Tanzu Spring enterprise support
  (see `references/enterprise-support.md`)

Use the latest stable release (never a milestone, RC, or SNAPSHOT) unless told otherwise. The previous minor
(e.g. `4.0.x`) is acceptable only if the user needs a dependency that hasn't caught up yet.

---

## Step 1 — Gather Project Parameters

Before generating, collect:

| Parameter | Default | Notes |
|---|---|---|
| `groupId` | `io.arrogantprogrammer` | Always ask if not provided |
| `artifactId` | *(required)* | The app name / artifact ID |
| `packageName` | `{groupId}.{artifactId}` (sanitized) | Initializr derives it; override if the user has a convention |
| `version` | `0.0.1-SNAPSHOT` | Initializr convention for new apps |
| `dependencies` | *(required)* | See dependency guidance below |
| `build tool` | Maven | Always ask — Gradle is equally first-class in the Spring ecosystem (and is Initializr's web default) |
| `Java version` | 25 | Always ask — suggest 25 (current LTS; Spring Boot 4 baseline is 17) |
| `packaging` | `jar` | Use `war` only for legacy servlet-container deployment |

If the user hasn't specified a Java version, ask which version they'd like to use and suggest 25 as the default.

If the user hasn't specified dependencies, ask or propose sensible defaults based on their described use case.

---

## Step 2 — Choose the Tooling

**Always generate through Spring Initializr (start.spring.io)** — never hand-write a `pom.xml` or
`build.gradle` for a new project. Initializr produces the correct parent/BOM setup, wrapper scripts, and any
extra BOMs (Spring Cloud, Spring AI) with compatible versions.

Note: Initializr always generates the `@SpringBootApplication` main class and one context-loads test — that
*is* the Spring Boot convention. There is no demo/example code to strip out.

### Option A: Initializr REST API via curl (preferred)

Works everywhere — no local tooling required. Omit `bootVersion` to get Initializr's current default release.

```bash
curl -s https://start.spring.io/starter.zip \
  -d type=maven-project \
  -d groupId={groupId} \
  -d artifactId={artifactId} \
  -d name={artifactId} \
  -d javaVersion=25 \
  -d dependencies={dependency1},{dependency2},{dependency3} \
  -d baseDir={artifactId} \
  -o {artifactId}.zip \
&& unzip -q {artifactId}.zip && rm {artifactId}.zip
```

**Key parameters:**
- `dependencies` — comma-separated Initializr dependency IDs (no spaces after commas)
- `type` — `maven-project` (default here), `gradle-project` (Groovy DSL), or `gradle-project-kotlin`
- `javaVersion` — `17`, `21`, `25`
- `baseDir` — puts the project in a subdirectory instead of extracting flat (always set it)
- `bootVersion` — only to pin a specific version, e.g. `-d bootVersion=4.1.2`; normally omit

### Option B: Spring Boot CLI (if installed)

```bash
spring init \
  --build=maven \
  --java-version=25 \
  --group-id={groupId} \
  --artifact-id={artifactId} \
  --dependencies={dependency1},{dependency2},{dependency3} \
  {artifactId}
```

**Installation note** (include if user wants the CLI):
```bash
# Via SDKMAN (recommended)
sdk install springboot

# Via Homebrew (macOS/Linux)
brew tap spring-io/tap && brew install spring-boot
```

### Option C: Web UI / IDE (for humans)

Point the user at https://start.spring.io, or the Spring Initializr integration built into IntelliJ IDEA,
VS Code (Spring Boot Extension Pack), and Spring Tool Suite. Same generator, same output.

---

## Step 3 — Dependency Selection Rules

### Always use official starters — never assemble Spring functionality from individual jars

| Source | When to use | How to reference |
|---|---|---|
| **Spring Boot starters** | Core features (web, data, security…) | Initializr ID (e.g. `web`) → `spring-boot-starter-web` |
| **Spring portfolio BOMs** | Spring Cloud, Spring AI, Spring Modulith | Select at generation time so Initializr adds the BOM |
| **Third-party starters** | e.g. springdoc, Testcontainers | Prefer ones listed on Initializr; otherwise their documented starter artifact |
| **Never raw** | Individual `spring-web`, `spring-orm`, etc. jars | The starter brings them in, version-aligned |

### Version management — the cardinal rule

Spring Boot manages dependency versions through `spring-boot-starter-parent` (Maven) or the Spring Boot
Gradle plugin. **Never add a `<version>` to any BOM-managed dependency.** If a managed version must change,
override the version property (e.g. `<jackson-bom.version>`) rather than pinning the artifact.

Spring Cloud and Spring AI have their own BOMs (`spring-cloud-dependencies`, `spring-ai-bom`) whose release
trains must match the Spring Boot minor. Initializr wires this correctly — one more reason to select these
dependencies at generation time rather than retrofitting them.

### Adding dependencies to an existing project

The Spring Boot idiom is to edit the build file directly — starters are plain dependencies with no version
tag, so there is nothing a CLI needs to resolve:

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

When unsure of coordinates or required BOMs, generate a reference build file with the target dependencies and
diff it against the project:

```bash
curl -s https://start.spring.io/pom.xml -d dependencies=data-jpa,postgresql
```

### Common dependency reference table

See `references/starters.md` for a curated list of commonly used dependencies by category, with Initializr IDs
and Maven coordinates.

---

## Step 4 — Post-Generation Guidance

After generating, always remind the user:

1. **Run the app**: `./mvnw spring-boot:run` (Maven) or `./gradlew bootRun` (Gradle)
2. **Live reload**: add the `devtools` dependency for automatic restart on code change
3. **Actuator**: if `actuator` was selected, health is at `http://localhost:8080/actuator/health`
4. **Local services**: the Spring Boot equivalent of auto-provisioned dev infrastructure is
   **Docker Compose support** (`docker-compose` dependency) — Spring Boot starts the services in
   `compose.yaml` and auto-configures connections on `bootRun`/`spring-boot:run` — and
   **Testcontainers** (`testcontainers` dependency) with `@ServiceConnection` for tests and
   `TestMyApplication` for container-backed local dev
5. **Configuration** location: `src/main/resources/application.properties` (or `application.yaml` —
   both work out of the box; pick one and stay consistent)

---

## Step 5 — Maintaining Existing Apps

### Upgrade Spring Boot version

Within the same minor (patch bump), update one line — the parent version in `pom.xml` or the plugin version
in `build.gradle` — and rebuild. All managed dependency versions move with it.

Across minors or majors, use OpenRewrite's maintained migration recipes:

```bash
# Maven — no build file changes needed; check the recipe catalog for the target version
./mvnw org.openrewrite.maven:rewrite-maven-plugin:run \
  -Drewrite.recipeArtifactCoordinates=org.openrewrite.recipe:rewrite-spring:RELEASE \
  -Drewrite.activeRecipes=org.openrewrite.java.spring.boot4.UpgradeSpringBoot_4_0
```

Recipe catalog: https://docs.openrewrite.org/recipes/java/spring — pick the recipe matching the target
version (recipes chain, so one recipe handles all intermediate hops).

Always read the release notes / migration guide first:
`https://github.com/spring-projects/spring-boot/wiki` (one page per release).

### Check for available updates

```bash
./mvnw versions:display-parent-updates      # newer Spring Boot parent
./mvnw versions:display-dependency-updates  # non-managed dependencies
```

### Inspect the dependency tree

```bash
./mvnw dependency:tree          # Maven
./gradlew dependencies          # Gradle
```

---

## Canonical Examples

### Minimal REST + PostgreSQL service (curl)

```bash
curl -s https://start.spring.io/starter.zip \
  -d type=maven-project \
  -d groupId=io.arrogantprogrammer \
  -d artifactId=my-service \
  -d name=my-service \
  -d javaVersion=25 \
  -d dependencies=web,data-jpa,postgresql,validation \
  -d baseDir=my-service \
  -o my-service.zip \
&& unzip -q my-service.zip && rm my-service.zip
```

### Same, with the Spring Boot CLI

```bash
spring init --build=maven --java-version=25 \
  --group-id=io.arrogantprogrammer --artifact-id=my-service \
  --dependencies=web,data-jpa,postgresql,validation \
  my-service
```

### AI service (Spring AI + OpenAI)

```bash
curl -s https://start.spring.io/starter.zip \
  -d type=maven-project \
  -d groupId=io.arrogantprogrammer \
  -d artifactId=ai-service \
  -d name=ai-service \
  -d javaVersion=25 \
  -d dependencies=web,spring-ai-openai \
  -d baseDir=ai-service \
  -o ai-service.zip \
&& unzip -q ai-service.zip && rm ai-service.zip
```

Initializr adds the `spring-ai-bom` automatically.

### Kafka event-driven service

```bash
curl -s https://start.spring.io/starter.zip \
  -d type=maven-project \
  -d groupId=io.arrogantprogrammer \
  -d artifactId=event-service \
  -d name=event-service \
  -d javaVersion=25 \
  -d dependencies=kafka,web \
  -d baseDir=event-service \
  -o event-service.zip \
&& unzip -q event-service.zip && rm event-service.zip
```

### DDD-shaped service (pairs with the `ddd-*` skills and `spring-boot-ddd`)

```bash
curl -s https://start.spring.io/starter.zip \
  -d type=maven-project \
  -d groupId=io.arrogantprogrammer \
  -d artifactId=orders-service \
  -d name=orders-service \
  -d javaVersion=25 \
  -d dependencies=web,data-jpa,postgresql,validation \
  -d baseDir=orders-service \
  -o orders-service.zip \
&& unzip -q orders-service.zip && rm orders-service.zip
```

Add `kafka` when domain events leave the service (outbox relay); consider `modulith` (Spring Modulith) for
enforced module boundaries and its built-in event publication log. After creation, hand over to
`ddd-foundations` for the package layout and to `spring-boot-ddd` for the Spring mechanics; this skill does
not generate domain code.

---

## Important Rules

- **NEVER** add `<version>` tags to dependencies managed by the Spring Boot BOM — version management comes
  from `spring-boot-starter-parent` (or the `spring-boot-dependencies` import) and the Gradle plugin
- **NEVER** hand-write the build file for a new project — always generate through Spring Initializr so the
  parent, wrapper, and any portfolio BOMs (Spring Cloud, Spring AI) are set up with compatible versions
- **NEVER** mix Spring Cloud or Spring AI versions freely with a Spring Boot version — their BOM release
  trains are paired with specific Spring Boot minors
- Always verify the Spring Boot version at Step 0 before filling in commands
- If the user needs a support lifecycle beyond OSS (or mentions Tanzu, Broadcom, or commercial support), see
  `references/enterprise-support.md`
