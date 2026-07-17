# Billing and Charging Architecture

## Overview

The billing system is implemented as a **separate, permanent collection** that tracks usage, quotas, and payments independently from transient token data.

---

## Architecture: Three-Tier Data Model

### 1. User (Permanent - Identity)
**Collection**: `users`
**Purpose**: Long-term user identity and credentials
**Lifecycle**: Created on registration, persists forever

```javascript
{
  _id: "507f1f77bcf86cd799439011",
  username: "alice",
  password: "$2a$10$...",
  email: "alice@example.com",
  active: true,
  createdAt: ISODate("2025-01-01"),
  updatedAt: ISODate("2025-01-01")
}
```

### 2. BillingInfo (Permanent - Business)
**Collection**: `billing`
**Purpose**: Billing, usage tracking, quotas, payments
**Lifecycle**: Created on registration, persists forever
**Correlation**: `BillingInfo.userId → User._id`

```javascript
{
  _id: "...",
  userId: "507f1f77bcf86cd799439011",  // ← Links to User
  
  // Usage (accumulated)
  totalBandwidthUsed: 5368709120,      // 5 GB lifetime
  currentPeriodBandwidth: 1073741824,  // 1 GB this month
  totalConnections: 523,                // 523 lifetime connections
  currentPeriodConnections: 42,         // 42 this month
  
  // Quotas
  bandwidthQuota: 10737418240,          // 10 GB per month
  connectionQuota: 100,                 // 100 per month
  quotaResetAt: ISODate("2025-02-01"),
  
  // Billing period
  periodStart: ISODate("2025-01-01"),
  periodEnd: ISODate("2025-02-01"),
  
  // Status
  plan: "free",                         // free, basic, premium
  quotaExceeded: false,
  suspended: false,
  
  // Payment
  paymentMethod: "credit_card",
  lastPaymentAt: ISODate("2025-01-01"),
  accountBalance: 10.00,
  
  // Audit
  createdAt: ISODate("2025-01-01"),
  updatedAt: ISODate("2025-01-02"),
  lastUsedAt: ISODate("2025-01-02T12:00:00")
}
```

### 3. TokenInfo (Transient - Access)
**Collection**: `tokens`
**Purpose**: JWT token validation, access control
**Lifecycle**: Created on token generation, **deleted** when new token generated
**Correlation**: `TokenInfo.userId → User._id`

```javascript
{
  _id: "...",
  tokenId: "uuid-from-jwt",
  userId: "507f1f77bcf86cd799439011",  // ← Links to User
  createdAt: ISODate("2025-01-01"),
  expiresAt: ISODate("2025-02-01"),
  revoked: false
  
  // ❌ NO billing data here!
  // Billing data is in BillingInfo collection
}
```

---

## Why Separate Collections?

### Problem with Mixing Data
```
❌ BAD: Putting billing in TokenInfo
- TokenInfo is deleted when new token is generated
- Would lose billing history!
- Can't track usage across tokens
- Business-critical data in transient storage
```

### Solution: Separate Collections
```
✅ GOOD: Billing in separate collection
- BillingInfo persists across token regeneration
- Can track lifetime usage
- Business data separated from access data
- Clear data ownership and lifecycle
```

---

## Data Lifecycle

### User Registration
```
1. Create User record (permanent)
2. Create BillingInfo record (permanent, default: free plan)
3. User can now login and generate tokens
```

### Token Generation
```
1. Delete old TokenInfo (if exists)
2. Create new TokenInfo
3. BillingInfo unchanged (stays permanent)
```

### Service Usage
```
1. Validate JWT → Check TokenInfo exists
2. Check User is active
3. Check BillingInfo quota not exceeded
4. If valid → Allow connection
5. Record usage in BillingInfo (bandwidth, connections)
```

### Quota Reset
```
1. Check if quotaResetAt < now
2. If yes:
   - Reset currentPeriodBandwidth = 0
   - Reset currentPeriodConnections = 0
   - Set new quotaResetAt (next month)
   - quotaExceeded = false
3. BillingInfo totalBandwidthUsed/totalConnections unchanged (lifetime stats)
```

