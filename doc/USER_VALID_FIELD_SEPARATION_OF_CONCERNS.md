# User `valid` Field - Separation of Concerns Architecture

## Overview

The `valid` field serves as a **clean separation interface** between different subsystems in the application. Various processes affect verification by changing the `valid` field value, while the verification process receives these effects by checking the field.

---

## Architecture Pattern

```
┌─────────────────────────────────────────────────────────────┐
│                   WRITE Operations                           │
│              (Other Processes Set valid)                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐      ┌──────────────┐                    │
│  │   Billing    │      │ Subscription │                    │
│  │   Service    │      │   Service    │                    │
│  └──────┬───────┘      └──────┬───────┘                    │
│         │                     │                             │
│         │ setValid(false)     │ setValid(false)            │
│         │                     │                             │
│         ↓                     ↓                             │
│  ┌──────────────────────────────────────┐                  │
│  │         User.valid (boolean)         │                  │
│  │                                       │                  │
│  │  true  = User can use proxy          │                  │
│  │  false = User blocked from proxy     │                  │
│  └──────────────────────────────────────┘                  │
│         ↑                     ↑                             │
│         │ setValid(true)      │ setValid(false)            │
│         │                     │                             │
│  ┌──────┴───────┐      ┌──────┴───────┐                    │
│  │    Admin     │      │   Security   │                    │
│  │   Service    │      │   Service    │                    │
│  └──────────────┘      └──────────────┘                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│                    READ Operation                            │
│             (Verification Checks valid)                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────┐                  │
│  │   SecretImp.verify()                 │                  │
│  │                                       │                  │
│  │   if (!user.isValid()) {             │                  │
│  │       return VERIFICATION_FAILURE;   │                  │
│  │   }                                   │                  │
│  │   return user.getBillingId();        │                  │
│  └──────────────────────────────────────┘                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Principle: Separation of Concerns

### Write Side (Other Processes)
**Responsibility**: Business logic that determines if user should have access

**Processes that WRITE to `valid`**:
1. **BillingService** - Quota enforcement
2. **SubscriptionService** - Subscription management
3. **AdminService** - Manual suspension
4. **SecurityService** - Fraud detection
5. **PaymentService** - Payment failure handling

**They don't care HOW verification works, only SET the flag**

### Read Side (Verification)
**Responsibility**: Grant/deny access based on `valid` field

**Process that READS `valid`**:
1. **SecretImp.verify()** - JWT verification

**It doesn't care WHY valid is false, only CHECKS the flag**

---

## Implementation Examples

### 1. BillingService - Quota Enforcement

**File**: `BillingService.java`

```java
@Service
public class BillingService {
    
    private final UserRepository userRepository;
    private final BillingRepository billingRepository;
    
    /**
     * Check and enforce quota limits.
     * If quota exceeded, set user.valid = false.
     */
    public Mono<Void> enforceQuota(String userId) {
        return billingRepository.findByUserId(userId)
            .flatMap(billing -> {
                if (billing.hasExceededQuota()) {
                    logger.warn("User {} exceeded quota, setting valid=false", userId);
                    
                    // WRITE to valid field
                    return userRepository.findById(userId)
                        .flatMap(user -> {
                            user.setValid(false);  // ← Other process sets valid
                            user.setUpdatedAt(Instant.now());
                            return userRepository.save(user);
                        });
                }
                return Mono.empty();
            })
            .then();
    }
    
    /**
     * Reset quota and restore access.
     */
    public Mono<Void> resetQuota(String userId) {
        return billingRepository.findByUserId(userId)
            .flatMap(billing -> {
                billing.setCurrentPeriodBandwidth(0L);
                billing.setQuotaExceeded(false);
                
                return billingRepository.save(billing)
                    .then(userRepository.findById(userId))
                    .flatMap(user -> {
                        user.setValid(true);  // ← Restore access
                        return userRepository.save(user);
                    });
            })
            .then();
    }
}
```

### 2. SubscriptionService - Subscription Management

**File**: `SubscriptionService.java`

```java
@Service
public class SubscriptionService {
    
    private final UserRepository userRepository;
    private final SubscriptionRepository subscriptionRepository;
    
