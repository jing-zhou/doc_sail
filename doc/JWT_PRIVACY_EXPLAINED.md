# JWT Token Privacy Explanation

Recent changes (2025-12-06):
- The server now treats MongoDB `ObjectId` strictly as an internal identifier. `ObjectId` (userId) is never returned to clients in API responses.
- JWT tokens continue to contain only `id` (random UUID) and `expiresAt`. They do NOT include `username` or `userId`.
- The server maps tokenId → ObjectId internally; clients only see the opaque token UUID.

## TL;DR: Your Token Reveals NOTHING About You

The JWT token contains only 2 pieces of information:
1. **A random UUID** (like `a1b2c3d4-e5f6-7890-abcd-ef1234567890`)
2. **An expiration timestamp** (like `1733270400000`)

**Neither of these reveals who you are.**

---

## What's Inside the Token?

### Decoded JWT Example

```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "expiresAt": 1733270400000
}
```

That's it. Nothing else.

---

## What is the "id" Field?

### It's NOT:
- ❌ Your username
- ❌ Your email address
- ❌ Your real name
- ❌ Your user ID in the database
- ❌ Any identifier that can be used to find you
- ❌ Anything that reveals your identity

### It IS:
- ✅ A randomly generated UUID (Universally Unique Identifier)
- ✅ Generated fresh each time you create a new token
- ✅ Completely different every time
- ✅ Used ONLY by the server to look up your account in the database
- ✅ Meaningless to anyone except the server

---

## How Token-to-User Correlation Works

### Step-by-Step Process

1. **You generate a token**
   ```
   Server creates: UUID = "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
   ```

2. **Server saves correlation in database**
   ```
   Database: "a1b2c3d4-..." → User "alice" (ID: 507f1f77...)
   ```

3. **Server gives you the JWT**
   ```
   JWT contains: {"id":"a1b2c3d4-...","expiresAt":1733270400000}
   ```

4. **You use the token to connect**
   ```
   Client sends: JWT with id="a1b2c3d4-..."
   ```

5. **Server validates**
   ```
   Server looks up: "a1b2c3d4-..." in database
   Server finds: User "alice"
   Server allows: Connection for alice
   ```

### Critical Point

The correlation (`a1b2c3d4-...` → `alice`) exists **ONLY** in the server's database.

- The token itself doesn't contain `"alice"`
- The token doesn't contain your email
- The token doesn't contain any personal data
- **The UUID is just a lookup key**

---

## Privacy Protection in Action

### Scenario: Token is Intercepted

Let's say someone intercepts your token and decodes it:

```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "expiresAt": 1733270400000
}
```

**What can they learn about you?**

- ❌ They don't know your username
- ❌ They don't know your email
- ❌ They don't know your name
- ❌ They don't know anything about you

**What CAN they do?**

- ⚠️ They can use the token to access the proxy (until you revoke it)
- ✅ But they still don't know who you are
- ✅ You can revoke the token and generate a new one
- ✅ New token will have a completely different UUID

---

## Comparison with Other Systems

### Standard OAuth2 JWT (Bad for Privacy)
```json
{
  "sub": "alice@example.com",        // ❌ Reveals email
  "name": "Alice Smith",              // ❌ Reveals real name
  "email": "alice@example.com",       // ❌ Reveals email
  "aud": "my-app",
  "iat": 1516239022,
  "exp": 1516242622
}
```

**Problem:** Anyone who intercepts this token knows:
- Your email address
- Your real name
- What app you're using

### Illiad Proxy JWT (Good for Privacy)
```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",    // ✅ Random UUID
  "expiresAt": 1733270400000                        // ✅ Just a timestamp
}
```

**Benefit:** Anyone who intercepts this token knows:
- Nothing about you
- Just a random ID that means nothing to them

---

## Technical Details

### UUID Generation

```java
// Each token generation creates a NEW random UUID
String tokenId = UUID.randomUUID().toString();
// Example outputs:
// "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
// "f9e8d7c6-b5a4-3210-9876-543210fedcba"
// "1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d"
// Every single one is different
```

