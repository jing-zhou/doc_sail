JWT2 Authentication (RSA-verified tokens)

Overview

JWT2 is a lightweight, RSA-verified authentication mode intended for use by clients that present an externally-signed token to access the proxy service. The server validates JWT2 tokens using a configured RSA public key and only enforces two things:

- Signature verification using the configured RSA public key (RS256 is recommended).
- The presence and validity of the "expiresAt" claim: the server checks that the token's expiresAt value is strictly in the future compared to server time.

Key characteristics

- RSA-based verification: The server only needs a public key. The private key used to sign tokens is owned and operated by an external authority (token issuer). The server does not sign JWT2 tokens.
- Minimal validation: Only signature and the "expiresAt" claim are checked. No user lookup, no session state.
- Anonymous access / no billing: A successfully verified JWT2 token is treated as an anonymous credential — the server will not create or attach any billing record. In internal code paths this is represented by returning a null billingId on successful verification.
- Detached signing: Token creation (signing) is intentionally separated from the server: a third-party token issuer (or a separate signing service) produces the token and distributes it to clients. The server only verifies.

Token contract

- Token type: JWS (signed JWT), algorithm RS256 (recommended). The server must be able to verify RS signatures.
- Claim required: "expiresAt" — required. Format recommended: epoch milliseconds (long). Example: 1700000000000
  - When the server verifies token validity it performs:
    - If "expiresAt" is missing or not a numeric millis value, verification fails.
    - If Instant.ofEpochMilli(expiresAt).isAfter(Instant.now()) == false, verification fails (token expired).
- Other claims are ignored by the server (issuer, subject, audience, etc. may be present and consumed by external systems, but are not required by the server).

Configuration

Add these properties to application configuration (example Spring Boot properties):

- auth.jwt2.enabled=false
- auth.jwt2.publicKeyPath=classpath:keys/jwt2_public.pem   # optional, supports classpath: or file:
- auth.jwt2.publicKey=-----BEGIN PUBLIC KEY-----\n...

Multiple public keys and key discovery

The server supports accepting multiple public keys from a single PEM file or via a JWKS URL to accommodate multiple token issuers and key rotation.

- Concatenated PEM file: You may place multiple `-----BEGIN PUBLIC KEY-----` / `-----END PUBLIC KEY-----` blocks in the file referenced by `auth.jwt2.publicKeyPath`. The server should parse and load every public key it finds and attempt verification with each key until one succeeds.
- Multiple file paths: As an alternative operators can provide multiple public key files (e.g. `auth.jwt2.publicKeyPath[0]=...`, `auth.jwt2.publicKeyPath[1]=...`) depending on your configuration system.
- JWKS (recommended for dynamic rotation): Configure a JWKS URL (e.g. `auth.jwt2.jwksUrl=https://issuer.example/.well-known/jwks.json`) — the server can fetch keys and refresh them periodically.

Token selection by kid

When multiple keys are available, tokens should include a `kid` (Key ID) header so the server can select the correct public key quickly. If `kid` is present the server should try to match the `kid` to the loaded keys first; if no match is found it may fall back to trying all loaded keys.

Fail-fast behavior

If `auth.jwt2.enabled=true` and no usable public key (or JWKS) is configured or loadable at startup, the server should fail-fast or start with a clear error depending on your operational preference. The recommended behavior is to fail-fast to avoid silently accepting none or misconfigured state.

Behavior when enabled

- If auth.jwt2.enabled=true the server will load the configured public key on startup. If the key cannot be loaded the server should log an error (recommended: fail-fast so the operator notices the misconfiguration).
- When the server receives a candidate token encoded with the JWT2 crypto byte (implementation detail: e.g. crypto byte 0x08), it will:
  1. Convert token bytes to UTF-8 string (the compact JWS string).
  2. Verify the RS signature against the configured public key.
  3. Parse the claims and check the numeric "expiresAt" claim as epoch millis.
  4. If success, the service returns success and no billingId (represented as null) — the request proceeds as an authenticated anonymous principal.
  5. On any failure (invalid signature, parse error, missing/invalid expiresAt, expired) the server treats the token as invalid and denies the request (map to existing numeric Reason codes used by the project; e.g. TOKEN_INVALID or TOKEN_EXPIRED).

