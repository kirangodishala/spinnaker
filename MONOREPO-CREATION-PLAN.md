# Plan: Recreating the Spinnaker Monorepo from Individual Services

Based on `init.sh`, `pull.sh`, `MONOREPO.md`, `ADOPTION.md`, and the GHA workflows, here are all the pieces needed.

---

## Phase 1 — Repository Initialization

**1.1 Create a new empty GitHub repo** (e.g., `your-org/spinnaker`)
- Clean slate: no branches, no tags, no releases
- Set up branch protection rules on `main`

**1.2 Create the initial empty commit**
```bash
git init
git commit --allow-empty -m "Initial empty commit"
```

**1.3 Create root scaffold files manually** — these do NOT come from any individual service repo; they are monorepo-specific:

| File | Purpose |
|------|---------|
| `settings.gradle` | `includeBuild` declarations for all services |
| `build.gradle` | Aggregating meta-tasks (`build`, `test`, `check`, `spotlessCheck`, service run aliases) |
| `versions.gradle` | Centralized Kotlin version catalog used across all services |
| `gradle.properties` | `org.gradle.parallel=true`, `-Xmx6g`, etc. |
| `gradlew` / `gradlew.bat` | Single root Gradle wrapper (Gradle 7.6.1) |
| `gradle/wrapper/gradle-wrapper.*` | Wrapper jar and properties |
| `init.sh` | Reference script documenting the subtree add commands |
| `pull.sh` + `subtree_pull_editor.sh` | For future ongoing sync from individual repos |

---

## Phase 2 — Import Individual Services via `git subtree add`

Run for each service **in this order** (shared library first):

```bash
# 1. Shared Gradle plugin — MUST be first (buildscript dependency)
git subtree add -P spinnaker-gradle-project \
  git@github.com:spinnaker/spinnaker-gradle-project.git master

# 2. Core shared library — everything depends on this
git subtree add -P kork git@github.com:spinnaker/kork.git master

# 3. Services
git subtree add -P clouddriver git@github.com:spinnaker/clouddriver.git master
git subtree add -P deck        git@github.com:spinnaker/deck.git master
git subtree add -P deck-kayenta git@github.com:spinnaker/deck-kayenta.git master
git subtree add -P echo        git@github.com:spinnaker/echo.git master
git subtree add -P gate        git@github.com:spinnaker/gate.git master
git subtree add -P orca        git@github.com:spinnaker/orca.git master
git subtree add -P rosco       git@github.com:spinnaker/rosco.git master
```

> Even if you only need clouddriver/deck/gate/orca/echo/rosco, you **must** include `kork` and
> `spinnaker-gradle-project` — all services depend on them. Add `fiat`, `front50`, `igor`, etc. as needed.

---

## Phase 3 — Per-Service Gradle Modifications

After importing, each service's build files need changes. For **each service**:

### Remove

| What | Why |
|------|-----|
| Nested `.github/` folder | Replaced by root-level GHA |
| Nested `.idea/` folder | IDE config not needed |
| Gradle wrapper files (`gradlew`, `gradle/wrapper/`) inside each service | Root wrapper takes over |
| All `mavenLocal()` repository declarations | Composite build substitution replaces local publishing |
| All `defaultTasks` declarations in service `build.gradle` | Root `build.gradle` defines entry points |
| Project version pin properties | Root `versions.gradle` centralizes these |
| Old partial-composite build code | Clean slate via full composite |
| Old `enableFeaturePreview` declarations | Out of preview in Gradle 7 |

### Add / Fix

| What | Why |
|------|-----|
| `duplicatesStrategy = DuplicatesStrategy.EXCLUDE` on `Copy` tasks | Gradle 7 now validates for classpath duplicates |
| Point each service `settings.gradle` to root `versions.gradle` | `apply from: '../versions.gradle'` |
| Upgrade Liquibase to 4.3.5 | Fixes classpath duplicate issue + MySQL test failures |
| Consolidate `kotlin.gradle` / `kotlin-test.gradle` files | Move to root, split off detekt config |
| Upgrade `spinnaker-gradle-project` plugins for Gradle 7 | Publishing plugins incompatible with older versions |

---

## Phase 4 — Composite Build Wiring

**Root `settings.gradle`:**
```groovy
apply from: './versions.gradle'

includeBuild 'spinnaker-gradle-project'  // FIRST — it's a buildscript dep
includeBuild 'kork'
includeBuild 'clouddriver'
includeBuild 'deck'
includeBuild 'deck-kayenta'
includeBuild 'echo'
includeBuild 'gate'
includeBuild 'orca'
includeBuild 'rosco'
```

**Root `build.gradle`** must wire up:
- `build`, `test`, `check`, `clean`, `assemble`, `publish`, `publishToMavenLocal`, `spotlessCheck`, `spotlessApply` — fan-out to all JVM services
- `buildAll`, `testAll`, `checkAll` — variants that include deck/spin
- Per-service run aliases (`./gradlew clouddriver`, `./gradlew echo`, etc. → `{service}-web:run`)

---

## Phase 5 — GitHub Actions Setup

