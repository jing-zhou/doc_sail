# CI and Local Reproduction Guide

This document explains the project's CI setup (GitHub Actions) and how to reproduce CI runs locally.

## CI workflow (GitHub Actions)

Location: `.github/workflows/ci.yml`

What it does:
- Runs on `push` and `pull_request` events (branches: `main`, `master` and others)
- Uses `ubuntu-latest` runner
- Starts `mongo:6` and `mailhog/mailhog` services in the runner
- Sets up JDK 17 and caches Gradle
- Runs `./gradlew check` (static analysis + tests) and `./gradlew assemble`
- Uploads test reports and built artifacts

### Environment variables
- `SPRING_DATA_MONGODB_URI` — points to `mongodb://127.0.0.1:27017/illiad` in CI
- `SPRING_MAIL_HOST` / `SPRING_MAIL_PORT` — configure MailHog in CI

## Local reproduction (recommended)

Use the provided `scripts/ci-smoke.sh` script which attempts to start Docker services (Mongo and optional MailHog) and then runs tests and build.

1. Ensure Docker is running locally.

2. Run the smoke script (default starts MailHog):

```bash
./scripts/ci-smoke.sh
```

3. Or run without MailHog (useful if you mock mail in tests):

```bash
./scripts/ci-smoke.sh --no-mailhog
```

Behavior:
- If port 27017 is already in use the script assumes a local Mongo is running and will not start a Docker container for it.
- The script waits for Mongo to be reachable and then runs:

```bash
./gradlew test
./gradlew assemble
```

## Reproducing CI manually (without the smoke script)

Start Mongo via Docker if you don't have one:

```bash
docker run --rm -d --name local-mongo -p 27017:27017 mongo:6
```

Start MailHog if you need it:

```bash
docker run --rm -d --name local-mailhog -p 1025:1025 -p 8025:8025 mailhog/mailhog:latest
```

Run tests and build:

```bash
./gradlew check
./gradlew assemble
```

## CI best practices
- Tests should not rely on external network calls (Let’s Encrypt etc.). Mock external services in unit tests.
- Keep integration tests separate (long-running); use a dedicated integration job in CI that uses docker-compose.
- Preserve test artifacts and reports (the workflow uploads `build/reports/tests/test`)

## Common issues
- `Bind for 0.0.0.0:27017 failed: port is already allocated` — stop local Mongo or allow the script to detect it.
- Mail tests failing — ensure `SPRING_MAIL_HOST` and `SPRING_MAIL_PORT` point to MailHog, or mock JavaMailSender in tests.

## Extending the workflow
- Add `jacoco` upload and coverage gating if you want coverage thresholds.
- Add static analysis fail-fast or allow warnings depending on team preference.

---

This file was generated/updated automatically by the developer helper. If you need changes to the CI behavior (matrix builds, windows/mac runners, or additional integrations), mention them and I'll update the workflow accordingly.

