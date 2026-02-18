# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is the **Spinnaker monorepo** — a Gradle composite build containing all Spinnaker microservices in a single repository. Each service is an `includedBuild` in Gradle's composite build system, meaning they can reference each other directly without publishing JARs first.

## Build Commands

All commands run from the repo root using the Gradle wrapper (`./gradlew`).

```bash
# Build all JVM services (excludes deck, spin)
./gradlew build

# Build everything including UI and CLI
./gradlew buildAll

# Build a single service
./gradlew :echo:build
./gradlew :clouddriver:build

# Run a specific service locally
./gradlew echo          # runs echo-web
./gradlew clouddriver   # runs clouddriver-web
./gradlew deck          # runs deck dev server

# Clean
./gradlew clean
```

## Testing

```bash
# Run all JVM tests
./gradlew test

# Run tests for a specific service
./gradlew :echo:test
./gradlew :clouddriver:test

# Run a specific test class
./gradlew :echo:echo-scheduler:test --tests MissedPipelineTriggerCompensationJobSpec
./gradlew :clouddriver:clouddriver-web:test --tests "com.netflix.spinnaker.clouddriver.web.SomeTest"

# Run a specific test method
./gradlew :orca:orca-core:test --tests "com.netflix.spinnaker.orca.SomeSpec.method name"

# Run Deck (UI) tests
cd deck && yarn test
```

## Code Style

```bash
# Check formatting (Java/Groovy/Kotlin)
./gradlew spotlessCheck

# Auto-fix formatting
./gradlew spotlessApply

# Deck linting
cd deck && yarn lint
cd deck && yarn prettier:check
cd deck && yarn prettier       # auto-fix
```

## Architecture

### Services

| Service | Role |
|---------|------|
| **gate** | API gateway — all client traffic enters here |
| **orca** | Pipeline/workflow orchestration engine |
| **clouddriver** | Cloud infrastructure abstraction (AWS, GCP, Azure, Kubernetes, etc.) |
| **front50** | Persistent storage for pipelines and applications |
| **echo** | Event system, notifications, pipeline triggers, webhooks |
| **fiat** | Role-based authorization |
| **igor** | CI/CD integration (Jenkins, Travis, GitLab CI, etc.) |
| **kayenta** | Automated canary analysis |
| **keel** | Managed delivery / declarative infrastructure |
| **rosco** | VM image baking |
| **halyard** | Configuration and lifecycle management tool |
| **kork** | Shared library used by all JVM services (~48 submodules) |
| **deck** | TypeScript/React web UI |
| **deck-kayenta** | Kayenta-specific UI components |
| **spin** | Go-based CLI |

### Request Flow

```
Deck (UI) → Gate (API gateway) → Orca / Clouddriver / Front50 / etc.
                                       ↓
                              Echo / Igor / Fiat / Kayenta / Keel / Rosco
```

### Composite Build Structure

- Root `settings.gradle` uses `includeBuild` for each service
- Root `build.gradle` defines aggregating meta-tasks (`build`, `test`, `check`, etc.) that fan out to all included builds
- Services that depend on `kork` (all of them) get it via composite build substitution — no local publishing needed
- `spinnaker-gradle-project` (also an included build) provides the shared Gradle plugin used by all services
- `versions.gradle` (root) centralizes version constraints

### Service Internal Structure

Each JVM service follows the same pattern:
- `{service}-web/` — Spring Boot application entry point, REST controllers
- `{service}-core/` — Core business logic
- `{service}-{provider}/` — Provider-specific implementations (e.g., `clouddriver-aws`, `clouddriver-kubernetes`)
- Tests use **Spock** (Groovy) for most legacy tests and **JUnit 5** for newer Java tests

### Technology Stack

- **JVM**: Java 11/17, Groovy, Kotlin; Spring Boot + Spring Cloud
- **Testing**: Spock (Groovy) for existing tests, JUnit 5 for new Java tests
- **HTTP clients**: Retrofit 1.x (`kork-retrofit`) and Retrofit 2.x (`kork-retrofit2`)
- **UI**: TypeScript, React, Yarn, Lerna (monorepo tooling within `deck/`)
- **CLI**: Go 1.23+ with Cobra (`spin/`)
- **Databases**: MySQL / PostgreSQL with Liquibase migrations; Redis for caching
- **CI**: GitHub Actions (`.github/workflows/`)

### Key Kork Modules

`kork` is a dependency of every other service. Key modules:
- `kork-core` — Core utilities and annotations
- `kork-retrofit`, `kork-retrofit2` — HTTP client factories
- `kork-web` — OkHttp and web infrastructure shared config
- `kork-sql` — SQL/database abstractions
- `kork-security` — Security configuration

### Gradle Notes

- Gradle 7.6.1 (Gradle 8 is not supported — Kotlin plugins incompatible)
- `org.gradle.parallel=true` and `-Xmx6g` are set in `gradle.properties`
- To target a subproject of an included build: `:serviceName:submodule:task`
  - e.g., `./gradlew :echo:echo-web:run`
