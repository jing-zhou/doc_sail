# Token Invalidation - Visual Guide

## 🔑 One Token Per User Rule

**CRITICAL: Only the MOST RECENT token works!**

When you generate a new token, **ALL** previous tokens are **immediately** invalidated.

---

## Visual Timeline

### Day 1: First Token

```
┌─────────────────────────────────────────────────┐
│ User generates Token A                          │
│ JWT: eyJhbGciOiJIUzI1NiJ9.eyJpZCI6ImFhYS0xMTEi │
│ Token ID inside JWT: "aaa-111"                  │
└─────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────┐
│ Database updates:                               │
│ users.currentTokenId = "aaa-111"                │
└─────────────────────────────────────────────────┘
                      ↓
              ✅ Token A WORKS
```

### Day 5: Second Token (Invalidates First)

```
┌─────────────────────────────────────────────────┐
│ User generates Token B                          │
│ JWT: eyJhbGciOiJIUzI1NiJ9.eyJpZCI6ImJiYi0yMjIi │
│ Token ID inside JWT: "bbb-222"                  │
└─────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────┐
│ Database updates:                               │
│ users.currentTokenId = "bbb-222"  ← CHANGED!    │
└─────────────────────────────────────────────────┘
                      ↓
         ✅ Token B WORKS
         ❌ Token A STOPS WORKING (invalidated!)
```

### Day 10: Third Token (Invalidates Second)

```
┌─────────────────────────────────────────────────┐
│ User generates Token C                          │
│ JWT: eyJhbGciOiJIUzI1NiJ9.eyJpZCI6ImNjYy0zMzMi │
│ Token ID inside JWT: "ccc-333"                  │
└─────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────┐
│ Database updates:                               │
│ users.currentTokenId = "ccc-333"  ← CHANGED!    │
└─────────────────────────────────────────────────┘
                      ↓
         ✅ Token C WORKS
         ❌ Token B STOPS WORKING (invalidated!)
         ❌ Token A STILL doesn't work
```

---

## Database State Comparison

### After Token A Generated

```javascript
// users collection
{
  _id: "507f1f77...",
  username: "alice",
  currentTokenId: "aaa-111"  ← Only this token ID is valid
}

// tokens collection
[
  {tokenId: "aaa-111", userId: "507f...", expiresAt: "2025-02-01"}
]

Valid tokens: Token A ✅
```

### After Token B Generated

```javascript
// users collection
{
  _id: "507f1f77...",
  username: "alice",
  currentTokenId: "bbb-222"  ← Changed! Only this is valid now
}

// tokens collection
[
  {tokenId: "aaa-111", userId: "507f...", expiresAt: "2025-02-01"},  // Still in DB but invalid
  {tokenId: "bbb-222", userId: "507f...", expiresAt: "2025-03-01"}
]

Valid tokens: Token B ✅
Invalid tokens: Token A ❌ (doesn't match currentTokenId)
```

### After Token C Generated

```javascript
// users collection
{
  _id: "507f1f77...",
  username: "alice",
  currentTokenId: "ccc-333"  ← Changed again! Only this is valid
}

// tokens collection
[
  {tokenId: "aaa-111", userId: "507f...", expiresAt: "2025-02-01"},  // Invalid
  {tokenId: "bbb-222", userId: "507f...", expiresAt: "2025-03-01"},  // Invalid
  {tokenId: "ccc-333", userId: "507f...", expiresAt: "2025-04-01"}   // Valid
]

Valid tokens: Token C ✅
Invalid tokens: Token A ❌, Token B ❌ (don't match currentTokenId)
```

---

## Validation Flow

```
┌──────────────────────────────────────┐
│ Client connects with Token B        │
│ (generated on Day 5)                 │
└───────────────┬──────────────────────┘
                ↓
┌──────────────────────────────────────┐
│ Server decodes JWT                   │
│ Extracts: id = "bbb-222"             │
└───────────────┬──────────────────────┘
                ↓
┌──────────────────────────────────────┐
│ Server queries database:             │
│ tokens.findOne({tokenId: "bbb-222"}) │
│ Returns: {userId: "507f...", ...}    │
└───────────────┬──────────────────────┘
                ↓
┌──────────────────────────────────────┐
│ Server queries database:             │
│ users.findOne({_id: "507f..."})      │
│ Returns: {currentTokenId: "ccc-333"} │
└───────────────┬──────────────────────┘
                ↓
┌──────────────────────────────────────┐
│ Server checks:                       │
│ "bbb-222" === "ccc-333" ?            │
│ NO! ❌                               │
└───────────────┬──────────────────────┘
                ↓
┌──────────────────────────────────────┐
│ REJECT: Token is invalidated        │
│ (superseded by newer token)          │
└──────────────────────────────────────┘
```

**Even though Token B:**
- ✅ Has valid signature
- ✅ Has not expired
- ✅ Exists in tokens database
- ❌ **Doesn't match currentTokenId** → REJECTED

---

## Expiration vs Invalidation

### Scenario 1: Token Expires

```
Token A generated: 2024-12-01, expires 2025-01-01

Timeline:
├─ 2024-12-01: Token A created ✅
├─ 2024-12-15: Token A works ✅
├─ 2024-12-30: Token A works ✅
└─ 2025-01-02: Token A expired ❌ (expiresAt passed)
```

### Scenario 2: Token Invalidated by New Generation

