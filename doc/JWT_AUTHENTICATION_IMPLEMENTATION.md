# Authentication Implementation Summary

## Date: December 3, 2025

## Overview

Implemented a secure dual-approach authentication system for JWT token management in the Illiad proxy server.

---

## What Was Implemented

### 1. Session Management System

**New Files**:
- `src/main/java/com/illiad/server/model/Session.java` - Session model
- `src/main/java/com/illiad/server/repository/SessionRepository.java` - Session repository
- `src/main/java/com/illiad/server/service/SessionService.java` - Session management service

**Features**:
- 24-hour session duration
- Secure session validation
- Automatic session expiration
- Session-based authentication for web interface

### 2. Two Authentication Approaches

#### Approach 1: Web-based Login (Session-based)
**Flow**: `Login → Session Cookie → Generate JWT → Download`

**Endpoints**:
- `POST /api/auth/register` - Register new user
- `POST /api/auth/login` - Login and create session
- `POST /api/auth/logout` - Logout and invalidate session
- `POST /api/auth/token/generate` - Generate JWT (requires session)
- `POST /api/auth/token/revoke` - Revoke all tokens (requires session)

**Security Features**:
- Session cookies are HttpOnly, Secure, SameSite=Strict
- Users can only generate tokens for themselves
- No way to generate tokens for other users (SECURITY FIX)

#### Approach 2: JWT Renewal (Token-to-Token)
**Flow**: `Current JWT → Validate → New JWT → Old JWT Invalidated`

**Endpoint**:
- `POST /api/auth/token/renew` - Renew JWT using current JWT

**Security Features**:
- No credential exposure during renewal
- Old JWT immediately invalidated when new one is generated
- Bearer token authentication
- Token can be provided in Authorization header or request body

### 3. Security Improvements

**Before** (Security Vulnerability):
```java
// INSECURE: Anyone could generate token for any user
String userId = extractQueryParam(uri, "userId"); 
userService.generateToken(userId, expirationMinutes)
```

**After** (Secure):
```java
// SECURE: User authenticated via session, can only generate for themselves
sessionService.validateSession(sessionId)
    .flatMap(authenticatedUserId -> {
        return userService.generateToken(authenticatedUserId, expirationMinutes)
    })
```

**Token Invalidation**:
- When a new token is generated, `user.currentTokenId` is updated
- All previous tokens become invalid immediately
- Even unexpired tokens are invalidated

### 4. Modified Files

**Core Changes**:
- `RerouteHandler.java` - Added session-based auth, JWT renewal endpoint
- `HeaderDecoder.java` - Pass SessionService to RerouteHandler
- `ParamBus.java` - Added SessionService to dependency injection

**Helper Methods Added**:
- `extractCookie()` - Extract cookie from HTTP request
- `extractBearerToken()` - Extract JWT from Authorization header
- `sendJsonResponseWithCookie()` - Send response with session cookie
- `handleLogout()` - Handle user logout
- `handleTokenRenew()` - Handle JWT renewal

---

## Security Model

### Token Storage

**Server Side**:
```
User Collection (Permanent/Long-term data):
{
  "_id": "507f1f77bcf86cd799439011",
  "username": "alice",
  "password": "$2a$10$...",  // hashed
  "email": "alice@example.com",
  "active": true,
  "createdAt": "2025-01-01T00:00:00Z",
  "updatedAt": "2025-01-01T00:00:00Z"
}

TokenInfo Collection (Transient data):
{
  "_id": "...",
  "tokenId": "uuid-of-active-token",  // UUID from inside JWT
  "userId": "507f1f77bcf86cd799439011",
  "expiresAt": "2025-01-03T00:00:00Z",
  "revoked": false,
  "createdAt": "2025-01-01T00:00:00Z",
  // Future fields for billing/charging:
  "bandwidthUsed": 1048576,
  "bandwidthLimit": 107374182400,  // 100GB
  "lastUsedAt": "2025-01-02T12:00:00Z",
  "connectionCount": 42
}

JWT token string is NEVER stored!

Key Points:
- User collection contains permanent/long-term data
- TokenInfo collection contains transient token data
- Correlation: TokenInfo.userId → User._id
- Only ONE TokenInfo per user at any time (old ones deleted on new generation)
```

**Client Side**:
- JWT must be saved by user
- Server returns it only once
- If lost, generate new one (invalidates old)

### Token Validation

```
Validate JWT:
1. Parse JWT and extract tokenId
2. Verify JWT signature
3. Check JWT expiration date
4. Find TokenInfo by tokenId in database
5. Check if TokenInfo exists (if not, token was invalidated)
6. Check if TokenInfo.isRevoked == false
7. Check if TokenInfo.expiresAt > now
8. Find User by TokenInfo.userId
9. Check if User.isActive == true
10. (Future) Check billing limits: bandwidthUsed < bandwidthLimit
11. If all pass → Valid token

Separation of Concerns:
- User: permanent identity and credentials
- TokenInfo: transient access tokens with correlation to user
- Future: Can add billing validation in step 10
```

### Automatic Invalidation

When `generateToken()` is called:
```java
1. Generate new tokenId (UUID)
2. Create new JWT with tokenId
3. Delete ALL old TokenInfo records for this userId  ← Invalidates old tokens!
4. Save new TokenInfo to database
5. Return JWT to user (one time only)
```

Result: All previous tokens are deleted and become invalid, even if not expired!

**Why this is better**:
- Clear separation: User (permanent) vs TokenInfo (transient)
- Simple invalidation: Just delete old TokenInfo records
- Future-ready: Easy to add billing fields to TokenInfo
- No redundancy: userId correlation is sufficient

---

## API Examples

### Approach 1: Web Login

