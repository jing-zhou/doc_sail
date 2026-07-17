# JWT Architecture Refactoring Summary

## Date: December 3, 2025

## Changes Made

Refactored the JWT authentication system to follow better separation of concerns by removing `currentTokenId` from the User model.

---

## Why This Change?

### Problem with Previous Design
- `currentTokenId` was stored in the User model
- Mixed permanent user data with transient token data
- Created redundancy: both User and TokenInfo tracked the same relationship
- Made billing/charging integration more complex

### New Design Philosophy
**Separation of Concerns**:
- **User Collection**: Permanent/long-term data (username, password, email, account status)
- **TokenInfo Collection**: Transient data (tokens, expiration, usage stats, billing)
- **Correlation**: TokenInfo.userId links to User._id

---

## What Changed

### 1. User Model (Simplified)

**Before**:
```java
@Document(collection = "users")
public class User {
    private String id;
    private String username;
    private String password;
    private String email;
    private boolean active;
    private String currentTokenId;  // ❌ Removed - doesn't belong here
}
```

**After**:
```java
@Document(collection = "users")
public class User {
    private String id;
    private String username;
    private String password;
    private String email;
    private boolean active;
    // ✅ Clean - only permanent user data
}
```

### 2. TokenInfo Model (Enhanced)

**Before**:
```java
@Document(collection = "tokens")
public class TokenInfo {
    private String tokenId;
    private String userId;
    private Instant expiresAt;
    private boolean revoked;
}
```

**After**:
```java
@Document(collection = "tokens")
public class TokenInfo {
    private String tokenId;
    private String userId;
    private Instant expiresAt;
    private boolean revoked;
    
    // ✅ Future billing/charging fields
    private Long bandwidthUsed;
    private Long bandwidthLimit;
    private Instant lastUsedAt;
    private Integer connectionCount;
}
```

### 3. UserService (Cleaner Logic)

**Before** (Complex):
```java
public Mono<String> generateToken(String userId, long expirationMinutes) {
    // ... generate token ...
    
    // Save token metadata AND update user
    return tokenRepository.save(tokenInfo)
        .flatMap(saved -> userRepository.findById(userId)
            .flatMap(user -> {
                user.setCurrentTokenId(tokenPair.tokenId);  // ❌ Extra step
                return userRepository.save(user);            // ❌ Extra DB write
            }))
        .thenReturn(tokenPair.jwt);
}

public Mono<User> validateToken(String jwtToken) {
    // ... 
    return tokenRepository.findByTokenId(tokenId)
        .filter(tokenInfo -> !tokenInfo.isRevoked() && ...)
        .flatMap(tokenInfo -> userRepository.findById(tokenInfo.getUserId())
            .filter(user -> user.isActive() && 
                   tokenId.equals(user.getCurrentTokenId())));  // ❌ Extra check
}
```

**After** (Simple):
```java
public Mono<String> generateToken(String userId, long expirationMinutes) {
    // ... generate token ...
    
    // Delete old tokens, save new one
    return tokenRepository.deleteByUserId(userId)  // ✅ Simple invalidation
            .then(tokenRepository.save(tokenInfo))
            .thenReturn(tokenPair.jwt);
}

public Mono<User> validateToken(String jwtToken) {
    // ...
    return tokenRepository.findByTokenId(tokenId)
        .filter(tokenInfo -> !tokenInfo.isRevoked() && ...)
        .flatMap(tokenInfo -> userRepository.findById(tokenInfo.getUserId())
            .filter(User::isActive));  // ✅ Simple check
}
```

---

## Benefits

### 1. Clear Separation of Concerns
- **User**: Identity and credentials (permanent)
- **TokenInfo**: Access tokens and usage (transient)
- **Session**: Web authentication (temporary)

### 2. Simpler Token Invalidation
- **Before**: Update User.currentTokenId + keep old TokenInfo
- **After**: Delete old TokenInfo records (cleaner)

### 3. Better Scalability
- No need to update User record on every token generation
- TokenInfo can grow with billing data without affecting User
- Easy to add new fields for usage tracking

### 4. Future-Ready for Billing
```java
// Easy to add validation logic
public Mono<User> validateToken(String jwtToken) {
    return tokenRepository.findByTokenId(tokenId)
        .filter(tokenInfo -> {
            // Basic validation
            if (tokenInfo.isRevoked()) return false;
            if (tokenInfo.getExpiresAt().isBefore(Instant.now())) return false;
            
            // Future: Billing validation
            if (tokenInfo.getBandwidthLimit() != null && 
                tokenInfo.getBandwidthUsed() > tokenInfo.getBandwidthLimit()) {
                return false;  // Exceeded bandwidth quota
            }
            
            return true;
        })
        .flatMap(tokenInfo -> userRepository.findById(tokenInfo.getUserId())
            .filter(User::isActive));
}
```

---

## Database Schema

### Users Collection
```javascript
{
  _id: ObjectId("507f1f77bcf86cd799439011"),
  username: "alice",
  password: "$2a$10$hashed...",
  email: "alice@example.com",
  active: true,
  createdAt: ISODate("2025-01-01T00:00:00Z"),
  updatedAt: ISODate("2025-01-01T00:00:00Z")
}
```

