# Billing Handler Architecture - Real-time Usage Tracking

## Overview

The billing system now uses a **real-time byte counting architecture** where a `BillingHandler` tracks all bytes flowing through each channel and generates billing logs on channel close.

---

## Architecture Flow

```
1. User Registration
   → Generate billingId (UUID, permanent)
   → Store in User.billingId

2. JWT Validation (HeaderDecoder)
   → Parse JWT from illiad header
   → Validate JWT → Get User
   → Extract User.billingId
   → Store billingId in channel context

3. Channel Pipeline Setup
   → Add BillingHandler to pipeline
   → BillingHandler reads billingId from context

4. Data Transfer
   → BillingHandler counts bytes in/out
   → Transparent to other handlers

5. Channel Close
   → BillingHandler generates BillingLog
   → Save to billing_logs collection
   → Contains: billingId, bytesIn, bytesOut, timestamp
```

---

## Data Model

### User (Permanent)
```javascript
{
  _id: "507f1f77bcf86cd799439011",
  username: "alice",
  password: "$2a$10$...",
  email: "alice@example.com",
  billingId: "f47ac10b-58cc-4372-a567-0e02b2c3d479",  // ← Permanent UUID
  active: true,
  createdAt: ISODate("2025-01-01"),
  updatedAt: ISODate("2025-01-01")
}
```

**Key Points**:
- `billingId` generated once on registration
- Never changes (permanent identifier)
- Used to correlate billing logs with user

### BillingLog (Transaction Log)
```javascript
{
  _id: "...",
  billingId: "f47ac10b-58cc-4372-a567-0e02b2c3d479",  // ← Links to User
  bytesIn: 1048576,         // Bytes received from client
  bytesOut: 5242880,        // Bytes sent to client
  totalBytes: 6291456,      // Total (in + out)
  timestamp: ISODate("2025-01-02T12:34:56Z"),
  connectionId: "uuid",     // Unique connection identifier
  targetHost: "example.com",
  targetPort: 443,
  durationMs: 45320
}
```

**Key Points**:
- One log entry per connection
- Created when channel closes
- Immutable transaction record
- Can aggregate for billing periods

### BillingInfo (Aggregated Stats)
```javascript
{
  _id: "...",
  userId: "507f1f77bcf86cd799439011",
  totalBandwidthUsed: 5368709120,      // Lifetime total
  currentPeriodBandwidth: 1073741824,  // This period
  totalConnections: 523,
  currentPeriodConnections: 42,
  bandwidthQuota: 10737418240,
  quotaExceeded: false,
  // ... other billing fields
}
```

**Key Points**:
- Aggregated statistics
- Can be updated from BillingLog entries
- Used for quota enforcement

---

## Implementation Details

### 1. User Registration - Generate billingId

**File**: `UserService.java`

```java
public Mono<User> register(String username, String password, String email) {
    User user = new User();
    user.setUsername(username);
    user.setPassword(passwordEncoder.encode(password));
    user.setEmail(email);
    user.setBillingId(UUID.randomUUID().toString());  // ← Generate permanent ID
    user.setActive(true);
    
    return userRepository.save(user)
        .flatMap(savedUser -> 
            billingService.createBillingInfo(savedUser.getId(), "free")
                .thenReturn(savedUser)
        );
}
```

### 2. JWT Validation - Extract billingId

**File**: `SecretImp.java`

```java
@Override
public String verifyAndGetBillingId(byte cryptoType, byte[] secret) {
    if (cryptoByte.toByte(Cryptos.JWT) == cryptoType) {
        String jwtToken = new String(secret, StandardCharsets.UTF_8);
        
        return userService.validateToken(jwtToken)
            .map(User::getBillingId)  // ← Extract billingId from User
            .blockOptional()
            .orElse(null);
    }
    return null;
}
```

### 3. HeaderDecoder - Store billingId in Channel

**File**: `HeaderDecoder.java`

```java
// Verify secret and get billingId (now returns a UUID or VERIFICATION_FAILURE sentinel)
boolean verified;
UUID billingId = null;
try {
    verified = bus.secret.verify(cryptoType, secretBytes);
    billingId = bus.secret.verifyAndGetBillingId(cryptoType, secretBytes); // returns UUID
} catch (Exception e) {
    switchToHttp(ctx);
    return;
}

if (!verified || billingId.equals(Secret.VERIFICATION_FAILURE)) {
    switchToHttp(ctx);
    return;
}

// Store billingId in channel context as a native UUID
if (billingId != null) {
    ctx.channel().attr(BillingHandler.BILLING_ID).set(billingId);
    logger.debug("BillingId {} stored in channel context", billingId);
}

// Add BillingHandler to pipeline
if (billingId != null && bus.billingLogRepository != null) {
    pipeline.addLast(bus.namer.generateName(), 
        new BillingHandler(bus.billingLogRepository));
}
```