    /**
     * Handle subscription expiration.
     * Set user.valid = false when subscription expires.
     */
    public Mono<Void> handleExpiration(String userId) {
        return subscriptionRepository.findByUserId(userId)
            .filter(sub -> sub.getExpiresAt().isBefore(Instant.now()))
            .flatMap(sub -> {
                logger.info("Subscription expired for user {}, setting valid=false", userId);
                
                // WRITE to valid field
                return userRepository.findById(userId)
                    .flatMap(user -> {
                        user.setValid(false);  // ← Subscription process sets valid
                        return userRepository.save(user);
                    });
            })
            .then();
    }
    
    /**
     * Renew subscription and restore access.
     */
    public Mono<Void> renewSubscription(String userId, String plan, int months) {
        return subscriptionRepository.findByUserId(userId)
            .flatMap(sub -> {
                sub.setPlan(plan);
                sub.setExpiresAt(Instant.now().plus(months, ChronoUnit.MONTHS));
                
                return subscriptionRepository.save(sub)
                    .then(userRepository.findById(userId))
                    .flatMap(user -> {
                        user.setValid(true);  // ← Restore access
                        return userRepository.save(user);
                    });
            })
            .then();
    }
}
```

### 3. AdminService - Manual Control

**File**: `AdminService.java`

```java
@Service
public class AdminService {
    
    private final UserRepository userRepository;
    private final AuditLogRepository auditLogRepository;
    
    /**
     * Admin suspends user.
     * Direct control over user.valid field.
     */
    public Mono<Void> suspendUser(String userId, String reason, String adminId) {
        return userRepository.findById(userId)
            .flatMap(user -> {
                logger.warn("Admin {} suspending user {}: {}", adminId, userId, reason);
                
                // WRITE to valid field
                user.setValid(false);  // ← Admin process sets valid
                user.setUpdatedAt(Instant.now());
                
                return userRepository.save(user)
                    .then(auditLogRepository.save(new AuditLog(
                        userId,
                        "valid",
                        true,
                        false,
                        reason,
                        adminId
                    )));
            })
            .then();
    }
    
    /**
     * Admin reinstates user.
     */
    public Mono<Void> reinstateUser(String userId, String reason, String adminId) {
        return userRepository.findById(userId)
            .flatMap(user -> {
                logger.info("Admin {} reinstating user {}: {}", adminId, userId, reason);
                
                user.setValid(true);  // ← Restore access
                user.setUpdatedAt(Instant.now());
                
                return userRepository.save(user)
                    .then(auditLogRepository.save(new AuditLog(
                        userId,
                        "valid",
                        false,
                        true,
                        reason,
                        adminId
                    )));
            })
            .then();
    }
}
```

### 4. SecurityService - Fraud Detection

**File**: `SecurityService.java`

```java
@Service
public class SecurityService {
    
    private final UserRepository userRepository;
    private final SecurityEventRepository securityEventRepository;
    
    /**
     * Detect suspicious activity and block user.
     */
    public Mono<Void> handleSuspiciousActivity(String userId, String eventType) {
        return securityEventRepository.countRecentEvents(userId, eventType)
            .flatMap(count -> {
                if (count > 10) {  // Threshold exceeded
                    logger.error("Suspicious activity detected for user {}, setting valid=false", userId);
                    
                    // WRITE to valid field
                    return userRepository.findById(userId)
                        .flatMap(user -> {
                            user.setValid(false);  // ← Security process sets valid
                            return userRepository.save(user);
                        })
                        .then(sendSecurityAlert(userId, eventType));
                }
                return Mono.empty();
            })
            .then();
    }
    
    /**
     * Clear security flag after review.
     */
    public Mono<Void> clearSecurityFlag(String userId, String reviewerId) {
        return userRepository.findById(userId)
            .flatMap(user -> {
                user.setValid(true);  // ← Restore after review
                return userRepository.save(user);
            })
            .then();
    }
    
    private Mono<Void> sendSecurityAlert(String userId, String eventType) {
        // Send alert to security team
        return Mono.empty();
    }
}
```

### 5. PaymentService - Payment Failure

**File**: `PaymentService.java`

```java
@Service
public class PaymentService {
    
    private final UserRepository userRepository;
    private final PaymentRepository paymentRepository;
    
    /**
     * Handle payment failure.
     * Grace period, then set valid=false.
     */
    public Mono<Void> handlePaymentFailure(String userId) {
        return paymentRepository.findLastPayment(userId)
            .flatMap(payment -> {
                Instant gracePeriodEnd = payment.getDueDate().plus(7, ChronoUnit.DAYS);
                
                if (Instant.now().isAfter(gracePeriodEnd)) {
                    logger.warn("Payment overdue for user {}, setting valid=false", userId);
                    
                    // WRITE to valid field
                    return userRepository.findById(userId)
                        .flatMap(user -> {
                            user.setValid(false);  // ← Payment process sets valid
                            return userRepository.save(user);
                        });
                }
                return Mono.empty();
            })
            .then();
    }
    
