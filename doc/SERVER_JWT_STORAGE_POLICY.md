# Server JWT Storage Policy - Final Clarification

## ✅ Critical Principle: Server NEVER Stores JWT Tokens

### What This Means

When a user generates a JWT token:

1. ✅ **JWT is created** - Server generates: `eyJhbGciOiJIUzI1NiJ9...`
2. ✅ **Metadata is extracted** - Server extracts: `tokenId`, `expiresAt`
3. ✅ **Metadata is saved to database** - Only these fields stored
4. ✅ **JWT is returned to user** - One time only
5. ✅ **JWT is discarded** - Removed from server memory
6. ❌ **JWT is NEVER stored** - Not in database, not in logs, nowhere

---

## What IS Stored in Database

### TokenInfo Collection (MongoDB)

```javascript
{
  "_id": ObjectId("674abc123def456789012345"),
  "userId": "507f1f77bcf86cd799439011",           // Which user owns this token
  "tokenId": "a1b2c3d4-e5f6-7890-abcd-ef1234...",  // UUID from inside JWT
  "createdAt": ISODate("2024-12-03T18:00:00Z"),
  "expiresAt": ISODate("2025-01-02T18:00:00Z"),
  "revoked": false
}
```

**Note:** The JWT string `eyJhbGciOiJIUzI1NiJ9...` is **NOT** in this document!

### User Collection (MongoDB)

```javascript
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "username": "alice",
  "password": "$2a$10$...",  // BCrypt hash
  "email": "alice@example.com",
  "currentTokenId": "a1b2c3d4-e5f6-7890-abcd-ef1234...",
  "createdAt": ISODate("2024-12-01T10:00:00Z"),
  "updatedAt": ISODate("2024-12-03T18:00:00Z"),
  "active": true
}
```

**Note:** The JWT string is **NOT** stored here either!

---

## What IS NOT Stored

❌ **Never stored anywhere on server:**
- The actual JWT token string (`eyJhbGciOiJIUzI1NiJ9...`)
- JWT tokens in logs
- JWT tokens in backups
- JWT tokens in cache
- JWT tokens in temporary files
- JWT tokens in session storage

---

## Validation Process (Without Storing JWT)

### When User Connects

```
1. User sends: JWT token "eyJhbGciOiJIUzI1NiJ9.eyJpZCI6ImExYjJjM2Q0..."

2. Server decodes JWT (in memory):
   {
     "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
     "expiresAt": 1735862400000
   }

3. Server validates JWT signature (in memory)

4. Server checks expiresAt > now

5. Server queries database:
   db.tokens.findOne({tokenId: "a1b2c3d4-..."})
   
6. Database returns metadata (if found)

7. Server verifies:
   - Token not revoked
   - expiresAt not passed
   - Get userId
   
8. Server queries database:
   db.users.findOne({_id: userId})
   
9. Server verifies:
   - User is active
   - currentTokenId matches

10. If all checks pass: ALLOW connection
    If any check fails: REJECT (reroute to HTTPS)
```

**At no point does the server look up or retrieve the JWT from storage** - because it's not stored!

---

## Security Benefits

### 1. Database Breach Protection
```
❌ Attacker compromises MongoDB
✅ Attacker CANNOT steal JWT tokens
✅ Because JWTs are not in the database!
✅ Attacker only gets metadata (tokenId, userId, expiresAt)
✅ Metadata is useless without the actual JWT
```

### 2. Log File Safety
```
✅ Even if logs are leaked
✅ No JWT tokens exposed
✅ Only tokenIds (UUIDs) appear in logs
✅ UUIDs reveal nothing about users
```

### 3. Backup Security
```
✅ Database backups don't contain JWTs
✅ Safe to store backups in cloud
✅ Safe to send backups to third parties for recovery
```

### 4. Admin Access Control
```
✅ Even server admins cannot see user JWTs
✅ Admins can only see metadata
✅ Prevents insider threats
✅ Prevents accidental exposure
```

---

## User Responsibility

Since the server doesn't store JWTs, users must:

### ✅ DO:
- Save token immediately after generation
- Store in password manager (recommended)
- Save to encrypted file
- Backup token securely
- Copy before closing browser

### ❌ DON'T:
- Expect server to retrieve lost tokens
- Assume tokens are backed up on server
- Think admins can help recover lost tokens
- Close browser without saving token

### 🔄 If Token is Lost:
1. Generate a new token (free, unlimited)
2. Old token becomes invalid automatically
3. Update client with new token
4. No penalty, no fees, no restrictions

---

## Implementation Details

### Code Comments Added

**UserService.java:**
```java
/**
 * Generate a new JWT token for a user. This invalidates any previous token.
 * 
 * IMPORTANT: The JWT token string is NEVER stored in the database!
 * Only the metadata (tokenId, userId, expiresAt) is saved.
 * The JWT is returned once to the user and then discarded.
 * 
 * @return JWT token string (plain text, returned once, never stored)
 */
public Mono<String> generateToken(String userId, long expirationMinutes) {
    // ...
    .thenReturn(tokenPair.jwt);  // JWT returned to user (one time only)
}
```