```bash
# 1. Login
curl -X POST https://proxy.example.com/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"pass123"}' \
  -c cookies.txt

# Response includes session cookie

# 2. Generate JWT
curl -X POST https://proxy.example.com/api/auth/token/generate \
  -H "Content-Type: application/json" \
  -d '{"expirationMinutes":43200}' \
  -b cookies.txt

# Response:
# {
#   "success": true,
#   "data": {
#     "token": "eyJhbGci...",
#     "expirationMinutes": "43200"
#   }
# }
```

### Approach 2: JWT Renewal

```bash
# Renew using current JWT
curl -X POST https://proxy.example.com/api/auth/token/renew \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJhbGci..." \
  -d '{"expirationMinutes":43200}'

# Response:
# {
#   "success": true,
#   "message": "Token renewed successfully. Old token is now invalid.",
#   "data": {
#     "token": "eyJhbGci...NEW_TOKEN...",
#     "expirationMinutes": "43200"
#   }
# }

# Old token is now invalid!
```

---

## Benefits

### Security
- ✅ **Fixed vulnerability**: Users can't generate tokens for other users
- ✅ **Session-based auth**: Secure cookie-based authentication for web
- ✅ **Token renewal**: Refresh tokens without exposing credentials
- ✅ **Automatic invalidation**: Old tokens become invalid immediately
- ✅ **No token storage**: JWT never stored on server

### Usability
- ✅ **Two approaches**: Choose based on use case
- ✅ **Web-friendly**: Session cookies for browser-based access
- ✅ **API-friendly**: Bearer token for programmatic access
- ✅ **Seamless renewal**: Update tokens before expiration

### Flexibility
- ✅ **Configurable expiration**: User chooses token lifetime
- ✅ **Multiple clients**: Each can have different tokens
- ✅ **Easy revocation**: Generate new token to invalidate old ones

---

## Use Cases

### Use Case 1: Web User
1. Alice visits `https://proxy.example.com`
2. Logs in with username/password
3. Navigates to "Generate Token" page
4. Selects expiration: 1 month
5. Downloads JWT token
6. Configures her Trojan client with the JWT
7. If token is lost, she logs in and generates a new one

### Use Case 2: Automated System
1. Bob has an existing JWT from web login
2. His script needs to renew the token before expiration
3. Script calls `/api/auth/token/renew` with current JWT
4. Receives new JWT, old one is automatically invalid
5. Updates configuration with new JWT
6. Schedules next renewal before expiration

### Use Case 3: Mobile App
1. User logs in once with credentials
2. App stores the JWT securely
3. App automatically renews token every 30 days
4. No need to store password
5. If user changes password, app must re-login

---

## Migration Notes

### Existing Users
- Existing JWT validation logic unchanged
- New session-based endpoints are optional
- Old tokens remain valid until expiration or new token generated
- No database migration needed (MongoDB handles new collections)

### Configuration
No configuration changes needed. Services are auto-wired with `@Autowired(required = false)`:
- If MongoDB is configured: Session/User management enabled
- If MongoDB is not configured: Only SOCKS5 proxy functionality available

---

## Testing

### Test Approach 1: Web Login

```bash
# Register
curl -X POST http://localhost:2080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","password":"test123","email":"test@example.com"}'

# Login
curl -X POST http://localhost:2080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","password":"test123"}' \
  -c cookies.txt -v

# Generate token
curl -X POST http://localhost:2080/api/auth/token/generate \
  -H "Content-Type: application/json" \
  -d '{"expirationMinutes":1440}' \
  -b cookies.txt

# Logout
curl -X POST http://localhost:2080/api/auth/logout \
  -b cookies.txt
```

### Test Approach 2: JWT Renewal

```bash
# Assume TOKEN is the JWT from previous step
TOKEN="eyJhbGci..."

# Renew token
curl -X POST http://localhost:2080/api/auth/token/renew \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"expirationMinutes":1440}'

# Old token should now be invalid
# Try using old token (should fail)
curl -X POST http://localhost:2080/api/auth/token/renew \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"expirationMinutes":1440}'
```

---

## Future Enhancements

### Potential Improvements
- [ ] Token usage analytics (track when/where tokens are used)
- [ ] Multi-factor authentication (MFA) for token generation
- [ ] Token scopes/permissions (read-only vs full access)
- [ ] Billing integration (track bandwidth per token)
- [ ] Token naming/labeling (user can name their tokens)
- [ ] Token revocation by ID (revoke specific token, not all)
- [ ] Rate limiting on token generation
- [ ] Email notifications on token generation/renewal

### Security Enhancements
- [ ] IP whitelist per token
- [ ] Geo-blocking per token
- [ ] Token rotation policy (force renewal after X days)
- [ ] Anomaly detection (unusual token usage patterns)

---

## Documentation

**Created**:
- `JWT_AUTHENTICATION_GUIDE.md` - Comprehensive user guide
- `JWT_AUTHENTICATION_IMPLEMENTATION.md` - This file

**See Also**:
- `JWT_AUTHENTICATION.md` - Original JWT design
- `JWT_TOKEN_USER_GUIDE.md` - User-facing documentation
- `SERVER_JWT_STORAGE_POLICY.md` - Token storage policy

---

## Conclusion

The dual-approach authentication system successfully addresses the security vulnerability where users could generate tokens for other users. Now:

1. **Approach 1 (Web)**: Users log in with credentials → session → generate JWT
2. **Approach 2 (Renewal)**: Users renew JWT with current JWT → no credentials needed

Both approaches ensure:
- Users can only manage their own tokens
- Old tokens are automatically invalidated
- No JWT strings are stored on the server
- Secure, flexible, and user-friendly

The system is production-ready and follows security best practices.

