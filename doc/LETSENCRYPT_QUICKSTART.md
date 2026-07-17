Let's Encrypt TLS-ALPN Quickstart

This doc explains how to enable the in-process Netty TLS-ALPN solver and how to run it locally for development.

Enable Netty TLS-ALPN solver

1) Set the property in your `application.properties` (or pass as JVM arg):

letsencrypt.tls-alpn-solver=netty

2) Ensure the server's local port (Params.localPort) is set to the port to bind (443 for TLS-ALPN).
   For local tests you can use an ephemeral port (0) — see the integration test below.

params.localPort=443

or pass as JVM arg:

-Dparams.localPort=443

Permissions to bind low ports (443)

Binding port 443 requires privilege. Choose one of these options:

- Use setcap to allow the java executable to bind privileged ports:

sudo setcap 'cap_net_bind_service=+ep' $(readlink -f $(which java))

- Use authbind to allow non-root processes to bind 443 (Ubuntu/Debian):

sudo apt-get install authbind
sudo touch /etc/authbind/byport/443
sudo chown youruser /etc/authbind/byport/443
sudo chmod 755 /etc/authbind/byport/443
# Then run your Java process under authbind
authbind --deep java -jar build/libs/server.jar

- Run the process as root (not recommended)

Manual presentation (ManualTlsAlpnSolver)

If you cannot bind 443 in-process, the application falls back to `ManualTlsAlpnSolver` which writes the challenge information to disk
and logs instructions to let you present the ACME challenge using external tooling like `openssl s_server`.

Example (using openssl, external):

# On a system that can present a certificate on port 443
# produce cert.pem and key.pem that contain the ACME TLS-ALPN validation extension
openssl s_server -accept 443 -cert cert.pem -key key.pem -alpn acme-tls/1

Ephemeral-port integration test (CI-friendly)

To exercise the Netty solver in CI without requiring privileged ports, an integration test binds to an ephemeral
port (port 0) and asserts the solver can start and stop cleanly:

- Test: `NettyTlsAlpnSolverIntegrationTest.bindAndStop_onEphemeralPort`

Command (run only the integration test):

```bash
./gradlew test --tests com.illiad.server.security.acme.NettyTlsAlpnSolverIntegrationTest
```

New Dev & CI helpers (added)

This project now includes a few convenience helpers and CI wiring to make TLS-ALPN validation repeatable and automated.

1) Gradle tasks

- `testTlsAlpn` — a focused test task that runs only TLS-ALPN related tests (fast; suitable for CI pre-checks):

```bash
./gradlew testTlsAlpn
```

- `generateAcmeTlsCert` — a small test-scoped JavaExec task that runs a utility to generate a PEM key+cert embedding the
  ACME TLS validation bytes (useful for manual testing with `openssl s_server`). Usage:

```bash
./gradlew generateAcmeTlsCert -PcertDomain=example.com -PcertValidationHex=<hex> -PcertOutPrefix=cert
```

This produces `<prefix>-cert.pem` and `<prefix>-key.pem` (e.g. `cert-cert.pem` and `cert-key.pem`). The utility is under
`src/test/java/com/illiad/server/security/acme/tools/AcmeTlsCertGenerator.java` and is intended for development/test use only.

2) ALPN verification scripts

- `scripts/check-tls-alpn.sh` — checks whether a server at HOST:PORT negotiates ALPN `acme-tls/1`.
  - Supports `--timeout N`, `--json`, and a new `--servername <name>` (alias `-s`) to set SNI separately from HOST.
  - Defaults: HOST=127.0.0.1, PORT=443, TIMEOUT=5, SERVERNAME=HOST
  - Exit codes: 0 (present), 2 (not present), 3 (error/timeout)

- `scripts/check-tls-alpn.json.sh` — wrapper that prints JSON for CI usage and accepts an optional 4th positional arg to pass
  the `SERVERNAME` to the underlying script.

Examples:

```bash
# human readable
./scripts/check-tls-alpn.sh 127.0.0.1 8443 --timeout 5 --servername example.com

# CI / JSON
./scripts/check-tls-alpn.json.sh 127.0.0.1 8443 5 example.com
```