**TokenInfo.java:**
```java
/**
 * Token metadata stored in database.
 * 
 * IMPORTANT: This class stores only METADATA about the token,
 * NOT the actual JWT token string itself!
 * 
 * The JWT token (eyJhbGciOiJIUzI1NiJ9...) is NEVER stored in the database.
 * It's generated once, returned to the user, and then discarded.
 */
@Document(collection = "tokens")
public class TokenInfo {
    private String tokenId; // UUID from inside the JWT (not the JWT itself!)
    // NOTE: The actual JWT token string is NOT stored here!
}
```

---

## Documentation Updated

All documentation has been updated to reflect this policy:

1. ✅ **JWT_PRIVACY_EXPLAINED.md**
   - Added "One-Time Token Display Policy" section
   - Updated database correlation explanation
   - Added Q&A about server not storing JWTs

2. ✅ **JWT_TOKEN_USER_GUIDE.md**
   - Updated "Token Behavior" section
   - Added "What the Server Stores" clarification

3. ✅ **JWT_AUTHENTICATION.md**
   - Updated "One-Time Download" section
   - Added "Why server doesn't store JWTs" explanation

4. ✅ **JWT_QUICK_REFERENCE.md**
   - Added "Server Storage" warning
   - Updated "If Lost" section

5. ✅ **SERVER_JWT_STORAGE_POLICY.md** (this document)
   - Complete explanation of storage policy

---

## Comparison: What's Stored vs What's Not

### ✅ STORED (Database)
| Field | Type | Example | Purpose |
|-------|------|---------|---------|
| tokenId | UUID | `a1b2c3d4-...` | Correlate to user |
| userId | ObjectId | `507f1f77...` | Which user owns token |
| expiresAt | Timestamp | `1735862400000` | When token expires |
| createdAt | Timestamp | `1733270400000` | When token was created |
| revoked | Boolean | `false` | Is token revoked? |
| username | String | `alice` | User's login name |
| password | Hash | `$2a$10$...` | BCrypt hash |
| email | String | `alice@example.com` | User's email |

### ❌ NOT STORED (Nowhere)
| Data | Why Not Stored | What Instead |
|------|----------------|--------------|
| JWT token | Security | Token returned once to user |
| JWT in logs | Privacy | Only tokenId logged |
| JWT in backups | Safety | Only metadata backed up |
| JWT in cache | Volatility | Generated fresh each time |

---

## Lifecycle Diagram

```
┌─────────────────────────────────────────────────────────────┐
│ 1. User clicks "Generate Token"                            │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. Server generates JWT in memory                          │
│    jwt = "eyJhbGciOiJIUzI1NiJ9.eyJpZCI6ImExYjJjM2Q0..."    │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. Server extracts metadata                                │
│    tokenId = "a1b2c3d4-..."                                │
│    expiresAt = 1735862400000                               │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. Server saves ONLY metadata to MongoDB                   │
│    {tokenId, userId, expiresAt, createdAt, revoked}        │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ 5. Server sends JWT to user (JSON response)                │
│    {"token": "eyJhbGciOiJIUzI1NiJ9..."}                    │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ 6. User displays JWT in browser                            │
│    Web UI shows: "eyJhbGciOiJIUzI1NiJ9..."                 │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ 7. User clicks "Copy" or "Download"                        │
│    Token copied to clipboard or saved to file              │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ 8. Server discards JWT from memory                         │
│    jwt variable is garbage collected                       │
│    JWT NEVER stored anywhere                               │
└─────────────────────────────────────────────────────────────┘
```

---

## Summary

### Server Storage Policy:
✅ **Metadata ONLY** - tokenId, userId, expiresAt  
❌ **NEVER JWT** - The token string itself  

### Token Invalidation:
✅ **One token per user** - Only currentTokenId is valid  
✅ **Automatic invalidation** - New token invalidates ALL previous ones  
✅ **No revoke needed** - Just generate new token  
❌ **Old tokens stop working** - Even if not expired  

### User Responsibility:
✅ **Save token immediately**  
✅ **Store securely**  
✅ **Generate new if lost**  
✅ **Update client when generating new token**  

### Security Benefits:
✅ **Database breach safe**  
✅ **Log files safe**  
✅ **Backups safe**  
✅ **Insider threats minimized**  
✅ **Lost token? Generate new one** - Instant invalidation  

### Result:
🔒 **Maximum security through NOT storing sensitive data**  
🔒 **Automatic invalidation through single currentTokenId**  

**The best way to protect data is to not store it in the first place!**  
**The best way to revoke tokens is to allow only one at a time!**

---

## Build Status

✅ **BUILD SUCCESSFUL** - All code and documentation updated

