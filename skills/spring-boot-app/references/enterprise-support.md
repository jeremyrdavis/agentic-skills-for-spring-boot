# Enterprise Support Reference (Tanzu Spring)

Use this guidance when:
- User mentions commercial support, Broadcom, VMware, or Tanzu Spring
- User needs security fixes for a Spring Boot minor past its OSS end-of-support date
- User needs a multi-year support lifecycle instead of the ~12-month OSS window per minor

---

## OSS Spring Boot vs. Tanzu Spring

| Aspect | OSS Spring Boot | Tanzu Spring (Broadcom) |
|---|---|---|
| Release cadence | Two minors per year (May/June and November) | Same releases |
| Support lifecycle per minor | ~12 months of OSS patches | Extended support (years beyond OSS EOL) |
| Requires subscription | No | Yes (for production use of enterprise artifacts) |
| Artifact source | Maven Central | Broadcom enterprise Maven repository |
| Extras | — | Enterprise starters/extensions (marked `[Enterprise]` on start.spring.io), Spring Cloud Gateway commercial modules, Tanzu Spring SDK |

Exact dates per minor are published at https://spring.io/projects/spring-boot#support — always check there
rather than relying on this file.

---

## How enterprise artifacts are consumed

Commercial patch releases (e.g. a `3.5.x` patch issued after OSS EOL) are published to the Broadcom
enterprise repository, not Maven Central. With a Tanzu Spring subscription, add the repository with
credentials (typically in `settings.xml`, not committed to the project):

```xml
<repositories>
  <repository>
    <id>spring-enterprise</id>
    <url>https://packages.broadcom.com/artifactory/spring-enterprise</url>
    <releases><enabled>true</enabled></releases>
    <snapshots><enabled>false</enabled></snapshots>
  </repository>
</repositories>
```

Check the current repository URL and authentication setup in the Tanzu Spring documentation — Broadcom has
changed hosting details before.

---

## Enterprise extensions on Spring Initializr

start.spring.io lists Tanzu Spring enterprise dependencies (marked `[Enterprise]`), e.g. the Governance
Starter and the commercial Spring Cloud Gateway modules (SSO, access control, traffic control,
transformation). Selecting them generates a project that requires the enterprise repository to build — warn
the user of this before including any.

---

## Alternatives worth mentioning

- **Stay current on OSS**: upgrading twice a year within the OSS window is the mainstream path; OpenRewrite
  recipes (see SKILL.md Step 5) keep the cost low
- **Third-party extended support** (e.g. HeroDevs NES for Spring) exists for EOL versions without a Broadcom
  subscription — mention only if the user is stuck on an EOL minor and cannot upgrade

---

## Resources

- Spring Boot support timelines: https://spring.io/projects/spring-boot#support
- Tanzu Spring product page: https://www.vmware.com/products/app-platform/tanzu-spring
- Spring enterprise support policy: https://spring.io/support-policy
