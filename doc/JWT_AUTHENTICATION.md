# JWT Authentication Feature

## Overview

The Illiad proxy server now supports JWT (JSON Web Token) as one of the authentication headers. This feature provides a flexible, user-managed token system that integrates with the existing Illiad protocol.

## Architecture

### Components

1. **JWT Header Type (`0x07`)**: A new variable-length header type for JWT authentication
2. **User Management System**: MongoDB-backed user database with registration and login
3. **Token Management**: One-time download tokens with configurable expiration
4. **REST API**: WebFlux-based API for user operations

### Header Format

The JWT header follows the Illiad protocol specification for variable-length headers:

```
+--------+--------+----------------------+------+
| Length (2 bytes)| Crypto | JWT Token (variable)|CRLF|
+--------+--------+----------------------+------+
```

- **Length (2 bytes)**: For JWT (crypto 0x07) the Length field encodes the JWT token byte length only (NOT including CRLF). For fixed-length crypto types the Length field is the total header length (see below).
- **Crypto (1 byte)**: `0x07` for JWT
- **JWT Token (variable)**: The JWT token string encoded as UTF-8 bytes
- **CRLF (2 bytes)**: `0x0D 0x0A` and MUST appear immediately after the JWT token (no padding/offset is used in the current strict mode)

**Important semantics**:
- Fixed-length signature crypto (e.g. hash-based types): the Length field is interpreted as the total header length (includes the 2-byte Length, 1-byte Crypto, signature bytes, and the trailing CRLF).
- Variable-length signature crypto (JWT - `0x07`): the Length field is the signature/token length only; CRLF must be immediately after the token. The decoder waits until `3 + length + 2` bytes are available before validating the CRLF.

### Validation Process

1. Client sends Illiad header with JWT token
2. Server extracts JWT from header (expects CRLF immediately after token)
3. Server validates JWT signature and expiration
4. Server looks up token ID in database
5. Server verifies token is current and user is active
6. If valid: proceed with SOCKS5 proxy
7. If invalid: reroute to HTTPS handler

## User Management API

All API endpoints are available via the disguise HTTPS server.

### Base URL
```
https://your-domain.com/api/auth
```

### Endpoints

#### 1. Register User
```
POST /api/auth/register
Content-Type: application/json

{
  "username": "user123",
  "password": "securePassword",
  "email": "user@example.com"
}

Response:
{
  "success": true,
  "message": "User registered successfully",
  "data": {
    "userId": "507f1f77bcf86cd799439011",
    "username": "user123"
  }
}
```

#### 2. Login
```
POST /api/auth/login
Content-Type: application/json

{
  "username": "user123",
  "password": "securePassword"
}

Response:
{
  "success": true,
  "message": "Login successful",
  "data": {
    "userId": "507f1f77bcf86cd799439011",
    "username": "user123"
  }
}
```

#### 3. Generate JWT Token
```
POST /api/auth/token/generate?userId=507f1f77bcf86cd799439011
Content-Type: application/json

{
  "expirationMinutes": 10080
}

Response:
{
  "success": true,
  "message": "Token generated successfully",
  "data": {
    "token": "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhYmMxMjMiLCJpYXQiOjE2MTYyMzkwMjIsImV4cCI6MTYxNjg0MzgyMn0.signature",
    "expirationMinutes": "10080"
  }
}
```

**Expiration Options:**
- 1 day: `1440` minutes
- 5 days: `7200` minutes
- 1 week: `10080` minutes
- 2 weeks: `20160` minutes
- 1 month: `43200` minutes
- 3 months: `129600` minutes (maximum)

#### 4. Revoke All Tokens
```
POST /api/auth/token/revoke?userId=507f1f77bcf86cd799439011

Response:
{
  "success": true,
  "message": "All tokens revoked",
  "data": null
}
```

## Token Behavior

### JWT Token Structure
The JWT token contains only two claims:
- **id**: A random UUID that correlates to a user in the database
  - **Reveals NOTHING about the user** (not username, email, or any personal info)
  - Only the server's database knows which user this UUID belongs to
  - Changes with each new token generation
- **expiresAt**: The expiration timestamp in epoch milliseconds

This minimal structure ensures:
- No user information is exposed in the token
- Token size is minimal for efficient transmission
- Expiration is explicitly encoded in the token itself

Example JWT payload (decoded):
```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "expiresAt": 1733270400000
}
```

**Privacy Note:** Even if someone intercepts and decodes your token, they cannot identify you. The `id` is just a random UUID that only the server can correlate to your account through its database.

### One-Time Download
- Token is displayed **once** when generated
- **Server never saves the JWT string** - only metadata (tokenId, userId, expiresAt)
- User must save token immediately (copy or download)
- If lost, generate new token (old one automatically invalidated)
- Server cannot retrieve your JWT after you leave the page