Integration notes

- Header/codec: JWT2 uses the same header flow as other crypto types. Ensure the header decoder accepts the new crypto type byte (e.g. add (byte)0x08 to the valid crypto byte list).
- Secret verification: The server's secret verification component should return null billingId when JWT2 validation succeeds so the billing subsystem is not invoked for JWT2 requests.

Security considerations

- Key rotation: Because the server only needs the public key, rotate signing keys by publishing a new public key and updating server configuration. For smooth rotation, support multiple configured public keys or use a URL-based JWKS if needed.
- Revocation: JWT2 tokens are stateless. There is no built-in token revocation. To allow revocation you must either:
  - Use short token lifetimes and re-issue tokens frequently, or
  - Add a revocation list (e.g., a blacklist keyed by token id) stored by an authoritative service which the server consults, or
  - Use a signed claim referencing a centralized auth service for revocation checks.
- Clock skew: Consider allowing a small clock skew tolerance (e.g., 30 seconds) when validating expiresAt if clients might have slight clock drift.
- Audience/issuer checks: If you need stronger assurance about token origin, include and validate issuer (iss) and audience (aud) claims — the server currently ignores them by design to keep verification minimal.

Token generation (test / example)

1) Generate RSA keypair (local test):

```bash
# 2048-bit RSA key pair
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out jwt2_private.pem
openssl rsa -pubout -in jwt2_private.pem -out jwt2_public.pem
```

2) Create a token (node + jose) — example script (recommended for testing):

```js
// generate-jwt2.js
const fs = require('fs');
const { SignJWT } = require('jose');

(async () => {
  const privateKeyPem = fs.readFileSync('jwt2_private.pem', 'utf8');
  const privateKey = await jose.importPKCS8(privateKeyPem, 'RS256');

  const expiresAt = Date.now() + 60_000; // 60s from now, in ms
  const token = await new SignJWT({ expiresAt })
    .setProtectedHeader({ alg: 'RS256' })
    .sign(privateKey);

  console.log(token);
})();
```

(You can also use other JWT libraries — the important part is that the token contains a numeric "expiresAt" claim in epoch milliseconds and is RS256-signed.)

Validation pseudocode

```java
// Jwt2Service.validate(jwt)
try {
  Jws<Claims> parsed = Jwts.parserBuilder().setSigningKey(publicKey).build().parseClaimsJws(jwt);
  Claims claims = parsed.getBody();
  Long expiresAt = claims.get("expiresAt", Long.class);
  if (expiresAt == null) return false;
  Instant expiry = Instant.ofEpochMilli(expiresAt);
  return expiry.isAfter(Instant.now());
} catch (JwtException | IllegalArgumentException e) {
  // invalid signature, malformed token, or missing claim
  return false;
}
```

Testing guidance

- Unit tests should cover:
  - valid token (signature OK, expiresAt in the future) → verification succeeds
  - expired token → verification fails
  - invalid signature (signed with other key) → verification fails
  - malformed token → verification fails

- Integration test: wire `Jwt2Service` into the secret verifier (SecretImp) and assert that a valid JWT2 token returns null billingId and an invalid token returns the verification-failure sentinel.

Operational notes

- Because JWT2 tokens are bearer tokens anyone possessing a valid token can access the proxy — treat token distribution as sensitive.
- Prefer short-lived tokens to limit blast radius if a token leaks.

References and implementation pointers

- Use a robust JWT library (e.g. io.jsonwebtoken JJWT or jose4j) for parsing and verifying JWS.
- Store the public key file under a known path (e.g. `classpath:keys/jwt2_public.pem`) and document how operators can replace it when rotating keys.
- Map verification failures to numeric Reason codes from the project's `Reason` enum (e.g. TOKEN_EXPIRED, TOKEN_INVALID) so the rest of the service can react consistently.

Questions or next steps

- I can implement the `Jwt2Service` and wire it into `SecretImp` and tests if you want — confirm the crypto byte to use for header decoding (I propose 0x08) and whether the server should fail-fast if `auth.jwt2.enabled=true` but no public key is available.
- I can also add example test keys under `src/test/resources/keys/` and unit tests to the repo.

---

Document created: doc/jwt2-authentication.md