3) CI workflow: `/.github/workflows/tls-alpn-check.yml`

The repository includes a ready-to-use GitHub Actions workflow that performs two phases:

- `build-and-test` (fast): runs `testTlsAlpn` and additionally runs `shellcheck` on the scripts to catch common shell issues early.
- `runtime-alpn-check` (optional / dispatchable): starts the app (via `./gradlew bootRun`) bound to an ephemeral port, parses
  the `boot.log` for the line logged by the solver:

  ```text
  TLS-ALPN solver bound to port <port> for domain <domain>
  ```

  it extracts `<port>` and runs `./scripts/check-tls-alpn.json.sh 127.0.0.1 <port> <timeout> [servername]`.
  If the check fails, the workflow uploads the captured `build/logs/boot.log` as an artifact for debugging.

Workflow inputs (available when triggering `workflow_dispatch`):

- `wait_timeout` — seconds to wait for the server to report its bound port (default 60)
- `alpn_timeout` — timeout for the openssl ALPN check (default 5)
- `alpn_host` — host to use for the ALPN check (default 127.0.0.1)
- `alpn_domain` — SNI/servername to present to the server for verification (default localhost)

The `runtime-alpn-check` job will use these inputs; it will fail early with log tail output if the port cannot be discovered.

How to run everything locally (quick guide)

1) Run the focused TLS-ALPN tests:

```bash
./gradlew testTlsAlpn --info
```

2) Generate a test cert/key (for manual OpenSSL presentation):

```bash
./gradlew generateAcmeTlsCert -PcertDomain=localhost -PcertValidationHex=010203 -PcertOutPrefix=testcert
# produces testcert-cert.pem and testcert-key.pem
```

3) Present the generated cert with OpenSSL (example uses port 8443 to avoid privileged ports):

```bash
openssl s_server -accept 8443 -cert testcert-cert.pem -key testcert-key.pem -alpn acme-tls/1 &
# note: background process; capture pid if you need to stop it
```

4) Verify ALPN negotiation (use SNI if needed):

```bash
# human-readable
./scripts/check-tls-alpn.sh 127.0.0.1 8443 --timeout 5 --servername localhost

# JSON output (CI-friendly)
./scripts/check-tls-alpn.json.sh 127.0.0.1 8443 5 localhost
```

5) Cleanup the OpenSSL server and artifacts when done:

```bash
kill $(pgrep -f 'openssl s_server') || true
rm testcert-cert.pem testcert-key.pem || true
```

Notes & Troubleshooting

- ShellCheck is installed in the CI job to lint `scripts/*.sh`. Locally you can install it via `sudo apt install shellcheck` or use the
  Docker image `koalaman/shellcheck` to lint without installing.

- The GitHub Actions `runtime-alpn-check` step reads `build/logs/boot.log` produced by `./gradlew bootRun` and looks for the
  exact solver log message `TLS-ALPN solver bound to port <port> for domain <domain>`; if your logs are customized, adjust the
  workflow extraction regex accordingly.

- If the server takes longer than usual to initialize in CI, increase the `wait_timeout` input when dispatching the workflow.

- `generateAcmeTlsCert` is a test-only convenience for manual testing; production should rely on the ACME client (acme4j) flow
  and the server's in-process `NettyTlsAlpnSolver`.

Files of interest

- `/.github/workflows/tls-alpn-check.yml` — CI workflow (build-and-test + runtime check + ShellCheck + artifact upload)
- `scripts/check-tls-alpn.sh` — ALPN checker with `--servername` support
- `scripts/check-tls-alpn.json.sh` — JSON wrapper (CI)
- `src/test/java/com/illiad/server/security/acme/tools/AcmeTlsCertGenerator.java` — test helper to generate certs for manual OpenSSL presentation
- `build.gradle.kts` additions:
  - `testTlsAlpn` task (focused tests)
  - `generateAcmeTlsCert` JavaExec task
  - `runAlpnCheckScript` Exec task (runs JSON wrapper)

If you want, I can also add a short `doc/USAGE_TLS_ALPN.md` example that contains a copy-pastable CI job snippet and an expanded
OpenSSL example for creating the DER extension by hand — say the word and I'll add it.
