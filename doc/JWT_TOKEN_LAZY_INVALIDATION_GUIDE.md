# JWT Token Lazy Invalidation - Quick Reference Guide

**Date**: December 4, 2025  
**Status**: ✅ Active Implementation

## Architecture Overview

The system implements **lazy token invalidation** by storing only the current `tokenId` in the `User` model. This ensures **one active token per user** at all times.

## Key Principle

**NO TOKEN STORAGE**: The JWT token string is NEVER stored. Only the current `tokenId` (UUID) is stored in the `User` record.

## Data Model

### User Model
```java
@Document(collection = "users")
public class User {
    @Id
    private ObjectId id;
    
    // Immutable - set once during registration, never changes
    @Indexed(unique = true)
    private String username;
    
    // Immutable - generated once during registration, never changes
    @Indexed(unique = true)
    private UUID billingId;
    
    // Current active token ID (lazy invalidation)
    @Indexed
    private UUID tokenId;  // Only current token ID!
    
    // Last update timestamp
    private Instant updatedAt;
    
    // Validation flag
    private boolean valid;
}
```

## User Flow

### 1. User Registration
```java
User user = new User();
user.setUsername("alice");  // Immutable - never changes
user.setPassword(bcrypt.encode("password"));
user.setBillingId(UUID.randomUUID());  // Immutable - permanent billing ID
user.setTokenId(null);  // No token yet
user.setUpdatedAt(Instant.now());
user.setValid(true);  // Valid by default
userRepository.save(user);
```

### 2. User Login & Token Generation
```java
// User logs in with username/password
User user = authenticate(username, password);

// Generate JWT token
String jwt = generateToken(user.getId(), 43200);  // 30 days

// This updates user.tokenId, invalidating ALL previous tokens!
```

