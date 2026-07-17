# User Valid Field Implementation

## Date: December 3, 2025

## Overview

Added a `valid` field to the User model to control user access at the verification level. If `valid=false`, the verification process returns `VERIFICATION_FAILURE` instead of the user's `billingId`.

---

## What Changed

### 1. User Model - Added `valid` Field

**File**: `User.java`

```java
@Data
@Document(collection = "users")
public class User {
    private String id;
    private String username;
    private String password;
    private String email;
    private String billingId;
    private Instant createdAt;
    private Instant updatedAt;
    private boolean active;
    
    // NEW: Validation flag
    private boolean valid = true;  // Default: true
}
```

**Purpose**:
- Controls whether a user can use the service
- Checked during JWT verification
- Independent from `active` field (different semantics)

### 2. UserService - Set `valid=true` on Registration

**File**: `UserService.java`

```java
public Mono<User> register(String username, String password, String email) {
    User user = new User();
    user.setUsername(username);
    user.setPassword(passwordEncoder.encode(password));
    user.setEmail(email);
    user.setBillingId(UUID.randomUUID().toString());
    user.setActive(true);
    user.setValid(true);  // NEW: User is valid by default
    
    return userRepository.save(user)...
}
```

### 3. SecretImp - Check `valid` Before Returning billingId

**File**: `SecretImp.java`

```java
@Override
public String verify(byte cryptoType, byte[] secret) {
    if (isJWT(cryptoType)) {
        return userService.validateToken(jwtToken)
            .map(user -> {
                // Check if user is valid
                if (!user.isValid()) {
                    logger.warn("User {} is not valid, returning VERIFICATION_FAILURE", 
                               user.getUsername());
                    return VERIFICATION_FAILURE;
                }
                // User is valid, return billingId
                return user.getBillingId();
            })
            .blockOptional()
            .orElse(VERIFICATION_FAILURE);
    }
    // ... hash-based crypto handling
}
```

---

## Verification Flow

### Before (Only `active` Check)
```
1. Parse JWT
2. Validate JWT signature
3. Find User by userId
4. Check user.active == true
5. Check billing quota
6. Return billingId
```

### After (With `valid` Check)
```
1. Parse JWT
2. Validate JWT signature
3. Find User by userId
4. Check user.active == true
5. Check user.valid == true  ← NEW!
   - If false → Return VERIFICATION_FAILURE
6. Check billing quota
7. Return billingId
```

---

## Semantics: `active` vs `valid`

### `active` Field
**Purpose**: General account status
**Use Cases**:
- User deactivates their account
- Admin temporarily disables account
- Account locked due to inactivity

**Behavior**:
- Checked in `UserService.validateToken()`
- Affects all authentication (web login, JWT validation)
- General-purpose flag

### `valid` Field (NEW)
**Purpose**: Service access control at verification level
**Use Cases**:
- User violates terms of service
- Payment issues (subscription expired)
- Security concerns (suspected fraud)
- Admin-initiated service suspension

**Behavior**:
- Checked in `SecretImp.verify()`
- Only affects JWT verification (proxy access)
- Specific to proxy service access
- Returns VERIFICATION_FAILURE (connection rejected at protocol level)

---

## Use Cases

### 1. Subscription Expiration
```java
// User's subscription expired
public Mono<Void> handleSubscriptionExpired(String userId) {
    return userRepository.findById(userId)
        .flatMap(user -> {
            user.setValid(false);  // Block proxy access
            return userRepository.save(user);
        })
        .then();
}

// Result: User can still login to web, but cannot use proxy
```

### 2. Terms of Service Violation
```java
// User violated ToS
public Mono<Void> suspendUserForViolation(String userId) {
    return userRepository.findById(userId)
        .flatMap(user -> {
            user.setValid(false);  // Block proxy access immediately
            return userRepository.save(user);
        })
        .then();
}

// Result: Existing connections continue, new connections rejected
```

### 3. Reinstate User
```java
// User paid or resolved issue
public Mono<Void> reinstateUser(String userId) {
    return userRepository.findById(userId)
        .flatMap(user -> {
            user.setValid(true);  // Restore proxy access
            return userRepository.save(user);
        })
        .then();
}

// Result: User can use proxy again
```

---

## Database Schema

### Users Collection
```javascript
{
  _id: "507f1f77bcf86cd799439011",
  username: "alice",
  password: "$2a$10$...",
  email: "alice@example.com",
  billingId: "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  active: true,      // General account status
  valid: true,       // Service access control (NEW)
  createdAt: ISODate("2025-01-01"),
  updatedAt: ISODate("2025-01-01")
}
```

### Migration for Existing Users
```javascript
// Add valid=true to existing users without this field
db.users.updateMany(
  { valid: { $exists: false } },
  { $set: { valid: true } }
);

// Create index (optional, for queries)
db.users.createIndex({ valid: 1 });
```

---

## API Endpoints (Future)

### Admin: Suspend User
```http
POST /api/admin/users/{userId}/suspend
Authorization: Bearer <admin-token>

Response:
{
  "success": true,
  "message": "User suspended, proxy access blocked"
}
```

### Admin: Reinstate User
```http
POST /api/admin/users/{userId}/reinstate
Authorization: Bearer <admin-token>

Response:
{
  "success": true,
  "message": "User reinstated, proxy access restored"
}
```

### User: Check Status
```http
GET /api/auth/status
Authorization: Session or JWT

Response:
{
  "active": true,
  "valid": true,
  "canUseProxy": true
}
```

---

## Behavior Comparison