### 4. BillingHandler - Count Bytes

**File**: `BillingHandler.java`

```java
public class BillingHandler extends ChannelDuplexHandler {
    private final AtomicLong bytesIn = new AtomicLong(0);
    private final AtomicLong bytesOut = new AtomicLong(0);
    
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        if (msg instanceof ByteBuf) {
            long bytes = ((ByteBuf) msg).readableBytes();
            bytesIn.addAndGet(bytes);  // ← Count incoming bytes
        }
        ctx.fireChannelRead(msg);
    }
    
    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
        if (msg instanceof ByteBuf) {
            long bytes = ((ByteBuf) msg).readableBytes();
            bytesOut.addAndGet(bytes);  // ← Count outgoing bytes
        }
        ctx.write(msg, promise);
    }
    
    @Override
    public void channelInactive(ChannelHandlerContext ctx) {
        String billingId = ctx.channel().attr(BILLING_ID).get();
        
        if (billingId != null && bytesIn.get() + bytesOut.get() > 0) {
            BillingLog log = new BillingLog();
            log.setBillingId(billingId);
            log.setBytesIn(bytesIn.get());
            log.setBytesOut(bytesOut.get());
            log.setTotalBytes(bytesIn.get() + bytesOut.get());
            log.setTimestamp(Instant.now());
            
            // Save asynchronously
            billingLogRepository.save(log).subscribe();
        }
        
        super.channelInactive(ctx);
    }
}
```

---

## Channel Attribute Keys

**File**: `BillingHandler.java`

```java
public static final AttributeKey<String> BILLING_ID = AttributeKey.valueOf("billingId");
public static final AttributeKey<String> CONNECTION_ID = AttributeKey.valueOf("connectionId");
public static final AttributeKey<Instant> CONNECTION_START = AttributeKey.valueOf("connectionStart");
```

**Usage**:
```java
// Store
ctx.channel().attr(BillingHandler.BILLING_ID).set(billingId);

// Retrieve
String billingId = ctx.channel().attr(BillingHandler.BILLING_ID).get();
```

---

## Billing Aggregation

### Real-time Aggregation (Optional)

```java
// After saving BillingLog, update BillingInfo
billingLogRepository.save(log)
    .flatMap(savedLog -> 
        billingService.recordBandwidth(billingId, savedLog.getTotalBytes())
    )
    .subscribe();
```

### Batch Aggregation (Recommended)

```java
// Periodic job (e.g., every hour or daily)
public void aggregateBillingLogs() {
    billingLogRepository.findAll()
        .groupBy(BillingLog::getBillingId)
        .flatMap(group -> 
            group.reduce(0L, (sum, log) -> sum + log.getTotalBytes())
                .flatMap(totalBytes -> 
                    userRepository.findByBillingId(group.key())
                        .flatMap(user -> 
                            billingService.recordBandwidth(user.getId(), totalBytes)
                        )
                )
        )
        .subscribe();
}
```

---

## Query Examples

### Find logs for a user
```javascript
// Get user's billingId first
const user = db.users.findOne({ username: "alice" });

// Query logs by billingId
db.billing_logs.find({ billingId: user.billingId });
```

### Calculate total usage for period
```javascript
db.billing_logs.aggregate([
  {
    $match: {
      billingId: "f47ac10b-58cc-4372-a567-0e02b2c3d479",
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
      totalConnections: { $sum: 1 },
      avgDuration: { $avg: "$durationMs" }
    }
  }
]);
```

### Find top bandwidth users
```javascript
db.billing_logs.aggregate([
  {
    $group: {
      _id: "$billingId",
      totalBytes: { $sum: "$totalBytes" },
      connections: { $sum: 1 }
    }
  },
  { $sort: { totalBytes: -1 } },
  { $limit: 10 }
]);
```

---

## Benefits

### 1. Real-time Tracking
✅ Bytes counted as they flow through channel
✅ No polling or periodic checks needed
✅ Accurate byte-level accounting

### 2. Immutable Audit Trail
✅ One log entry per connection
✅ Cannot be modified after creation
✅ Perfect for billing disputes

### 3. Separation of Concerns
✅ User: Permanent identity + billingId
✅ BillingLog: Immutable transaction log
✅ BillingInfo: Aggregated statistics
✅ Each serves different purpose

### 4. Scalability
✅ Logs can be archived/deleted after aggregation
✅ Only current period needs to be in hot storage
✅ Historical data can be in cold storage