**What happens:**
1. New `tokenId` is generated (UUID)
2. JWT is created with this `tokenId` embedded
3. `user.tokenId` is updated to the new value (overwrites old)
4. JWT is returned to user (ONE TIME ONLY, never stored)
5. All previous tokens are now invalid (their tokenId won't match)

### 3. User Connects with JWT (Verification)
```java
// User sends JWT in Illiad header
// Extract tokenId from JWT
UUID tokenId = jwtService.validateAndGetTokenId(jwt);

// Find user by tokenId
User user = userRepository.findByTokenId(tokenId);

// If found and user.valid == true, return billingId
// If not found, return VERIFICATION_FAILURE
```

**Validation checks:**
1. JWT signature valid?
2. JWT not expired?
3. User exists with this `tokenId`?
4. User is valid?
5. Billing quota not exceeded?

### 4. User Generates New Token (Auto-Invalidation)
```java
// User generates new token (either via login or "refresh token" endpoint)
String newJwt = generateToken(userId, 43200);

// This automatically invalidates ALL previous tokens!
// Because user.tokenId is now updated to new value
```

**What happens:**
1. Old token: `user.tokenId = "uuid-123"` 
2. Generate new token: `newTokenId = "uuid-456"`
3. Update: `user.setTokenId("uuid-456")`
4. Old token with `tokenId="uuid-123"` is now invalid!
   - Query `findByTokenId("uuid-123")` returns empty
   - User cannot connect with old token

### 5. Token Revocation (Manual)
```java
// Admin or user revokes all tokens
revokeAllTokens(userId);

// This sets user.tokenId = null
// All tokens become invalid immediately
```

## Implementation Details

### UserService Methods

#### Generate Token
```java
public Mono<String> generateToken(String userId, long expirationMinutes) {
    JwtService.TokenPair tokenPair = jwtService.generateToken(expirationMinutes);
    UUID tokenId = UUID.fromString(tokenPair.tokenId);
    
    return userRepository.findById(new ObjectId(userId))
        .flatMap(user -> {
            user.setTokenId(tokenId);  // LAZY INVALIDATION
            user.setUpdatedAt(Instant.now());
            return userRepository.save(user);
        })
        .thenReturn(tokenPair.jwt);  // Return JWT once
}
```

#### Validate Token
```java
public Mono<User> validateToken(String jwtToken) {
    String tokenIdStr = jwtService.validateAndGetTokenId(jwtToken);
    if (tokenIdStr == null) return Mono.empty();
    
    UUID tokenId = UUID.fromString(tokenIdStr);
    
    return userRepository.findByTokenId(tokenId)
        .filter(User::isValid)
        .filterWhen(user -> billingService.canUseService(user.getId().toString()));
}
```

#### Revoke Tokens
```java
public Mono<Void> revokeAllTokens(String userId) {
    return userRepository.findById(new ObjectId(userId))
        .flatMap(user -> {
            user.setTokenId(null);  // Invalidate
            user.setUpdatedAt(Instant.now());
            return userRepository.save(user);
        })
        .then();
}
```

## Security Features

### ✅ Enforced
- **One token per user**: User can only have ONE valid token at any time
- **Auto-invalidation**: New token automatically invalidates all old ones
- **No token storage**: JWT is never stored in database
- **Lazy invalidation**: No deletion operations needed
- **Fast validation**: Single query to find user by tokenId

### ✅ Maintained
- JWT signature verification (in JwtService)
- JWT expiration enforcement (in JwtService)
- User validity check (`user.valid` field)
- Billing quota enforcement (in BillingService)

## Why This Works

### 1. **Simple State Management**
Only one field (`user.tokenId`) tracks the current valid token.

### 2. **Automatic Invalidation**
Updating `user.tokenId` automatically makes all previous tokens invalid because:
- Old token has `tokenId = "old-uuid"`
- Database query: `findByTokenId("old-uuid")` returns **empty**
- Validation fails → connection rejected

### 3. **No Token Leakage**
- JWT is returned **once** to user
- JWT is **never** stored in database
- Only `tokenId` (UUID) is stored
- Even if database is compromised, attacker doesn't have JWTs

### 4. **Clean Revocation**
- Set `user.tokenId = null` → all tokens invalid
- No need to track token blacklist
- No need to delete token records

## Example Scenario

```
Day 1:
- Alice logs in → JWT1 with tokenId="uuid-111"
- user.tokenId = "uuid-111"
- Alice can connect ✅

Day 5:
- Alice generates new token → JWT2 with tokenId="uuid-222"
- user.tokenId = "uuid-222" (updated!)
- Alice can connect with JWT2 ✅
- Alice CANNOT connect with JWT1 ❌ (findByTokenId("uuid-111") returns empty)

Day 10:
- Admin revokes Alice's tokens
- user.tokenId = null
- Alice CANNOT connect with JWT2 ❌
- Alice must log in again to get new token
```

## Files Involved

### Models
- `User.java` - Contains `tokenId` field

### Repositories
- `UserRepository.java` - Contains `findByTokenId(UUID)` method

### Services
- `UserService.java` - Token generation, validation, revocation
- `JwtService.java` - JWT creation and parsing
- `BillingService.java` - Quota enforcement

### Security
- `SecretImp.java` - Calls `userService.validateToken()` for JWT verification
- `HeaderDecoder.java` - Extracts JWT from Illiad header, calls `verify()`

## Migration from Old System

If migrating from `TokenInfo` approach:

1. Drop `tokens` collection (if exists)
2. Add `tokenId` field to users (initially `null`)
3. All users must generate new tokens
4. Old tokens will be invalid automatically

## Summary

This implementation achieves:
- ✅ True "one token per user" enforcement
- ✅ No token storage (only current tokenId)
- ✅ Lazy invalidation (field update, no deletions)
- ✅ Simple, maintainable, efficient
- ✅ Secure by design

**Key Insight**: By storing only the current `tokenId` in the user record, we eliminate the possibility of multiple valid tokens while maintaining a clean, simple validation mechanism.

