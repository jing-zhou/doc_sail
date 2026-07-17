# JWT Authentication Implementation Summary

## Date: December 3, 2025

## Overview
Today we implemented JWT (JSON Web Token) authentication as a new header type for the Illiad proxy server. This feature provides user management capabilities with MongoDB backend and a REST API for token generation.

## Changes Made

### 1. Dependencies Added (build.gradle.kts)
- **JWT Libraries**:
  - `io.jsonwebtoken:jjwt-api:0.12.5`
  - `io.jsonwebtoken:jjwt-impl:0.12.5`
  - `io.jsonwebtoken:jjwt-jackson:0.12.5`
- **Spring Security**: For BCrypt password encoding
  - `org.springframework.boot:spring-boot-starter-security`

### 2. Crypto Types Updated
- **Cryptos.java**: Added `JWT("JWT")` enum value
- **CryptoByte.java**: 
  - Added byte value `0x07` for JWT
  - JWT returns `0` for `byteLength()` (variable length)

### 3. New Model Classes
**User.java** (`com.illiad.server.model`)
- User entity with MongoDB mapping
- Fields: id, username, password (hashed), email, timestamps, active status, currentTokenId
- Uses Lombok `@Data` annotation
- MongoDB `@Document` and `@Indexed` annotations

**TokenInfo.java** (`com.illiad.server.model`)
- Token metadata entity
- Fields: id, userId, tokenId (UUID), createdAt, expiresAt, revoked
- Tracks token lifecycle in database

### 4. Repository Interfaces
**UserRepository.java** (`com.illiad.server.repository`)
- Extends `ReactiveMongoRepository<User, String>`
- Methods: `findByUsername()`, `existsByUsername()`

**TokenRepository.java** (`com.illiad.server.repository`)
- Extends `ReactiveMongoRepository<TokenInfo, String>`
- Methods: `findByTokenId()`, `deleteByUserId()`

### 5. Service Layer
**JwtService.java** (`com.illiad.server.service`)
- JWT token generation with configurable expiration
- Token validation and parsing
- Uses JJWT library with HS256 signing
- Returns `TokenPair` (jwt string, tokenId, expiresAt)

**UserService.java** (`com.illiad.server.service`)
- User registration with BCrypt password hashing
- User authentication
- Token generation (one active token per user)
- Token validation (checks JWT + database + user status)
- Token revocation

### 6. REST API Controllers
**AuthController.java** (`com.illiad.server.controller`)
Endpoints:
- `POST /api/auth/register` - Register new user
- `POST /api/auth/login` - Authenticate user
- `POST /api/auth/token/generate?userId=xxx` - Generate JWT token
- `POST /api/auth/token/revoke?userId=xxx` - Revoke all tokens

### 7. DTOs
**LoginRequest.java** - username, password
**RegisterRequest.java** - username, password, email
**TokenGenerateRequest.java** - expirationMinutes
**ApiResponse.java** - Generic response wrapper with success, message, data

### 8. Security Configuration
**SecurityConfig.java** (`com.illiad.server.config`)
- Disables CSRF (not needed for stateless API)
- Permits all requests (authentication handled by Illiad protocol)

### 9. Updated Secret Verification
**SecretImp.java** (`com.illiad.server.security`)
- Enhanced `verify()` method to handle JWT authentication
- For crypto type `0x07` (JWT):
  - Converts secret bytes to JWT string
  - Validates JWT using `UserService.validateToken()`
  - Checks signature, expiration, database record, and user status
- Falls back to hash-based verification for other crypto types

### 10. Configuration
**application.properties**
Added:
```properties
# MongoDB Configuration
spring.data.mongodb.uri=mongodb://localhost:27017/illiad_proxy

# JWT Configuration
jwt.secret=your-secret-key-must-be-at-least-256-bits-long-for-hs256-change-this-in-production
```

### 11. Documentation
**JWT_AUTHENTICATION.md**
- Complete feature documentation
- Architecture overview
- API endpoints with examples
- Token lifecycle explanation
- MongoDB collection schemas
- Security considerations
- Client usage examples
- Migration guide

## JWT Header Format

```
+--------+--------+----------------------+------+
| Length (2 bytes)| Crypto | JWT Token (variable)|CRLF|
+--------+--------+----------------------+------+
```

- **Length**: For variable-length JWT (crypto 0x07) the Length field encodes the JWT token byte length only (does NOT include CRLF). For fixed-length crypto types the Length field is the total header length (includes the 2-byte Length, 1-byte Crypto, signature, and CRLF).
- **Crypto**: `0x07` for JWT
- **JWT Token**: Variable-length UTF-8 encoded JWT string
- **CRLF**: `0x0D 0x0A` and MUST immediately follow the token for JWT (no padding/offset is used in strict mode)

**Semantics**:
- Fixed-signature crypto: Length = total header length; decoder validates CRLF at expected position: `readerIndex + length - 2`.
- Variable-signature (JWT): Length = signature length only; decoder waits for `3 + length + 2` bytes and validates CRLF immediately after the signature. If CRLF is not immediately present, the header is considered invalid and the connection is rerouted to HTTPS.

## JWT Token Structure

The JWT contains only two claims for minimal payload size and maximum privacy:

```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "expiresAt": 1733270400000
}
```

