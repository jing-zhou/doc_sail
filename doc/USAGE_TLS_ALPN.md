Usage: TLS-ALPN (developer + CI guide)

This document gives a compact, copy-pastable guide to exercising TLS‑ALPN‑01 locally and in CI for development and debugging.
It complements `doc/LETSENCRYPT_QUICKSTART.md` with hands-on commands, an example CI snippet, and notes about certificate creation.

Goals
- Verify the server can present an ACME TLS certificate advertising ALPN `acme-tls/1`.
- Provide quick tools and commands for local developer testing.
- Provide a reproducible CI pattern that is fast and debuggable.

Quick summary of recommended flow
1) Use the project's Java helper to generate a TLS cert that embeds the ACME TLS‑ALPN validation bytes (safe and reliable).
2) Present that cert locally with `openssl s_server` (non-privileged port for dev).
3) Use `./scripts/check-tls-alpn.sh` or `./scripts/check-tls-alpn.json.sh` to assert ALPN `acme-tls/1` is negotiated.
4) For CI: run `./gradlew testTlsAlpn` (fast unit tests) then run the runtime `bootRun` + ALPN check (workflow provided).

Contents
- Local quick-run (developer)
- Generating test certs (preferred: Java helper)
- Optional: manual OpenSSL/DER approach (advanced operators)
- CI pattern and example workflow snippet (ready-to-use)
- Troubleshooting notes

Local quick-run (developer)

1) Run focused TLS-ALPN tests (fast):

```bash
# runs only the TLS-ALPN related unit/integration tests
./gradlew testTlsAlpn --info
```

2) Generate a test ACME TLS certificate and key (test-only helper):

```bash
# certValidationHex should be the hex representation of the ACME validation bytes
# for testing you can use a short sample hex like 010203
./gradlew generateAcmeTlsCert \
  -PcertDomain=localhost \
  -PcertValidationHex=010203 \
  -PcertOutPrefix=testcert

# produces testcert-cert.pem and testcert-key.pem
```

3) Present the generated cert with OpenSSL on a non-privileged port (8443):

```bash
# present the cert and advertise ALPN acme-tls/1
openssl s_server -accept 8443 -cert testcert-cert.pem -key testcert-key.pem -alpn acme-tls/1 &
# (capture the background PID if you want to kill it later)
```

4) Check ALPN negotiation (human readable):

```bash
./scripts/check-tls-alpn.sh 127.0.0.1 8443 --timeout 5 --servername localhost
# Exit codes: 0 = present, 2 = not present, 3 = timeout/error
```

5) Check ALPN negotiation (CI / JSON friendly):

```bash
./scripts/check-tls-alpn.json.sh 127.0.0.1 8443 5 localhost
# prints JSON: {"present": true, "alpn": "acme-tls/1"}
```

6) Cleanup:

```bash
kill $(pgrep -f 'openssl s_server') || true
rm testcert-cert.pem testcert-key.pem || true
```

New: Smoke script `scripts/run-alpn-smoke.sh` and `--keep` option

A convenience smoke-test script is included at `scripts/run-alpn-smoke.sh`. It automates the local smoke flow:
- generates a test cert/key via `generateAcmeTlsCert`
- starts `openssl s_server` on a free high port (or a provided --port)
- runs the ALPN JSON checker (with retries)
- cleans up generated artifacts by default

The script accepts `--keep` which will preserve generated artifacts and the openssl log for inspection. Example:

```bash
# run smoke test and preserve artifacts for debugging
./scripts/run-alpn-smoke.sh --keep

# run smoke test on specific port and servername
./scripts/run-alpn-smoke.sh --port 8443 --servername localhost --timeout 10 --keep
```

Generated artifacts when `--keep` is used:
- `<prefix>-cert.pem` and `<prefix>-key.pem` created in the repo root (prefix = smoketest-<timestamp>)
- openssl server log: `/tmp/openssl-smoketest-<prefix>.log`

Generating test certs — preferred approach (use the Java helper)

Why the helper: the ACME TLS extension must contain the raw validation bytes exactly. Creating that extension reliably with OpenSSL is cumbersome.
The project includes a small test-only CLI `AcmeTlsCertGenerator` whose Gradle task `generateAcmeTlsCert` produces a cert/key pair embedding the validation bytes.

Usage example (repeat):

```bash
./gradlew generateAcmeTlsCert -PcertDomain=localhost -PcertValidationHex=010203 -PcertOutPrefix=testcert
```