### Database Correlation

```javascript
// MongoDB tokens collection
{
  "_id": ObjectId("..."),
  "userId": "507f1f77bcf86cd799439011",        // Your actual user ID
  "tokenId": "a1b2c3d4-e5f6-7890-abcd-...",    // Random UUID in JWT
  "createdAt": ISODate("2024-12-03T00:00:00Z"),
  "expiresAt": ISODate("2025-01-02T00:00:00Z"),
  "revoked": false
}
```

The server:
1. Receives JWT with `id="a1b2c3d4-..."`
2. Queries database for token with `tokenId="a1b2c3d4-..."`
3. Gets back `userId="507f1f77..."`
4. Queries users collection for user with `_id="507f1f77..."`
5. Finds your account and allows connection

**No one except the server can do this lookup.**

---

## Why This Matters

### For Privacy
- ✅ Tokens can be safely logged without revealing user identities
- ✅ Network monitoring can't identify users from tokens
- ✅ Leaked tokens don't expose personal information
- ✅ Public Wi-Fi interception reveals nothing about you

### For Security
- ✅ Different UUID for each token makes tracking harder
- ✅ Revoking a token just removes the database entry
- ✅ No need to change username or email if token is compromised
- ✅ Easy to audit token usage without exposing user data

### For Compliance
- ✅ GDPR-friendly: no personal data in tokens
- ✅ Minimal data exposure
- ✅ Easy to anonymize logs
- ✅ Privacy by design

---

## Summary

### The `id` Field:
- Is a **random UUID**
- Reveals **nothing** about the user
- Only **correlates** to user in server database
- **Changes** with every new token
- Serves **only** as a lookup key

### The `expiresAt` Field:
- Is just a **timestamp**
- Tells when token **expires**
- Contains **no user information**

### Together:
- ✅ Minimal token size
- ✅ Maximum privacy
- ✅ Secure validation
- ✅ Easy revocation
- ✅ No personal data exposure

**Your identity is safe.** The token is just a temporary, revocable access credential that reveals nothing about who you are.

---

## Questions?

**Q: Can someone identify me from my token?**  
A: No. The UUID is random and meaningless to everyone except the server.

**Q: What if someone decodes my JWT?**  
A: They'll only see a random UUID and a timestamp. Nothing about you.

**Q: Is the UUID related to my username somehow?**  
A: No. It's completely random, generated fresh each time.

**Q: Can the server admin see my username from the UUID?**  
A: Only by querying the database. The UUID itself doesn't contain it.

**Q: Why not just use my username in the token?**  
A: For privacy! Your username shouldn't be exposed in the token.

**Q: What happens if I generate a new token?**  
A: You get a completely different random UUID. **ALL old tokens become invalid immediately** - even if they haven't expired yet. The server only accepts the most recent token because it updates `currentTokenId` in your user record.

**Q: Can I have multiple valid tokens at the same time?**  
A: No. Only ONE token per user is valid at any time. Generating a new token automatically invalidates all previous ones by updating `currentTokenId`.

**Q: My old token hasn't expired yet, will it still work?**  
A: No. Even if the expiration date hasn't passed, the token is invalid because its `id` doesn't match your current `currentTokenId` in the database.

**Q: Does the server store my JWT token?**  
A: No! The server only stores metadata (tokenId, userId, expiresAt). The actual JWT string is never saved anywhere on the server.

**Q: Can I retrieve my token if I lose it?**  
A: No. The server doesn't have it. You must generate a new token (which invalidates the old one).

**Q: Why doesn't the server save my JWT?**  
A: Security! If the server database is compromised, attackers can't steal JWTs because they're not stored there.

**Q: What if I click "Cancel" instead of "Download"?**  
A: The JWT is discarded. You'll need to generate a new one. The server never saves it.

---

**Bottom line:** Your JWT token is designed for maximum privacy. The `id` field is just a random number used as a database lookup key—nothing more. **The server never stores your JWT** - it's shown once and then you're responsible for saving it.