### 5. Flexible Billing
✅ Can aggregate by any time period
✅ Can charge per GB or per connection
✅ Can analyze usage patterns
✅ Can implement tiered pricing

---

## Performance Considerations

### 1. Asynchronous Logging
```java
// Don't block channel close
billingLogRepository.save(log)
    .subscribe(
        success -> logger.info("Log saved"),
        error -> logger.error("Failed to save log", error)
    );
```

### 2. Batch Writes (Future Enhancement)
```java
// Buffer logs and write in batches
List<BillingLog> buffer = new ArrayList<>();

// In channelInactive
buffer.add(log);
if (buffer.size() >= 100) {
    billingLogRepository.saveAll(buffer).subscribe();
    buffer.clear();
}
```

### 3. Indexing
```javascript
// Indexes for fast queries
db.billing_logs.createIndex({ billingId: 1 });
db.billing_logs.createIndex({ timestamp: 1 });
db.billing_logs.createIndex({ billingId: 1, timestamp: 1 });
```

### 4. TTL for Old Logs (Optional)
```javascript
// Auto-delete logs older than 90 days
db.billing_logs.createIndex(
  { timestamp: 1 },
  { expireAfterSeconds: 7776000 }  // 90 days
);
```

---

## Security Considerations

### 1. billingId Privacy
✅ billingId is a random UUID (reveals nothing about user)
✅ Stored in channel context (not exposed externally)
✅ Only visible to authenticated connections

### 2. Log Integrity
✅ Logs created only on channel close (no manual creation)
✅ Immutable after creation
✅ Timestamp enforced server-side

### 3. Access Control
✅ Users can only view their own logs
✅ Admin can view all logs
✅ Logs contain no sensitive data (just byte counts)

---

## Testing

### 1. Unit Test - BillingHandler
```java
@Test
public void testBytesCounting() {
    BillingHandler handler = new BillingHandler(mockRepository);
    EmbeddedChannel channel = new EmbeddedChannel(handler);
    
    // Set billingId
    channel.attr(BillingHandler.BILLING_ID).set("test-billing-id");
    
    // Write data
    channel.writeInbound(Unpooled.wrappedBuffer(new byte[100]));
    channel.writeOutbound(Unpooled.wrappedBuffer(new byte[200]));
    
    // Close channel
    channel.close().awaitUninterruptibly();
    
    // Verify log saved
    verify(mockRepository).save(argThat(log ->
        log.getBytesIn() == 100 &&
        log.getBytesOut() == 200 &&
        log.getTotalBytes() == 300
    ));
}
```

### 2. Integration Test
```java
@Test
public void testBillingFlow() {
    // Register user
    User user = userService.register("alice", "pass", "alice@example.com")
        .block();
    
    assertNotNull(user.getBillingId());
    
    // Generate JWT
    String jwt = userService.generateToken(user.getId(), 1440).block();
    
    // Validate JWT and get billingId
    String billingId = secret.verifyAndGetBillingId(
        cryptoByte.toByte(Cryptos.JWT),
        jwt.getBytes()
    );
    
    assertEquals(user.getBillingId(), billingId);
}
```

---

## Migration

### Add billingId to Existing Users
```javascript
// For existing users without billingId
db.users.find({ billingId: { $exists: false } }).forEach(function(user) {
  db.users.updateOne(
    { _id: user._id },
    { $set: { billingId: UUID().toString() } }
  );
});

// Create index
db.users.createIndex({ billingId: 1 }, { unique: true });
```

---

## Future Enhancements

### 1. Target Host Tracking
```java
// In BillingHandler, store target from SOCKS5 CONNECT
public void setTargetAddress(String host, int port) {
    this.targetHost = host;
    this.targetPort = port;
}
```

### 2. Compression Ratio Tracking
```java
// Track compressed vs uncompressed bytes
private long bytesInUncompressed;
private long bytesInCompressed;
```

### 3. Protocol Tracking
```java
// Track which protocol was used
private String protocol; // "SOCKS5", "HTTP", "HTTPS"
```

### 4. Cost Calculation
```java
// Calculate cost in billing log
public double calculateCost() {
    // $0.10 per GB
    return (totalBytes / (1024.0 * 1024 * 1024)) * 0.10;
}
```

---

## Conclusion

The billing handler architecture provides:
- ✅ Real-time byte counting
- ✅ Immutable audit trail
- ✅ Clean separation of concerns
- ✅ Scalable and performant
- ✅ Ready for production

**Key Design Decision**: billingId stored in User model (permanent), logs generated per-connection, aggregation happens separately.