- `certValidationHex` is the hex string of the bytes returned by ACME client: `TlsAlpn01Challenge#getAcmeValidation()`.
- The helper writes `testcert-cert.pem` and `testcert-key.pem` that are ready to present via `openssl s_server`.

Advanced: OpenSSL + DER extension (operator note)

If you prefer to use OpenSSL exclusively, you must construct a DER-formatted extension whose value is the raw validation bytes and attach it to the X.509 certificate. This is platform- and OpenSSL-version-dependent and error-prone. The recommended options are:

- Use the Java helper above (simple, reproducible).
- If you must use OpenSSL only, create a small Java or Python helper that writes a DER file with the exact bytes and calls `openssl x509` to include it — or use `openssl asn1parse` and `openssl ca` with a custom extfile. If you want, I can add a small script that does this (requires careful handling of DER embedding).

CI pattern (recommended)

The repository includes a CI workflow `/.github/workflows/tls-alpn-check.yml` with three related jobs:

- `build-and-test`: executes `./gradlew testTlsAlpn` (fast focused tests) and runs ShellCheck on our helper scripts to catch shell issues.
- `runtime-alpn-check`: (dispatchable) runs `./gradlew bootRun -Dletsencrypt.tls-alpn-solver=netty -Dparams.localPort=0` and captures `build/logs/boot.log`.
  The workflow then watches the log for the line printed by `NettyTlsAlpnSolver` and runs the ALPN JSON check against the discovered ephemeral port.
- `smoke-alpn` (manual): optional job that runs the `scripts/run-alpn-smoke.sh` when `workflow_dispatch` input `run_smoke=true`.
- `auto-smoke-alpn` (automatic): runs automatically on pushes to the `main` branch and on tag pushes (controlled by triggers) and preserves artifacts (it invokes smoke script with `--keep`).

Workflow inputs (available when triggering `workflow_dispatch`):

- `wait_timeout` — seconds to wait for the server to report its bound port (default 60)
- `alpn_timeout` — timeout for the openssl ALPN check (default 5)
- `alpn_host` — host to use for the ALPN check (default 127.0.0.1)
- `alpn_domain` — SNI/servername to present to the server for verification (default localhost)
- `run_smoke` — set to `true` to run the manual `smoke-alpn` job when dispatching the workflow

How the automatic `auto-smoke-alpn` job behaves

- triggers: pushes to `main` and tag pushes (configured in `on:` triggers)
- depends on `build-and-test` (so focused tests and script linting are performed first)
- runs the smoke test with `--keep` so artifacts/logs are preserved in `/tmp` on the runner
- uploads the saved `/tmp/openssl-smoketest-*.log` as an artifact for diagnostics (on success and failure)

CI snippet (simplified)

```yaml
# part of .github/workflows/tls-alpn-check.yml
# auto-smoke-alpn runs on main and tag pushes (see file)
jobs:
  auto-smoke-alpn:
    runs-on: ubuntu-latest
    needs: build-and-test
    if: startsWith(github.ref, 'refs/heads/main') || startsWith(github.ref, 'refs/tags/')
    steps:
      - uses: actions/checkout@v4
      - name: Run automated smoke test
        run: |
          chmod +x scripts/run-alpn-smoke.sh
          ./scripts/run-alpn-smoke.sh --keep --timeout 10 --servername localhost
      - name: Upload logs
        uses: actions/upload-artifact@v4
        with:
          name: tls-alpn-auto-smoke-log
          path: /tmp/openssl-smoketest-*.log
```

Troubleshooting

- If the smoke test fails on CI, download the uploaded `tls-alpn-auto-smoke-log` artifact and inspect the openssl log for clues.
- If the server in CI fails to bind or initialize the TLS-ALPN solver, check `build/logs/boot.log` (runtime job uploads it on failure).
- If ShellCheck warnings are raised, fix scripts accordingly; ShellCheck is run in CI to catch common errors before runtime steps.

Security note

- `run-alpn-smoke.sh` is a developer helper and test utility; do not use it in production environments. The `generateAcmeTlsCert` helper is test-only and should not be used to generate production certificates.

If you want, I can add a short `scripts/collect-smoke-artifacts.sh` helper to compress and upload smoke artifacts from the runner for faster debugging; tell me and I will add it.

---

Document created: `doc/USAGE_TLS_ALPN.md`