    /**
     * Process successful payment.
     */
    public Mono<Void> processPayment(String userId, double amount) {
        return paymentRepository.save(new Payment(userId, amount, Instant.now()))
            .then(userRepository.findById(userId))
            .flatMap(user -> {
                user.setValid(true);  // ← Restore after payment
                return userRepository.save(user);
            })
            .then();
    }
}
```

### 6. Verification - Read Only

**File**: `SecretImp.java`

```java
@Component
public class SecretImp implements Secret {
    
    /**
     * Verification process - READ ONLY.
     * Does not care WHY user.valid is false, only CHECKS it.
     */
    @Override
    public String verify(byte cryptoType, byte[] secret) {
        if (cryptoByte.toByte(Cryptos.JWT) == cryptoType) {
            String jwtToken = new String(secret, StandardCharsets.UTF_8);
            
            try {
                return userService.validateToken(jwtToken)
                    .map(user -> {
                        // READ from valid field
                        if (!user.isValid()) {  // ← Verification checks valid
                            logger.warn("User {} is not valid, returning VERIFICATION_FAILURE", 
                                       user.getUsername());
                            return VERIFICATION_FAILURE;
                        }
                        return user.getBillingId();
                    })
                    .blockOptional()
                    .orElse(VERIFICATION_FAILURE);
            } catch (Exception e) {
                logger.error("JWT validation failed", e);
                return VERIFICATION_FAILURE;
            }
        } else {
            // Hash-based verification
            MessageDigest digest = MessageDigest.getInstance(...);
            boolean verified = Arrays.equals(secret, digest.digest(...));
            return verified ? null : VERIFICATION_FAILURE;
        }
    }
}
```

**Key Point**: Verification doesn't know or care if `valid=false` because of:
- Quota exceeded
- Subscription expired
- Admin suspension
- Fraud detection
- Payment failure

It just **checks the flag and acts accordingly**.

---

## Benefits of This Separation

### 1. Decoupling
✅ Verification logic isolated from business logic
✅ Business processes don't need to know about verification implementation
✅ Can change verification without affecting business logic

### 2. Single Responsibility
✅ Each service has one job
✅ BillingService handles billing
✅ SubscriptionService handles subscriptions
✅ Verification handles access control

### 3. Easy to Extend
✅ Add new process? Just write to `valid` field
✅ No need to modify verification code
✅ No need to coordinate between services

### 4. Clear Interface
✅ Boolean field = simple interface
✅ No complex state machines
✅ Easy to understand and debug

### 5. Audit Trail
```java
// Track who changed valid and why
@Document(collection = "audit_logs")
public class AuditLog {
    private String userId;
    private String field;        // "valid"
    private Object oldValue;     // true
    private Object newValue;     // false
    private String reason;       // "Quota exceeded"
    private String changedBy;    // "BillingService" or adminId
    private Instant changedAt;
}
```

---

## Data Flow Example

### Scenario: User Exceeds Quota

```
1. User transfers 11 GB (quota = 10 GB)
   
2. BillingHandler counts bytes
   → Saves BillingLog
   
3. Scheduled job aggregates logs
   → Updates BillingInfo.currentPeriodBandwidth = 11 GB
   
4. BillingService.enforceQuota() runs
   → Detects: currentPeriodBandwidth > quota
   → Sets: user.valid = false
   → Saves: User to database
   
5. User tries to connect again
   → SecretImp.verify() called
   → Loads User from database
   → Checks: user.isValid() → false
   → Returns: VERIFICATION_FAILURE
   → Connection rerouted to HTTPS
```

**No coordination needed** - each service does its job independently!

---

## Multiple Processes Can Affect `valid`

### Scenario: Multiple Reasons to Block

```java
// User might be blocked for multiple reasons
User user = userRepository.findById(userId).block();

// Check 1: Quota exceeded?
if (billing.hasExceededQuota()) {
    user.setValid(false);
}

// Check 2: Subscription expired?
if (subscription.isExpired()) {
    user.setValid(false);
}

// Check 3: Payment overdue?
if (payment.isOverdue()) {
    user.setValid(false);
}

