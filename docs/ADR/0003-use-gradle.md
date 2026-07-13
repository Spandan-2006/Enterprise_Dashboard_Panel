# ADR-0003: Use Gradle with Kotlin DSL as the Build Tool

## Status
Accepted

## Context
The project needs a build tool to handle:
- Dependency management (Spring Boot BOM, JJWT, test frameworks)
- Compilation (Java 21 with preview features potentially needed)
- Test execution with JaCoCo coverage enforcement
- Docker image building (bootJar)
- CI integration (GitHub Actions)
- Future multi-module support if the project grows

## Decision
Gradle 8.x with Kotlin DSL (`build.gradle.kts`) and the Gradle Wrapper (`./gradlew`).

## Rationale
- **Build cache**: Gradle's incremental build cache skips unchanged tasks — `./gradlew build` after a minor change runs in seconds vs. Maven's full rebuild
- **Kotlin DSL**: type-safe, IDE-completable build scripts (vs. Groovy DSL's dynamic nature); better IntelliJ IDEA support
- **Spring Boot Gradle plugin**: `bootJar`, `bootRun`, `bootBuildImage` — equivalent to Maven Spring Boot plugin with better multi-stage Docker support
- **JaCoCo integration**: `jacocoTestReport` and `jacocoTestCoverageVerification` tasks integrate cleanly
- **Spotless plugin**: code formatting enforcement in Gradle is simpler than Maven's Checkstyle XML configuration

## Consequences

### Positive
- Faster incremental builds in CI due to Gradle's task output caching
- Type-safe build scripts reduce typo-induced build configuration bugs
- Native Wrapper means no Gradle installation needed on any machine — `./gradlew` downloads the correct version

### Negative / Trade-offs
- Team members unfamiliar with Gradle have a learning curve (especially Kotlin DSL vs. Groovy DSL)
- Gradle's Kotlin DSL syntax can be verbose for simple plugin configurations vs. Maven POM
- Gradle's daemon (background process) occasionally causes memory issues on resource-constrained CI runners — mitigated by `--no-daemon` flag in CI

## Alternatives Considered

| Alternative | Pros | Cons | Reason Rejected |
|---|---|---|---|
| Maven (POM.xml) | Familiar to most Java developers, extensive plugin ecosystem, predictable lifecycle | Verbose XML, no incremental build caching, slower full builds | Gradle's build cache and type-safe DSL better suit this project's CI performance requirements |
| Gradle with Groovy DSL | Same build engine, more familiar syntax for many developers | Dynamic typing leads to harder-to-debug build scripts; poorer IDE completion | Kotlin DSL's type safety was judged to outweigh the familiarity advantage |
| Bazel | Hermetic builds, excellent at monorepo scale | Extremely steep learning curve, overkill for a single-module project | Inappropriate for the project's current scope and team size |
