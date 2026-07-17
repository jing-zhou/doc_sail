# TokenInfo Refactoring - Fixing Major Logical Defect

**Date**: December 4, 2025

## Problem Identified

The previous implementation had a **major logical defect** in the token management system:

### Issue 1: N:1 Relationship Violation
The `TokenInfo` collection could maintain multiple records for the same `userId`, creating an **N:1 correspondence** (multiple tokens → one user). This violated our core design principle: **"Only one active token per user at any time."**

```java
// PROBLEM: TokenInfo could have multiple records per user
@Document(collection = "tokens")
public class TokenInfo {
    @Indexed
    private ObjectId userId;  // Multiple TokenInfo records can have same userId!
    
    @Indexed
    private UUID tokenId;
    // ...
}
```

### Issue 2: Token Information Storage Violation
`TokenInfo` stored token metadata in a separate collection, violating our principle of **"Never store token information in the system."** The JWT token itself wasn't stored, but the metadata persistence still created unnecessary complexity.

### Issue 3: Inefficient Invalidation
The `deleteByUserId()` approach required database deletion operations to invalidate old tokens, adding overhead.

## Solution Implemented

### Architecture Change: Remove TokenInfo Entirely

1. **Deleted `TokenInfo` model** - No separate collection for token metadata
2. **Deleted `TokenRepository`** - No separate repository needed
3. **Added `tokenId` field to `User` model** - Store only the current active token ID

### New User Model

```java
@Data
@Document(collection = "users")
public class User {
    @Id
    private ObjectId id;
    
    @Indexed(unique = true)
    private String username;
    
    private String password;
    
    private String email;
    
    @Indexed(unique = true)
    private UUID billingId;
    
    // NEW: Current active token ID (UUID from JWT)
    // Only the CURRENT token ID is stored, not the JWT itself!
    @Indexed
    private UUID tokenId;
    
    private Instant createdAt;
    private Instant updatedAt;
    private boolean active;
    private boolean valid = true;
}
```

### Lazy Invalidation Pattern

When a new token is generated, the new `tokenId` **overwrites** the old one in the `User` record. This automatically invalidates all previous tokens without requiring database deletions.

```java
// Generate new token - old tokens automatically invalidated
public Mono<String> generateToken(String userId, long expirationMinutes) {
    JwtService.TokenPair tokenPair = jwtService.generateToken(expirationMinutes);
    UUID tokenId = UUID.fromString(tokenPair.tokenId);
    
    // Update user's tokenId field (lazy invalidation of previous tokens)
    return userRepository.findById(new ObjectId(userId))
            .flatMap(user -> {
                user.setTokenId(tokenId);  // Overwrites old tokenId!
                user.setUpdatedAt(Instant.now());
                return userRepository.save(user);
            })
            .thenReturn(tokenPair.jwt);  // JWT returned once, never stored
}
```

### Simplified Validation

```java
// Validate token by looking up User by tokenId directly
public Mono<User> validateToken(String jwtToken) {
    String tokenIdStr = jwtService.validateAndGetTokenId(jwtToken);
    if (tokenIdStr == null) {
        return Mono.empty();
    }
    
    UUID tokenId = UUID.fromString(tokenIdStr);
    
    // Direct lookup - if tokenId doesn't match current user's tokenId, it's invalid
    return userRepository.findByTokenId(tokenId)
            .filter(User::isActive)
            .filter(User::isValid)
            .filterWhen(user -> billingService.canUseService(user.getId().toString()));
}
```

### Simplified Revocation

```java
// Revoke token by clearing tokenId field
public Mono<Void> revokeAllTokens(String userId) {
    return userRepository.findById(new ObjectId(userId))
            .flatMap(user -> {
                user.setTokenId(null);  // Clear tokenId = invalidate
                user.setUpdatedAt(Instant.now());
                return userRepository.save(user);
            })
            .then();
}
```

## Benefits

### 1. **Enforces 1:1 Token-User Relationship**
Only one `tokenId` can exist per user at any time. No possibility of multiple valid tokens.

### 2. **True "No Token Storage" Design**
- JWT string is **never stored**
- Only the current `tokenId` (UUID) is stored in the `User` record
- Token metadata is not persisted separately

### 3. **Lazy Invalidation**
- No database deletions required
- Simple field update invalidates old tokens
- More efficient and cleaner

### 4. **Simplified Architecture**
- One less model (`TokenInfo` removed)
- One less repository (`TokenRepository` removed)
- Direct user lookup by `tokenId`
- Cleaner validation logic

### 5. **Better Performance**
- Single database query for validation (find user by tokenId)
- No join operations needed
- No deletion overhead

## Migration Notes

### Database Changes Required

If migrating from the old system:

1. Add `tokenId` field to existing users (initially `null`)
2. Drop the `tokens` collection (if it exists)
3. Users will need to generate new tokens (old ones will be invalid)

### Code Changes

- **Removed Files:**
  - `src/main/java/com/illiad/server/model/TokenInfo.java`
  - `src/main/java/com/illiad/server/repository/TokenRepository.java`

- **Modified Files:**
  - `src/main/java/com/illiad/server/model/User.java` - Added `tokenId` field
  - `src/main/java/com/illiad/server/repository/UserRepository.java` - Added `findByTokenId()`
  - `src/main/java/com/illiad/server/service/UserService.java` - Refactored all token operations

## Security Considerations

### What This Prevents
- ✅ Multiple simultaneous valid tokens per user
- ✅ Token reuse after new token generation
- ✅ Unauthorized token persistence

### What Still Applies
- ✅ JWT expiration is still enforced (checked in `JwtService`)
- ✅ JWT signature verification is still performed
- ✅ User validation (`valid` field) is still checked
- ✅ Billing quota is still enforced

## Summary

This refactoring fixes a fundamental architectural flaw and simplifies the system significantly. The new design properly enforces the "one token per user" principle through a simple field update mechanism, eliminating the need for a separate token metadata collection entirely.

**Key Principle**: The system never stores tokens or token metadata. It only stores the **current tokenId** in the user record, which serves as a reference point for validation. When a new token is generated, it simply updates this reference, making all previous tokens invalid by design.