// Check 4: Security flag?
if (security.isFlagged()) {
    user.setValid(false);
}

// Save once
userRepository.save(user);
```

**Result**: If ANY condition triggers, `valid=false`

### Restoration Requires ALL Clear

```java
// To restore access, all conditions must be resolved
public Mono<Void> checkAndRestoreAccess(String userId) {
    return Mono.zip(
        billingService.isQuotaOk(userId),
        subscriptionService.isActive(userId),
        paymentService.isCurrentOnPayment(userId),
        securityService.isClear(userId)
    ).flatMap(tuple -> {
        boolean allOk = tuple.getT1() && tuple.getT2() && 
                       tuple.getT3() && tuple.getT4();
        
        if (allOk) {
            return userRepository.findById(userId)
                .flatMap(user -> {
                    user.setValid(true);  // All clear, restore
                    return userRepository.save(user);
                });
        }
        return Mono.empty();
    }).then();
}
```

---

## Testing

### Test: Business Process Sets Valid
```java
@Test
public void testBillingServiceSetsValid() {
    // Setup: User with quota exceeded
    BillingInfo billing = new BillingInfo();
    billing.setUserId(userId);
    billing.setCurrentPeriodBandwidth(11L * 1024 * 1024 * 1024); // 11 GB
    billing.setBandwidthQuota(10L * 1024 * 1024 * 1024);        // 10 GB
    billingRepository.save(billing).block();
    
    User user = new User();
    user.setId(userId);
    user.setValid(true);  // Initially valid
    userRepository.save(user).block();
    
    // Action: Enforce quota
    billingService.enforceQuota(userId).block();
    
    // Assert: valid = false
    User updated = userRepository.findById(userId).block();
    assertFalse(updated.isValid());
}
```

### Test: Verification Checks Valid
```java
@Test
public void testVerificationChecksValid() {
    // Setup: User with valid=false
    User user = new User();
    user.setId(userId);
    user.setBillingId("f47ac10b-...");
    user.setValid(false);  // Not valid
    userRepository.save(user).block();
    
    String jwt = generateJwt(userId);
    
    // Action: Verify
    String result = secret.verify(Cryptos.JWT, jwt.getBytes());
    
    // Assert: Returns VERIFICATION_FAILURE
    assertEquals(Secret.VERIFICATION_FAILURE, result);
}
```

---

## Monitoring

### Metrics for Each Process

```java
// BillingService
Counter quotaExceededCounter = Counter.builder("user.quota.exceeded")
    .tag("service", "billing")
    .register(meterRegistry);

// When setting valid=false
quotaExceededCounter.increment();
user.setValid(false);

// SubscriptionService
Counter subscriptionExpiredCounter = Counter.builder("user.subscription.expired")
    .tag("service", "subscription")
    .register(meterRegistry);

// AdminService
Counter adminSuspensionCounter = Counter.builder("user.admin.suspended")
    .tag("service", "admin")
    .register(meterRegistry);

// SecurityService
Counter fraudDetectedCounter = Counter.builder("user.fraud.detected")
    .tag("service", "security")
    .register(meterRegistry);
```

### Dashboard: Why Users are Invalid

```
Invalid Users by Reason:
- Quota exceeded: 42 users
- Subscription expired: 18 users
- Admin suspension: 5 users
- Fraud detection: 2 users
- Payment overdue: 13 users
```

---

## Scheduled Jobs

### Periodic Enforcement
```java
@Scheduled(cron = "0 */5 * * * *")  // Every 5 minutes
public void enforceAllPolicies() {
    // Check all users for violations
    userRepository.findAll()
        .flatMap(user -> 
            billingService.enforceQuota(user.getId())
                .then(subscriptionService.checkExpiration(user.getId()))
                .then(paymentService.checkOverdue(user.getId()))
        )
        .subscribe();
}
```

---

## Conclusion

The `valid` field is a **perfect example of separation of concerns**:

✅ **Write Side** (Business Logic):
- BillingService
- SubscriptionService
- AdminService
- SecurityService
- PaymentService

**They SET `valid` based on their domain logic**

✅ **Read Side** (Verification):
- SecretImp.verify()

**It CHECKS `valid` without caring why**

This creates:
- **Loose coupling** between subsystems
- **Clear responsibilities** for each service
- **Easy extension** for new business rules
- **Simple interface** (boolean field)
- **Auditability** through change logs

**The architecture is clean, maintainable, and scalable!** 🎯

