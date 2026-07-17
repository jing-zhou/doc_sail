Integration tests (Pebble / ACME)

This project includes integration tests that exercise the Let's Encrypt (ACME) integration using a local ACME test server (Pebble).

Prerequisites
- Docker running locally
- scripts/run-pebble.sh available (this project includes scripts to start a Pebble container and save its root CA at `pebble-data/pebble-root-ca.pem`)

Quick steps (local)
1. Start Pebble and save its root CA:

```bash
bash scripts/run-pebble.sh --recreate
```

2. Prepare the Pebble truststore and run integration tests:

```bash
./gradlew preparePebbleTruststore integrationTest --info
```

Notes
- `preparePebbleTruststore` will import `pebble-data/pebble-root-ca.pem` into `build/pebble-truststore.jks` and `integrationTest` JVM will use that keystore via `-Djavax.net.ssl.trustStore=...`.
- Integration tests use an injected `org.shredzone.acme4j.connector.DefaultConnection` built from `org.shredzone.acme4j.connector.jdk.JdkHttpConnector`, allowing tests to provide a custom `java.net.http.HttpClient` with the Pebble CA trusted.
- If your CI runner cannot access the internet to download `acme4j-connector-jdk`, pre-populate the Gradle cache or vendor the JAR into the `libs/` directory and add it to the buildscript.

Troubleshooting
- If tests fail with PKIX errors, re-run `scripts/run-pebble.sh` and `preparePebbleTruststore` to ensure the PEM is up-to-date.
- If Gradle cannot download dependencies, run with `--refresh-dependencies` or ensure the CI network permits access to Maven Central.

Contact
If you want me to wire these integration tests into CI (optional job), I can add a GitHub Actions example that starts Pebble and runs `./gradlew integrationTest`.

