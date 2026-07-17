# Billing Quick Reference

## Collections

### Permanent
- **users** - Identity (username, password, email)
- **billing** - Usage, quotas, payments ✨

### Transient
- **tokens** - JWT validation (deleted on regeneration)
- **sessions** - Web auth (24h expiry)

---

## BillingService API

```java
// Get billing info (create if not exists)
Mono<BillingInfo> getBillingInfo(String userId)

// Record usage
Mono<Void> recordBandwidth(String userId, long bytes)
Mono<Void> recordConnection(String userId)

// Check quota
Mono<Boolean> canUseService(String userId)

// Manage plan
Mono<BillingInfo> updatePlan(String userId, String plan)

// Account management
Mono<Void> suspendAccount(String userId)
Mono<Void> unsuspendAccount(String userId)
```

---

## Plans

| Plan | Bandwidth | Connections | Cost |
|------|-----------|-------------|------|
| free | 10 GB | 100 | $0 |
| basic | 100 GB | 1,000 | $5 |
| premium | 1 TB | Unlimited | $20 |
| unlimited | Unlimited | Unlimited | $50 |

---

## Usage

### User Registration
```java
userService.register("alice", "password", "alice@example.com")
// Auto-creates BillingInfo with free plan
```

### Token Validation
```java
userService.validateToken(jwt)
// Automatically checks billing quota
```

### Record Usage
```java
billingService.recordBandwidth(userId, 1048576)  // 1 MB
billingService.recordConnection(userId)
```

### Check Quota
```java
boolean canUse = billingService.canUseService(userId)
// false if quota exceeded or suspended
```

### Upgrade Plan
```java
billingService.updatePlan(userId, "premium")
// Updates quotas automatically
```

---

## BillingInfo Fields

```java
// Usage tracking
totalBandwidthUsed       // Lifetime total
currentPeriodBandwidth   // This period
totalConnections         // Lifetime total
currentPeriodConnections // This period

// Quotas
bandwidthQuota           // Bytes per period
connectionQuota          // Count per period
quotaResetAt            // When quota resets

// Status
plan                    // "free", "basic", "premium", "unlimited"
quotaExceeded          // true if over quota
suspended              // true if suspended

// Payment
accountBalance         // Prepaid balance
lastPaymentAt         // Last payment date

// Audit
createdAt, updatedAt, lastUsedAt
```

---

## Auto-Reset

Quota automatically resets when:
```java
if (billing.getQuotaResetAt().isBefore(Instant.now())) {
    // Reset currentPeriodBandwidth = 0
    // Reset currentPeriodConnections = 0
    // Set new quotaResetAt (next month)
    // quotaExceeded = false
}
```

---

## Integration Points

### UserService
```java
// On registration
billingService.createBillingInfo(userId, "free")

// On token validation
.filterWhen(user -> billingService.canUseService(user.getId()))
```

### Connection Handler (Future)
```java
// Before connection
if (!billingService.canUseService(userId)) {
    return; // Reject
}

// After data transfer
billingService.recordBandwidth(userId, bytesTransferred);
billingService.recordConnection(userId);
```

---

## Database Queries

```javascript
// Find user's billing
db.billing.findOne({ userId: "507f..." })

// Find users over quota
db.billing.find({ quotaExceeded: true })

// Find suspended accounts
db.billing.find({ suspended: true })

// Usage by plan
db.billing.aggregate([
  { $group: {
    _id: "$plan",
    avgBandwidth: { $avg: "$currentPeriodBandwidth" },
    totalUsers: { $sum: 1 }
  }}
])
```

---

## Key Points

✅ **Permanent**: BillingInfo never deleted
✅ **Separate**: Independent from TokenInfo
✅ **Auto-reset**: Monthly quota refresh
✅ **Integrated**: Checked in token validation
✅ **Scalable**: Ready for payment integration

---

For complete details, see:
- **BILLING_ARCHITECTURE.md** - Full architecture guide
- **BILLING_IMPLEMENTATION_SUMMARY.md** - Implementation details

