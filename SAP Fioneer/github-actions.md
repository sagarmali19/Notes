# GitHub Actions & CI/CD – Interview Notes (Java Backend)

> JD line: *"Knowledge of GitHub Actions, CI/CD pipelines"*

---

## 1. CI/CD in 30 Seconds

| Term | Meaning |
|---|---|
| **CI (Continuous Integration)** | Developers merge code frequently; every push/PR automatically **builds and tests** the code to catch problems early |
| **Continuous Delivery** | Every successful build is **deployable**; release to production is a **manual approval** |
| **Continuous Deployment** | Every successful build is **automatically deployed** to production, no manual step |

**Say in interview:** *"CI gives fast feedback on every change through automated build, test and static analysis. CD automates packaging and deployment so releases are small, repeatable and low risk."*

**Typical Java pipeline:**
```
Commit/PR → Checkout → Build (Maven/Gradle) → Unit tests → Static analysis (Sonar)
→ Integration tests → Package (JAR/Docker image) → Push to registry
→ Deploy to Dev → Test → Staging → (approval) → Production
```

---

## 2. What is GitHub Actions?

GitHub's built-in automation platform. Workflows are defined as **YAML files** in `.github/workflows/` and triggered by repository **events** (push, pull request, schedule, manual, etc.).

### Core Concepts

| Concept | Description |
|---|---|
| **Workflow** | YAML file defining an automated process |
| **Event (trigger)** | What starts it: `push`, `pull_request`, `schedule`, `workflow_dispatch`, `release`, `workflow_call` |
| **Job** | A set of steps running on the same runner. Jobs run **in parallel** by default; use `needs:` for order |
| **Step** | Single task: a shell command (`run`) or a reusable action (`uses`) |
| **Action** | Reusable unit (e.g., `actions/checkout@v4`) from Marketplace or your own |
| **Runner** | Machine executing the job: **GitHub-hosted** (ubuntu/windows/macos) or **self-hosted** |
| **Secrets / Variables** | Secure and plain config values, at repo, environment or org level |
| **Artifacts** | Files produced by a job (JAR, test reports) shared between jobs/downloaded |
| **Cache** | Saved dependencies (e.g., `~/.m2`) to speed up builds |
| **Environment** | Deployment target (dev/staging/prod) with protection rules, approvals and secrets |

**Memory trick:** *Event → Workflow → Jobs → Steps → Actions, running on Runners.*

---

## 3. Basic CI Workflow for Spring Boot (Maven)

```yaml
name: CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'
          cache: maven            # caches ~/.m2 automatically

      - name: Build and test
        run: mvn -B clean verify

      - name: Upload test report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-reports
          path: target/surefire-reports/

      - name: Upload JAR
        uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: target/*.jar
```

Notes: `-B` = batch mode (clean logs), `if: always()` runs even if tests fail, `cache: maven` is the simplest caching approach.

---

## 4. CI + CD Workflow with Docker and Deployment

```yaml
name: CI-CD

on:
  push:
    branches: [ main ]

env:
  IMAGE_NAME: ghcr.io/${{ github.repository }}

jobs:
  build-test:
    runs-on: ubuntu-latest
    services:
      postgres:                         # real DB for integration tests
        image: postgres:16
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: appdb
        ports: [ "5432:5432" ]
        options: >-
          --health-cmd "pg_isready -U test"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'
          cache: maven
      - run: mvn -B verify
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/appdb
          SPRING_DATASOURCE_USERNAME: test
          SPRING_DATASOURCE_PASSWORD: test

  docker:
    needs: build-test                    # runs only if build-test succeeds
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ${{ env.IMAGE_NAME }}:latest
            ${{ env.IMAGE_NAME }}:${{ github.sha }}

  deploy-staging:
    needs: docker
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - run: echo "Deploy ${{ github.sha }} to staging (kubectl/helm/az/aws cli)"

  deploy-prod:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production              # configure required reviewers = manual approval
    steps:
      - run: echo "Deploy ${{ github.sha }} to production"
```

**Key points to mention**
- `needs:` creates the pipeline order.
- `services:` spins up sidecar containers (Postgres) for integration tests.
- `environment:` + **required reviewers** gives manual approval for production.
- Tag images with the **commit SHA** for traceability and rollbacks, not only `latest`.

---

## 5. Useful Features

### Matrix builds (test multiple versions/OS)
```yaml
strategy:
  fail-fast: false
  matrix:
    java: [ 17, 21 ]
    os: [ ubuntu-latest, windows-latest ]
runs-on: ${{ matrix.os }}
steps:
  - uses: actions/setup-java@v4
    with:
      distribution: temurin
      java-version: ${{ matrix.java }}
```

### Conditions
```yaml
if: github.ref == 'refs/heads/main' && github.event_name == 'push'
```

### Secrets and variables
```yaml
env:
  DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
  APP_ENV: ${{ vars.APP_ENV }}
```
- Secrets are masked in logs; **never echo them**; not passed to workflows from forks by default.

### Reusable workflows
```yaml
# caller
jobs:
  build:
    uses: my-org/shared-workflows/.github/workflows/java-build.yml@v1
    with:
      java-version: '17'
    secrets: inherit
```
The called workflow uses `on: workflow_call`. Good for standardizing pipelines across teams.

### Concurrency (avoid parallel/duplicate deployments)
```yaml
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: true
```

### Scheduled and manual triggers
```yaml
on:
  schedule:
    - cron: '0 2 * * *'     # nightly at 02:00 UTC
  workflow_dispatch:         # manual "Run workflow" button
```

---

## 6. Security Best Practices

