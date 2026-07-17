# Billing Handler Quick Reference

## Channel Attributes

```java
// Store billingId
ctx.channel().attr(BillingHandler.BILLING_ID).set(billingId);

// Retrieve billingId
String billingId = ctx.channel().attr(BillingHandler.BILLING_ID).get();

// Connection metadata
ctx.channel().attr(BillingHandler.CONNECTION_ID).get();
ctx.channel().attr(BillingHandler.CONNECTION_START).get();
```

---

## BillingHandler Usage

```java
// Add to pipeline (done in HeaderDecoder)
if (billingId != null && bus.billingLogRepository != null) {
    pipeline.addLast(new BillingHandler(bus.billingLogRepository));
}
```

---

## Database Queries

### Get user's billing logs
```javascript
const user = db.users.findOne({ username: "alice" });
db.billing_logs.find({ billingId: user.billingId });
```

### Calculate monthly usage
```javascript
db.billing_logs.aggregate([
  {
    $match: {
      billingId: "f47ac10b-58cc...",
      timestamp: {
        $gte: ISODate("2025-01-01"),
        $lt: ISODate("2025-02-01")
      }
    }
  },
  {
    $group: {
      _id: "$billingId",
      totalBytes: { $sum: "$totalBytes" },
      connections: { $sum: 1 }
    }
  }
]);
```

### Top bandwidth users
```javascript
db.billing_logs.aggregate([
  { $group: { _id: "$billingId", total: { $sum: "$totalBytes" } } },
  { $sort: { total: -1 } },
  { $limit: 10 }
]);
```

---

## Implementation Flow

1. **Registration** → Generate `billingId` (UUID)
2. **JWT Validation** → Extract `billingId` from User
3. **HeaderDecoder** → Store in channel context
4. **BillingHandler** → Count bytes in/out
5. **Channel Close** → Generate BillingLog
6. **Save to DB** → billing_logs collection

---

## Key Classes

- **BillingLog** - Transaction log model
- **BillingLogRepository** - MongoDB access
- **BillingHandler** - Byte counter (ChannelDuplexHandler)
- **User.billingId** - Permanent correlation ID

---

## Data Model

```javascript
// User
{
  billingId: "uuid",  // Permanent, never changes
  username: "alice",
  ...
}

// BillingLog (per connection)
{
  billingId: "uuid",
  bytesIn: 1048576,
  bytesOut: 5242880,
  totalBytes: 6291456,
  timestamp: ISODate("..."),
  connectionId: "uuid",
  durationMs: 45320
}
```

---

## Thread Safety

✅ `AtomicLong` for byte counters
✅ Channel attributes are thread-safe
✅ Asynchronous DB save (non-blocking)

---

For complete details, see:
- **BILLING_HANDLER_ARCHITECTURE.md**
- **BILLING_HANDLER_IMPLEMENTATION_SUMMARY.md**