- **id**: Random UUID that correlates to a user in the database (no user information exposed)
- **expiresAt**: Expiration timestamp in epoch milliseconds

This design ensures:
- Minimal token size for efficient transmission in UDP packets
- No personally identifiable information in the token
- Explicit expiration validation
- Token ID rotation on each generation

## Validation Flow

1. Client connects with Illiad header containing JWT
2. `HeaderDecoder` parses header and extracts JWT token
3. `SecretImp.verify()` is called with crypto type `0x07`
4. JWT signature and expiration are validated
5. Token ID is looked up in MongoDB
6. User status and current token ID are verified
7. **Valid**: Proceed to SOCKS5 handler
8. **Invalid**: Reroute to HTTPS handler

## Token Management Features

### One-Time Download
- Tokens displayed once upon generation
- Not stored permanently on server after display
- User must save token securely
- Lost tokens require generating new one

### Security Features
- One active token per user at a time
- New token generation invalidates previous token
- Tokens contain random UUID (no user info exposed)
- BCrypt password hashing
- JWT signature validation
- Expiration enforcement
- Database-backed token revocation

### Expiration Options
Users can choose token validity:
- 1 day: 1,440 minutes
- 5 days: 7,200 minutes
- 1 week: 10,080 minutes
- 2 weeks: 20,160 minutes
- 1 month: 43,200 minutes
- 3 months: 129,600 minutes (maximum)

## Backward Compatibility

- **No breaking changes**: Existing hash-based authentication continues to work
- **Dual mode**: Both JWT and hash authentication supported simultaneously
- **Gradual migration**: Clients can migrate at their own pace

## Next Steps for Deployment

### Development/Testing
1. Install and start MongoDB locally:
   ```bash
   sudo apt-get install mongodb
   sudo systemctl start mongodb
   ```

2. Update `application.properties` if needed

3. Run the server:
   ```bash
   ./gradlew bootRun
   ```

4. Test API endpoints using curl or Postman

### Production Deployment
1. **MongoDB Setup**: Use cloud MongoDB service (Atlas, DocumentDB, etc.)
2. **Update Configuration**:
   - Change `jwt.secret` to a strong random key
   - Update MongoDB connection URI
   - Configure TLS/SSL certificates
3. **Security Hardening**:
   - Enable MongoDB authentication
   - Use encrypted MongoDB connections
   - Set up rate limiting
   - Configure proper firewall rules
4. **Monitoring**: Set up logging and monitoring for failed auth attempts

## Testing Suggestions

### Manual Testing
1. Register a user via POST `/api/auth/register`
2. Login via POST `/api/auth/login`
3. Generate token via POST `/api/auth/token/generate`
4. Use token in Illiad header to connect to proxy
5. Verify connection works
6. Generate new token and verify old token is invalidated

### Integration Testing
- Test token expiration
- Test token revocation
- Test concurrent token generation
- Test invalid JWT tokens
- Test expired JWT tokens
- Test rerouting to HTTPS on invalid tokens

## Benefits Achieved

1. **User Management**: Individual user accounts with credentials
2. **Auditability**: Track which user/token uses the proxy
3. **Flexibility**: Users control their token expiration
4. **Security**: Token revocation, expiration, one-token-per-user
5. **Scalability**: MongoDB backend supports large user base
6. **Integration**: REST API for user operations
7. **Future Ready**: Foundation for billing, quotas, analytics

## Files Modified/Created

### Created (17 files):
1. `/src/main/java/com/illiad/server/model/User.java`
2. `/src/main/java/com/illiad/server/model/TokenInfo.java`
3. `/src/main/java/com/illiad/server/repository/UserRepository.java`
4. `/src/main/java/com/illiad/server/repository/TokenRepository.java`
5. `/src/main/java/com/illiad/server/service/JwtService.java`
6. `/src/main/java/com/illiad/server/service/UserService.java`
7. `/src/main/java/com/illiad/server/controller/AuthController.java`
8. `/src/main/java/com/illiad/server/dto/LoginRequest.java`
9. `/src/main/java/com/illiad/server/dto/RegisterRequest.java`
10. `/src/main/java/com/illiad/server/dto/TokenGenerateRequest.java`
11. `/src/main/java/com/illiad/server/dto/ApiResponse.java`
12. `/src/main/java/com/illiad/server/config/SecurityConfig.java`
13. `JWT_AUTHENTICATION.md`

### Modified (3 files):
1. `build.gradle.kts` - Added JWT and Spring Security dependencies
2. `src/main/resources/application.properties` - Added MongoDB and JWT config
3. `src/main/java/com/illiad/server/security/SecretImp.java` - Added JWT validation

### Previously Updated (2 files):
1. `src/main/java/com/illiad/server/security/Cryptos.java` - Already had JWT enum
2. `src/main/java/com/illiad/server/security/CryptoByte.java` - Already had 0x07 mapping

## Build Status
✅ **Build Successful** - All files compiled without errors

## Conclusion

The JWT authentication feature is now fully implemented and ready for testing. It provides a complete user management system with:
- User registration and login
- JWT token generation with configurable expiration
- Token validation integrated into the Illiad protocol
- MongoDB backend for persistence
- REST API for all user operations
- Backward compatibility with existing hash-based auth

The implementation follows Spring Boot best practices with reactive programming (WebFlux), proper separation of concerns (model-repository-service-controller), and secure password handling.