### Tokens Collection
```javascript
{
  _id: ObjectId("..."),
  tokenId: "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  userId: "507f1f77bcf86cd799439011",  // ← Links to User
  createdAt: ISODate("2025-01-01T00:00:00Z"),
  expiresAt: ISODate("2025-02-01T00:00:00Z"),
  revoked: false,
  
  // Future billing fields
  bandwidthUsed: 1048576,              // 1 MB
  bandwidthLimit: 107374182400,        // 100 GB
  lastUsedAt: ISODate("2025-01-02T12:00:00Z"),
  connectionCount: 42
}

// Indexes:
// - tokenId (unique, for fast lookup)
// - userId (for finding user's tokens)
```

### Sessions Collection
```javascript
{
  _id: "session-uuid",
  userId: "507f1f77bcf86cd799439011",  // ← Links to User
  createdAt: ISODate("2025-01-01T00:00:00Z"),
  expiresAt: ISODate("2025-01-02T00:00:00Z"),
  active: true
}
```

---

## Token Lifecycle

### Generation
```
1. User requests new token (via web or JWT renewal)
2. Generate new tokenId (UUID)
3. Create new JWT containing tokenId
4. Delete all old TokenInfo for this userId     ← Old tokens invalidated
5. Save new TokenInfo to database
6. Return JWT to user
```

### Validation
```
1. Receive connection with JWT in header
2. Parse JWT, extract tokenId
3. Verify JWT signature
4. Look up TokenInfo by tokenId
5. If not found → INVALID (was deleted/never existed)
6. If found, check:
   - Not revoked
   - Not expired
   - (Future) Bandwidth not exceeded
7. Get User by userId
8. Check if user is active
9. If all pass → VALID
```

### Invalidation
```
Automatic:
- Generate new token → Old tokens deleted
- Token expires → Validation fails on expiresAt check
- User deactivated → Validation fails on user.active check

Manual:
- User calls /api/auth/token/revoke → All TokenInfo deleted
- Admin revokes user → user.active = false
- (Future) Bandwidth exceeded → Token validation fails
```

---

## Migration Impact

### For Existing Deployments
**Database Migration Needed**: Remove `currentTokenId` from existing User documents
```javascript
// MongoDB migration script
db.users.updateMany(
  { currentTokenId: { $exists: true } },
  { $unset: { currentTokenId: "" } }
)
```

**Behavior Changes**:
- Old tokens remain valid until they expire or user generates new token
- New token generation deletes old TokenInfo instead of updating User
- Validation no longer checks User.currentTokenId

### No Code Changes Needed For
- Client applications (JWT format unchanged)
- HTTP API endpoints (same as before)
- Authentication flow (same as before)

---

## Future Enhancements

### 1. Billing Integration (✅ NOW IMPLEMENTED!)
```java
// Track bandwidth usage
billingService.recordBandwidth(userId, bytes);

// Track connections
billingService.recordConnection(userId);

// Check limits during validation
public Mono<User> validateToken(String jwtToken) {
    return tokenRepository.findByTokenId(tokenId)
        .filter(/* token checks */)
        .flatMap(token -> userRepository.findById(token.getUserId())
            .filter(User::isActive)
            .filterWhen(user -> 
                billingService.canUseService(user.getId())  // ✅ Implemented!
            )
        );
}
```

**See**: `BILLING_ARCHITECTURE.md` for complete billing documentation.

### 2. Payment Integration (Future)
```java
// Process payments
public Mono<Void> processPayment(String userId, double amount) {
    return billingService.getBillingInfo(userId)
        .flatMap(billing -> {
            billing.setAccountBalance(billing.getAccountBalance() + amount);
            billing.setLastPaymentAt(Instant.now());
            return billingRepository.save(billing);
        })
        .then();
}
```

### 3. Usage Analytics (Ready)
```java
// Query usage stats (already available via BillingInfo)
public Mono<UsageStats> getUserStats(String userId) {
    return billingService.getBillingInfo(userId)
        .map(billing -> new UsageStats(
            billing.getTotalBandwidthUsed(),
            billing.getCurrentPeriodBandwidth(),
            billing.getTotalConnections(),
            billing.getLastUsedAt()
        ));
}
```

### 4. Multiple Active Tokens (Future)
If future requirements need multiple tokens per user:
```java
// Just don't delete old tokens
public Mono<String> generateToken(String userId, long expirationMinutes) {
    // Remove: tokenRepository.deleteByUserId(userId)
    // Just save new token without deleting old ones
    return tokenRepository.save(tokenInfo)
        .thenReturn(tokenPair.jwt);
}
```

---

## Conclusion

This refactoring achieves:
- ✅ **Cleaner architecture**: Separation of User (permanent) and TokenInfo (transient)
- ✅ **Simpler code**: Fewer database operations, clearer logic
- ✅ **Better scalability**: TokenInfo can evolve independently
- ✅ **Future-ready**: Easy to add billing/charging features
- ✅ **No functionality loss**: All features work as before

**Recommendation**: Apply this pattern to other parts of the system where transient data is mixed with permanent data.