```
Token A generated: 2024-12-01, expires 2025-01-01
Token B generated: 2024-12-05, expires 2025-02-01

Timeline:
├─ 2024-12-01: Token A created ✅
├─ 2024-12-03: Token A works ✅
├─ 2024-12-05: Token B created → Token A INVALIDATED ❌
├─ 2024-12-06: Token A doesn't work ❌ (invalidated, not expired)
├─ 2024-12-10: Token A doesn't work ❌ (still invalidated)
├─ 2024-12-30: Token A doesn't work ❌ (still invalidated)
└─ 2025-01-02: Token A doesn't work ❌ (invalidated AND expired)
```

**Key Point:** Token A stops working on 2024-12-05 (when Token B is generated), **not** on 2025-01-02 (when it expires).

---

## Why Only One Token?

### Security

```
❌ Multiple tokens allowed:
   - Lost token? Hard to know which one
   - Compromised token? Must track all tokens
   - Revoke all? Complicated

✅ Single token (current implementation):
   - Lost token? Generate new one → old one stops instantly
   - Compromised token? Generate new one → problem solved
   - Simple: Only one currentTokenId to check
```

### Simplicity

```
❌ Multiple tokens:
   users.validTokens = ["aaa-111", "bbb-222", "ccc-333"]
   Validation: Check if token in array
   Revocation: Remove from array

✅ Single token:
   users.currentTokenId = "ccc-333"
   Validation: Check if token === currentTokenId
   Revocation: Generate new token (automatic)
```

---

## Common Questions

### Q: I have Token A (not expired). I generate Token B. Does Token A still work?

**A:** ❌ **NO!** Token A stops working **immediately** when Token B is generated, even if Token A hasn't expired yet.

### Q: Can I use both Token A and Token B?

**A:** ❌ **NO!** Only the MOST RECENT token (Token B) works. Token A is invalid.

### Q: I lost my token. Can I get it back?

**A:** ❌ **NO!** The server doesn't store JWT tokens. **Solution:** Generate a new token (the old one becomes invalid anyway).

### Q: I want to revoke my token. How?

**A:** Generate a new token! The old one is automatically invalidated. No manual revocation needed.

### Q: Can I have one token for mobile and one for desktop?

**A:** ❌ **NO!** Only one token per user. Use the same token on all devices, or generate a new one (which invalidates the old one on other devices).

---

## User Impact

### What Users Need to Know

1. **One Token at a Time**
   - You can only have ONE valid token
   - Generating new token invalidates ALL old tokens

2. **Update All Clients**
   - If you use the token on multiple devices
   - Generate new token → update ALL devices
   - Old token stops working everywhere

3. **Lost Token Recovery**
   - Generate new token (free, unlimited)
   - Old token becomes invalid automatically
   - No need to "revoke" old token

4. **Token Rotation**
   - Want to change token? Just generate new one
   - Old one stops working instantly
   - No waiting, no manual revocation

---

## Code Implementation

### Token Generation (UserService.java)

```java
public Mono<String> generateToken(String userId, long expirationMinutes) {
    JwtService.TokenPair tokenPair = jwtService.generateToken(expirationMinutes);
    
    // Save metadata
    TokenInfo tokenInfo = new TokenInfo();
    tokenInfo.setTokenId(tokenPair.tokenId);
    
    return tokenRepository.save(tokenInfo)
        .flatMap(saved -> userRepository.findById(userId)
            .flatMap(user -> {
                // THIS LINE invalidates all previous tokens!
                user.setCurrentTokenId(tokenPair.tokenId);
                return userRepository.save(user);
            }))
        .thenReturn(tokenPair.jwt);
}
```

### Token Validation (UserService.java)

```java
public Mono<User> validateToken(String jwtToken) {
    String tokenId = jwtService.validateAndGetTokenId(jwtToken);
    
    return tokenRepository.findByTokenId(tokenId)
        .filter(tokenInfo -> !tokenInfo.isRevoked() && 
                tokenInfo.getExpiresAt().isAfter(Instant.now()))
        .flatMap(tokenInfo -> userRepository.findById(tokenInfo.getUserId())
            // THIS LINE rejects old tokens!
            .filter(user -> tokenId.equals(user.getCurrentTokenId())));
}
```

---

## Summary

### ✅ How It Works
- Server stores `currentTokenId` in user record
- New token → `currentTokenId` updated → old tokens invalid
- Validation checks if token ID matches `currentTokenId`
- Users can generate new token **anytime** they want

### ⏰ Expiration Enforcement
- `expiresAt` is **mandatory** - token stops working after this date
- Users **must** generate new token before expiration
- Cannot extend or renew expired tokens
- Recommended: Generate new token before expiration (don't wait)

### ❌ What Doesn't Work
- Multiple valid tokens per user
- Old tokens after new generation
- Tokens that don't match `currentTokenId`
- Extending expired tokens
- Using token after `expiresAt` date

### 🔒 Security Benefits
- Generate new token **anytime** (not just when lost)
- Lost token? Instant invalidation (generate new one)
- Compromised token? Instant revocation (generate new one)
- Regular rotation encouraged (generate every 30 days)
- Simple validation (one ID to check)
- No complex token management

### 📝 User Workflow
- Generate new token **whenever needed**
- Generate **before** expiration (enforcement)
- Update ALL devices/clients when generating new token
- One token works across all devices
- Old tokens invalidated instantly on new generation

**Bottom Line:** 
- Only ONE token per user is valid at any time
- Users can generate new token **anytime** (free, unlimited)
- Generating new token **immediately invalidates ALL** previous tokens
- `expiresAt` is **enforced** - must generate new token before that date