| Scenario | `active` | `valid` | Web Login | Proxy Access |
|----------|----------|---------|-----------|--------------|
| Normal user | true | true | ✅ Allowed | ✅ Allowed |
| Deactivated account | false | true | ❌ Denied | ❌ Denied |
| Suspended service | true | false | ✅ Allowed | ❌ Denied |
| Both disabled | false | false | ❌ Denied | ❌ Denied |

**Key Insight**: 
- `active=false` → Cannot login at all
- `valid=false` → Can login to web, but cannot use proxy

---

## Logging

### When User is Invalid
```
WARN  SecretImp - User alice is not valid, returning VERIFICATION_FAILURE
```

### In HeaderDecoder
```
DEBUG HeaderDecoder - secret verification failed, rerouting to http
```

**Result**: Connection appears as normal HTTPS traffic (disguise maintained)

---

## Security Considerations

### 1. Immediate Effect
✅ Takes effect on next connection attempt
✅ Existing connections continue (graceful degradation)
✅ No need to invalidate JWT tokens

### 2. Granular Control
✅ Can block proxy access without affecting web access
✅ User can still login to pay bills or contact support
✅ Maintains user experience for account management

### 3. Stealth Mode
✅ Connection rerouted to HTTPS handler (disguise maintained)
✅ No error message exposed to client
✅ Appears as normal web traffic

---

## Testing

### Unit Test
```java
@Test
public void testVerifyInvalidUser() {
    User user = new User();
    user.setUsername("alice");
    user.setBillingId("f47ac10b-...");
    user.setActive(true);
    user.setValid(false);  // Invalid user
    
    when(userService.validateToken(anyString()))
        .thenReturn(Mono.just(user));
    
    String result = secret.verify(Cryptos.JWT, jwtBytes);
    
    assertEquals(Secret.VERIFICATION_FAILURE, result);
}

@Test
public void testVerifyValidUser() {
    User user = new User();
    user.setUsername("alice");
    user.setBillingId("f47ac10b-...");
    user.setActive(true);
    user.setValid(true);  // Valid user
    
    when(userService.validateToken(anyString()))
        .thenReturn(Mono.just(user));
    
    String result = secret.verify(Cryptos.JWT, jwtBytes);
    
    assertEquals("f47ac10b-...", result);
    assertNotEquals(Secret.VERIFICATION_FAILURE, result);
}
```

### Integration Test
```java
@Test
public void testSuspendUserBlocksProxy() {
    // Register and login user
    User user = userService.register("alice", "pass", "email").block();
    String jwt = userService.generateToken(user.getId(), 1440).block();
    
    // Suspend user
    user.setValid(false);
    userRepository.save(user).block();
    
    // Try to use proxy
    String billingId = secret.verify(Cryptos.JWT, jwt.getBytes());
    
    // Should return VERIFICATION_FAILURE
    assertEquals(Secret.VERIFICATION_FAILURE, billingId);
}
```

---

## Performance Impact

### Minimal Overhead
```
Before: validateToken() → map(User::getBillingId)
After:  validateToken() → map(user -> user.isValid() ? getBillingId() : FAILURE)
```

**Impact**: 
- Single boolean check (`user.isValid()`)
- No additional database queries
- Negligible performance difference (~1ns)

---

## Monitoring

### Metrics to Track
```java
// Count invalid user access attempts
Counter invalidUserAttempts = Counter.builder("user.invalid.attempts")
    .tag("username", user.getUsername())
    .register(meterRegistry);

// In SecretImp.verify()
if (!user.isValid()) {
    invalidUserAttempts.increment();
    logger.warn("User {} is not valid", user.getUsername());
    return VERIFICATION_FAILURE;
}
```

### Alerts
- Spike in invalid user attempts (possible attack)
- Same user repeatedly attempting with invalid status
- Geographic anomalies with invalid users

---

## Best Practices

### 1. Always Provide Reason
```java
public Mono<Void> suspendUser(String userId, String reason) {
    return userRepository.findById(userId)
        .flatMap(user -> {
            user.setValid(false);
            user.setUpdatedAt(Instant.now());
            // Store reason in separate SuspensionReason collection
            return suspensionReasonRepository.save(new SuspensionReason(userId, reason))
                .then(userRepository.save(user));
        })
        .then();
}
```

### 2. Notify User
```java
public Mono<Void> suspendUser(String userId, String reason) {
    return userRepository.findById(userId)
        .flatMap(user -> {
            user.setValid(false);
            return userRepository.save(user)
                .then(emailService.sendSuspensionNotification(user.getEmail(), reason));
        })
        .then();
}
```

### 3. Log All Changes
```java
public Mono<Void> setUserValid(String userId, boolean valid, String reason) {
    return userRepository.findById(userId)
        .flatMap(user -> {
            boolean oldValue = user.isValid();
            user.setValid(valid);
            
            return userRepository.save(user)
                .then(auditLogRepository.save(new AuditLog(
                    userId, 
                    "valid", 
                    oldValue, 
                    valid, 
                    reason
                )));
        })
        .then();
}
```

---

## Conclusion

The `valid` field provides:

✅ **Granular control**: Block proxy access without affecting web access
✅ **Immediate effect**: No need to invalidate JWT tokens
✅ **Stealth operation**: Maintains HTTPS disguise
✅ **User-friendly**: Can still login to resolve issues
✅ **Admin-friendly**: Simple boolean flag to control access
✅ **Minimal overhead**: Single boolean check, no performance impact

**Key Use Cases**:
- Subscription management
- ToS violation enforcement
- Security incident response
- Payment issue handling

**Status**: ✅ Implemented and compiled successfully