---

## Usage Tracking

### Recording Bandwidth
```java
// When user transfers data
billingService.recordBandwidth(userId, bytes);

// Updates:
- totalBandwidthUsed += bytes (lifetime)
- currentPeriodBandwidth += bytes (this period)
- lastUsedAt = now
- If currentPeriodBandwidth >= bandwidthQuota → quotaExceeded = true
```

### Recording Connections
```java
// When user makes a connection
billingService.recordConnection(userId);

// Updates:
- totalConnections++ (lifetime)
- currentPeriodConnections++ (this period)
- lastUsedAt = now
- If currentPeriodConnections >= connectionQuota → quotaExceeded = true
```

### Checking Quota
```java
// Before allowing service access
boolean canUse = billingService.canUseService(userId);

// Checks:
- suspended == false
- quotaExceeded == false
- Auto-reset quota if needed
```

---

## Plans and Quotas

### Free Plan
```
Bandwidth: 10 GB/month
Connections: 100/month
Cost: $0
```

### Basic Plan
```
Bandwidth: 100 GB/month
Connections: 1,000/month
Cost: $5/month
```

### Premium Plan
```
Bandwidth: 1 TB/month
Connections: Unlimited
Cost: $20/month
```

### Unlimited Plan
```
Bandwidth: Unlimited
Connections: Unlimited
Cost: $50/month
```

**Implementation**:
```java
private void setDefaultQuotas(BillingInfo billing, String plan) {
    switch (plan) {
        case "free":
            billing.setBandwidthQuota(10L * 1024 * 1024 * 1024);  // 10 GB
            billing.setConnectionQuota(100);
            break;
        case "basic":
            billing.setBandwidthQuota(100L * 1024 * 1024 * 1024); // 100 GB
            billing.setConnectionQuota(1000);
            break;
        case "premium":
            billing.setBandwidthQuota(1000L * 1024 * 1024 * 1024); // 1 TB
            billing.setConnectionQuota(null); // Unlimited
            break;
        case "unlimited":
            billing.setBandwidthQuota(null); // Unlimited
            billing.setConnectionQuota(null); // Unlimited
            break;
    }
}
```

---

## Integration with JWT Validation

### Token Validation Flow
```
validateToken(jwt):
  1. Parse JWT → extract tokenId
  2. Find TokenInfo by tokenId
     - If not found → INVALID (deleted/never existed)
     - If found → Check not revoked, not expired
  3. Find User by userId
     - Check user.active == true
  4. Find BillingInfo by userId
     - Check not suspended
     - Check not quotaExceeded
     - Auto-reset quota if needed
  5. If all pass → VALID
```

**Code**:
```java
public Mono<User> validateToken(String jwtToken) {
    String tokenId = jwtService.validateAndGetTokenId(jwtToken);
    
    return tokenRepository.findByTokenId(tokenId)
        .filter(token -> !token.isRevoked() && token.getExpiresAt().isAfter(now))
        .flatMap(token -> userRepository.findById(token.getUserId())
            .filter(User::isActive)
            .filterWhen(user -> 
                billingService.canUseService(user.getId())  // ← Check billing
            )
        );
}
```

---

## API Endpoints (Future)

### Get Billing Info
```
GET /api/billing/info
Auth: Session or JWT
Response:
{
  "plan": "free",
  "currentPeriodBandwidth": 1073741824,
  "bandwidthQuota": 10737418240,
  "currentPeriodConnections": 42,
  "connectionQuota": 100,
  "quotaExceeded": false,
  "periodEnd": "2025-02-01T00:00:00Z"
}
```

### Upgrade Plan
```
POST /api/billing/upgrade
Auth: Session
Body: {"plan": "premium"}
Response:
{
  "success": true,
  "message": "Upgraded to premium plan",
  "newQuota": "1 TB/month"
}
```