1. **Least privilege `GITHUB_TOKEN`**: set `permissions:` explicitly (e.g., `contents: read`).
2. Store credentials in **Secrets / Environments**, never in code.
3. Prefer **OIDC** (OpenID Connect) to authenticate to Azure/AWS instead of long-lived cloud keys.
4. **Pin third-party actions** to a version tag or, best, a **commit SHA**.
5. Protect `main`: required PR reviews, required status checks, no direct pushes.
6. Be careful with `pull_request_target` and untrusted PR code (secret exposure risk).
7. Scan: **Dependabot** (dependencies), **CodeQL** (SAST), **Trivy** (container image), secret scanning.
8. Use **self-hosted runners** only in trusted setups (isolation, ephemeral runners).

---

## 7. Speed and Reliability Tips

- **Cache** dependencies (`cache: maven`/`gradle`).
- Run independent jobs in **parallel**; split slow test suites.
- Use `concurrency` to cancel outdated runs.
- Keep Docker builds fast: multi-stage Dockerfile, layer caching (`cache-from: type=gha`).
- Fail fast: lint/unit tests first, heavier integration tests later.
- Set `timeout-minutes` on jobs to avoid stuck runs.

---

## 8. GitHub Actions vs Jenkins (Common Comparison)

| Aspect | GitHub Actions | Jenkins |
|---|---|---|
| Hosting | SaaS (plus self-hosted runners) | Self-hosted, you maintain it |
| Config | YAML in repo | Jenkinsfile (Groovy) / UI |
| Setup effort | Very low | Higher (server, plugins, upgrades) |
| Ecosystem | Marketplace actions | Huge plugin ecosystem |
| Integration | Native with GitHub (PRs, checks, secrets) | Needs plugins/webhooks |
| Flexibility | Good | Very high, highly customizable |

---

## 9. Deployment Strategies (Often Asked Along With CI/CD)

| Strategy | Idea | Benefit |
|---|---|---|
| **Rolling** | Replace instances gradually | No downtime, simple |
| **Blue-Green** | Two identical environments; switch traffic | Instant rollback |
| **Canary** | Release to small % of users first | Limits blast radius |
| **Feature flags** | Deploy code, enable feature later | Decouples deploy from release |

**Rollback:** redeploy previous image tag (SHA), or switch traffic back (blue-green). DB changes should be **backward compatible** (use Flyway/Liquibase migrations, expand-then-contract).

---

## 10. Interview Questions and Crisp Answers

**Q1. What is CI/CD and why use it?**
Automates build, test and deployment on every change, giving fast feedback, fewer integration bugs, repeatable releases and faster delivery.

**Q2. Explain GitHub Actions components.**
Workflows (YAML) triggered by events contain jobs, which contain steps. Steps run commands or reusable actions on runners.

**Q3. Difference between a job and a step?**
Jobs run on separate runners, parallel by default and don't share the filesystem (use artifacts/outputs). Steps run sequentially within a job and share the same workspace.

**Q4. How do you pass data between jobs?**
Job `outputs` for small values, **artifacts** (`upload-artifact`/`download-artifact`) for files, and `needs.<job>.outputs.<name>` to consume them.

**Q5. How do you manage secrets?**
GitHub Secrets at repo/org/environment level, accessed via `${{ secrets.NAME }}`; masked in logs. Prefer OIDC for cloud credentials.

**Q6. How do you speed up Maven builds in Actions?**
`cache: maven` in `setup-java`, parallel jobs, skip unnecessary steps, Docker layer caching.

**Q7. How do you add manual approval before production?**
Use `environment: production` with **required reviewers** configured in repo settings.

**Q8. How do you run integration tests needing a database?**
Use `services:` to start a Postgres container (with health checks), or Testcontainers inside the test run.

**Q9. How do you reuse pipeline logic across repos?**
**Reusable workflows** (`workflow_call`) or **composite actions**.

**Q10. GitHub-hosted vs self-hosted runners?**
Hosted: zero maintenance, clean VM per job, limited customization. Self-hosted: custom hardware/network access (internal systems), but you manage security and updates.

**Q11. How do you ensure only tested code reaches main?**
Branch protection with required status checks (CI must pass) and required code reviews.

**Q12. How would you roll back a bad deployment?**
Redeploy the previous immutable image tag (commit SHA) or switch traffic back in blue-green; ensure DB migrations are backward compatible.

**Q13. How do you handle environment-specific configs?**
GitHub Environments with their own secrets/variables, plus Spring profiles (`SPRING_PROFILES_ACTIVE`) or externalized config (ConfigMaps/Key Vault).

**Q14. What triggers can a workflow have?**
`push`, `pull_request`, `schedule` (cron), `workflow_dispatch` (manual), `release`, `workflow_call`, `repository_dispatch`.

**Q15. Describe a CI/CD pipeline you built or would build for a Spring Boot microservice.**
On PR: build, unit and integration tests, Sonar/CodeQL, dependency scan. On merge to main: build Docker image, scan, push to registry with SHA tag, deploy to dev automatically, staging after tests, production with approval, followed by smoke tests, monitoring and rollback plan.

---

## 11. Quick Revision Cheat Sheet

- Files live in **`.github/workflows/*.yml`**
- **Event → Workflow → Jobs → Steps → Actions**, on **Runners**
- Jobs parallel by default; order with **`needs`**
- **`actions/checkout`**, **`actions/setup-java`** (with `cache: maven`), **`upload-artifact`**
- **`services:`** for DB in tests, **`matrix`** for multi-version builds
- **Environments + required reviewers** = approval gates
- **Secrets**, **OIDC**, least-privilege **`permissions`**, pinned actions
- Tag images with **commit SHA**; rollback = redeploy previous tag
- Pipeline: **build → test → scan → package → push → deploy (dev → staging → prod)**

---

*Next topic suggestions: REST APIs, Spring Boot internals, Hibernate/JPA, OData, Microservices, PostgreSQL, Design Patterns.*