### 5.1 Secrets to configure in the repo

| Secret | Used For |
|--------|---------|
| `GAR_JSON_KEY` | Google Artifact Registry access (Docker, Debian, Maven) |
| `GRADLE_PUBLISH_KEY` | Publishing `spinnaker-gradle-project` to Gradle Plugin Portal |
| `GRADLE_PUBLISH_SECRET` | Same |
| `NEXUS_USERNAME` | Java library publishing to Nexus/Maven Central |
| `NEXUS_PASSWORD` | Same |
| `NEXUS_PGP_SIGNING_KEY` | JAR signing |
| `NEXUS_PGP_SIGNING_PASSWORD` | Same |
| `NPM_AUTH_TOKEN` | Deck packages → npmjs.org |
| `GAR_NPM_PASSWORD` | Deck packages → GAR |
| GitHub PAT (for `update-monorepo` action) | PRs created by bot must use PAT, not `GITHUB_TOKEN` |

### 5.2 Workflows under `.github/workflows/`

| Workflow | Triggered By |
|----------|-------------|
| Per-service workflows (`clouddriver.yml`, `echo.yml`, etc.) | `paths:` filter on each service directory |
| `spinnaker-libraries.yml` | Reusable — publishes all Java libs together with one coherent version |
| `version.yml` | Reusable — centralizes versioning, called once per workflow to avoid double build-number bumps |
| `rebuild-all.yml` | Manual `workflow_dispatch` only |
| `kork.yml` | kork changes → triggers all service rebuilds |

### 5.3 Custom composite actions under `.github/actions/`

| Action | Purpose |
|--------|---------|
| `version/` | Generates version info (build number, release train, semver, tags) |
| `build-tag-number/` | Stores incrementing build counters as git tags (e.g., `bn-clouddriver-main-42`) |
| `generic-build-publish/` | Reusable build+test+Docker+deb+halconfig+NPM publish pipeline |
| `publish-docker/` | Multi-platform Docker image publishing to GAR |
| `publish-deb/` | Debian package publishing to GAR apt repo |
| `publish-npm/` | NPM package publishing (Deck) |
| `publish-halconfig/` | Publishes Halyard service configs |
| `setup-node/` | Sets up Node.js only if `package.json` is present |
| `update-monorepo/` | Pulls changes from individual repos into subtrees, creates PRs |
| `spinnaker-release/` | Fully automated BOM/release generation |

---

## Phase 6 — Versioning System

Build numbers are stored as git tags: `bn-<namespace>-<ref>-<N>`

| Branch | Version Format | Example |
|--------|---------------|---------|
| `main` | `main-<N>` | `main-42` |
| `release-2023.1.x` | `2023.1.<N>` | `2023.1.5` |
| Pull Request | `pr<number>-0` | `pr123-0` |

- The `spinnaker` namespace coordinates all Java library versions so `-bom` packages are internally coherent.
- The `deck` namespace aligns all Deck package versions — all packages ship under the same version.
- **Java libraries must publish on every push** — `-bom` packages reference the global version internally, so a partial publish breaks dependency resolution.
- Containers and debs only publish when their service or a direct dependency (e.g., `kork`) changed.

---

## Phase 7 — Artifact Publishing Targets

Replace test/staging references with production targets before going live:

| Artifact Type | Staging (test) | Production |
|--------------|---------------|------------|
| Maven JARs | `spinnaker-monorepo-test` GAR | Nexus / Maven Central |
| Docker images | `spinnaker-monorepo-test` GAR | `spinnaker-community` GAR |
| Debian packages | `spinnaker-monorepo-test` GAR apt | `spinnaker-community` GAR apt |
| NPM packages | `spinnaker-monorepo-test` GAR npm | `npmjs.org` |

---

## Phase 8 — Ongoing Sync (Post-Creation)

To pull upstream changes from individual repos into the monorepo:

```bash
# Pull one service from master
./pull.sh clouddriver

# Pull from a specific release branch
./pull.sh clouddriver orca -r release-1.27.x

# Pull all services
./pull.sh
```

Or use the `update-monorepo` GHA action, which creates a PR with mergeable subtrees auto-applied and conflicts flagged.

**Critical rule**: Always merge with a **regular merge commit** — never squash or rebase.
Squashing destroys the git history that `git subtree` needs to compute diffs for future pulls.

---

## Summary — Order of Operations

```
1.  Empty repo + root scaffold files (settings.gradle, build.gradle, versions.gradle,
    gradle.properties, gradlew wrapper)
2.  git subtree add: spinnaker-gradle-project  (buildscript dep — must be first)
3.  git subtree add: kork                      (runtime dep for all services)
4.  git subtree add: each service              (clouddriver, deck, echo, gate, orca, rosco, ...)
5.  Per-service Gradle cleanup                 (remove mavenLocal, wrappers, .github, .idea, etc.)
6.  Root settings.gradle + build.gradle wiring
7.  GHA workflows + secrets
8.  Versioning system (build-tag-number action)
9.  Validate: ./gradlew build
10. Cut over artifact publishing: test GAR → production GAR / Nexus / npmjs.org
```