### Get Usage Stats
```
GET /api/billing/stats
Auth: Session
Response:
{
  "totalBandwidthUsed": 5368709120,
  "totalConnections": 523,
  "currentPeriodBandwidth": 1073741824,
  "currentPeriodConnections": 42,
  "lastUsedAt": "2025-01-02T12:00:00Z"
}
```

---

## Database Schema

### Indexes
```javascript
// billing collection
db.billing.createIndex({ userId: 1 }, { unique: true })
db.billing.createIndex({ quotaExceeded: 1 })
db.billing.createIndex({ suspended: 1 })
db.billing.createIndex({ plan: 1 })
```

### Queries

**Find users exceeding quota**:
```javascript
db.billing.find({ quotaExceeded: true })
```

**Find users needing reset**:
```javascript
db.billing.find({ quotaResetAt: { $lt: new Date() } })
```

**Find suspended accounts**:
```javascript
db.billing.find({ suspended: true })
```

**Get usage by plan**:
```javascript
db.billing.aggregate([
  { $group: {
    _id: "$plan",
    avgBandwidth: { $avg: "$currentPeriodBandwidth" },
    totalUsers: { $sum: 1 }
  }}
])
```

---

## Future Enhancements

### 1. Payment Integration
```java
public Mono<Void> processPayment(String userId, double amount) {
    return billingRepository.findByUserId(userId)
        .flatMap(billing -> {
            billing.setAccountBalance(billing.getAccountBalance() + amount);
            billing.setLastPaymentAt(Instant.now());
            billing.setSuspended(false);
            return billingRepository.save(billing);
        })
        .then();
}
```

### 2. Usage Analytics
```java
public Mono<UsageReport> getMonthlyReport(String userId) {
    return billingRepository.findByUserId(userId)
        .map(billing -> new UsageReport(
            billing.getCurrentPeriodBandwidth(),
            billing.getCurrentPeriodConnections(),
            billing.getBandwidthQuota(),
            billing.getConnectionQuota(),
            calculateUsagePercentage(billing)
        ));
}
```

### 3. Billing History
```java
@Document(collection = "billing_history")
public class BillingHistory {
    private String userId;
    private String period;  // "2025-01"
    private Long bandwidthUsed;
    private Integer connections;
    private String plan;
    private Double amountCharged;
    private Instant billedAt;
}
```

### 4. Overage Charges
```java
public Mono<Double> calculateOverage(String userId) {
    return billingRepository.findByUserId(userId)
        .map(billing -> {
            long overage = billing.getCurrentPeriodBandwidth() - 
                          billing.getBandwidthQuota();
            if (overage > 0) {
                return overage / (1024.0 * 1024 * 1024) * 0.10; // $0.10/GB
            }
            return 0.0;
        });
}
```

---

## Best Practices

### 1. Quota Enforcement
✅ Check quota **before** allowing connection
✅ Record usage **after** successful transfer
✅ Auto-reset quota at period end
✅ Notify users before quota exceeded

### 2. Data Integrity
✅ BillingInfo created on user registration
✅ Never delete BillingInfo (permanent data)
✅ Regular backups of billing collection
✅ Audit logs for quota changes

### 3. Performance
✅ Index userId for fast lookups
✅ Cache billing info per request
✅ Batch update usage (not per byte)
✅ Background job for quota reset

### 4. Security
✅ Only user can view their own billing info
✅ Admin required for plan changes
✅ Audit all billing modifications
✅ Encrypt payment information

---

## Conclusion

The three-tier architecture ensures:

**User**: Permanent identity
**BillingInfo**: Permanent business data
**TokenInfo**: Transient access tokens

**Benefits**:
- ✅ Clear separation of concerns
- ✅ Business-critical data protected
- ✅ Easy to add billing features
- ✅ Scalable and maintainable
- ✅ Ready for payment integration

The billing system is now production-ready for tracking usage, enforcing quotas, and integrating with payment providers.