**Why server doesn't store JWTs:**
- Enhanced security: Database breach won't leak JWT tokens
- User responsibility: You control your own credentials
- No server-side JWT logs or backups
- Even server admins cannot see your JWT string

### Token Lifecycle
1. **Generation**: User can generate token **anytime** with desired expiration (free, unlimited)
2. **Download**: Token is displayed once and not stored on server
3. **Usage**: Client uses token in Illiad header for proxy connections
4. **Expiration Enforcement**: Token **must** be regenerated before `expiresAt` date
   - Expiration is enforced - cannot extend expired tokens
   - User must generate new token to continue access
5. **Anytime Renewal**: User can generate new token **whenever they want**
   - Regular rotation recommended (e.g., every 30 days)
   - Not just for lost/compromised tokens
   - Generating new token **immediately invalidates ALL previous tokens**
   - Server updates `currentTokenId` - only newest token works

### Security Features
- Only one active token per user at any time
- New token generation **automatically** invalidates ALL previous tokens
- Users can generate new token **anytime** (instant revocation)
- Tokens contain random UUID (no user information exposed)
- Token ID correlates to user only in database
- Expired tokens are automatically rejected (enforcement)
- Old tokens (even if not expired) are rejected if new token generated
- **Instant revocation**: Generate new token → all old ones stop working immediately

## MongoDB Collections

### Users Collection
```javascript
{
  "_id": ObjectId("..."),
  "username": "user123",
  "password": "$2a$10$...", // bcrypt hash
  "email": "user@example.com",
  "createdAt": ISODate("..."),
  "updatedAt": ISODate("..."),
  "active": true,
  "currentTokenId": "abc-123-def-456"
}
```

### Tokens Collection
```javascript
{
  "_id": ObjectId("..."),
  "userId": "507f1f77bcf86cd799439011",
  "tokenId": "abc-123-def-456",
  "createdAt": ISODate("..."),
  "expiresAt": ISODate("..."),
  "revoked": false
}
```

## Configuration

### application.properties
```properties
# MongoDB
spring.data.mongodb.uri=mongodb://localhost:27017/illiad_proxy

# JWT Secret (change in production!)
jwt.secret=your-secret-key-must-be-at-least-256-bits-long-for-hs256-change-this-in-production
```

### MongoDB Setup

#### Local Development
```bash
# Install MongoDB
sudo apt-get install mongodb

# Start MongoDB
sudo systemctl start mongodb

# Verify
mongo --eval 'db.runCommand({ connectionStatus: 1 })'
```

#### Production (Cloud)
Most cloud providers offer MongoDB services:
- MongoDB Atlas
- AWS DocumentDB
- Google Cloud Firestore
- Azure Cosmos DB

Update the connection URI in `application.properties`:
```properties
spring.data.mongodb.uri=mongodb+srv://username:password@cluster.mongodb.net/illiad_proxy
```

## Client Usage Example

### Python Client
```python
import socket
import struct

# Your JWT token from the API
jwt_token = "eyJhbGciOiJIUzI1NiJ9..."

# Build Illiad JWT header
crypto_type = 0x07  # JWT
token_bytes = jwt_token.encode('utf-8')
crlf = b"\r\n"

# Length includes: 2 (length field) + 1 (crypto) + token length
length = 2 + 1 + len(token_bytes)

header = struct.pack('>H', length) + bytes([crypto_type]) + token_bytes + crlf

# Now send SOCKS5 command after the header
# ... (SOCKS5 connection code)
```

## Benefits Over Hash-Based Headers

1. **User Management**: Each user has their own credentials and tokens
2. **Revocation**: Tokens can be revoked immediately
3. **Expiration**: Built-in expiration enforcement
4. **Auditability**: Track which user/token accessed the proxy
5. **Flexibility**: Users can generate tokens with different expirations
6. **No Shared Secret**: Each user has unique tokens

## Migration from Hash-Based Auth

Both JWT and hash-based authentication work simultaneously:
- Existing clients with hash secrets continue to work
- New clients can use JWT tokens
- Gradual migration is possible
- No breaking changes to existing deployments

## Security Considerations

1. **Change JWT Secret**: Update `jwt.secret` in production
2. **HTTPS Only**: API endpoints should only be accessed via HTTPS
3. **Rate Limiting**: Consider adding rate limiting to token generation
4. **Token Storage**: Users should store tokens securely
5. **MongoDB Security**: Use authentication and encryption for MongoDB
6. **Backup**: Regular backups of user and token data
7. **Monitoring**: Log failed authentication attempts

## Future Enhancements

Potential future features:
- Token usage statistics
- Multiple tokens per user
- Token permissions/scopes
- Bandwidth quotas
- Payment integration
- Two-factor authentication
- Token refresh mechanism
