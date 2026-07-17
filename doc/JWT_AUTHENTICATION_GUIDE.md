# JWT Authentication Guide

This guide explains the two authentication approaches for generating JWT tokens in the Illiad proxy server.

## Overview

The system supports two distinct approaches for JWT token management:

1. **Web-based Login Approach**: Traditional username/password authentication via web interface
2. **JWT Renewal Approach**: Token-to-token renewal without exposing credentials

---

## Approach 1: Web-based Login (Session-based)

This approach is used when a user wants to access the web interface, manage their account, and generate JWT tokens through a browser or API client.

### Flow

```
User → Login → Session Cookie → Generate JWT → Download Token
```

### Steps

#### 1. Register (One-time)

**Endpoint**: `POST /api/auth/register`

**Request**:
```json
{
  "username": "alice",
  "password": "securePassword123",
  "email": "alice@example.com"
}
```

**Response**:
```json
{
  "success": true,
  "message": "User registered successfully",
  "data": {
    "userId": "507f1f77bcf86cd799439011",
    "username": "alice"
  }
}
```

#### 2. Login

**Endpoint**: `POST /api/auth/login`

**Request**:
```json
{
  "username": "alice",
  "password": "securePassword123"
}
```

**Response**:
```json
{
  "success": true,
  "message": "Login successful",
  "data": {
    "userId": "507f1f77bcf86cd799439011",
    "username": "alice",
    "sessionId": "f47ac10b-58cc-4372-a567-0e02b2c3d479"
  }
}
```

**Cookie Set**: `SESSION_ID=f47ac10b-58cc-4372-a567-0e02b2c3d479; Path=/; HttpOnly; Secure; SameSite=Strict; Max-Age=86400`

- Session duration: 24 hours
- HttpOnly: Prevents JavaScript access
- Secure: Only sent over HTTPS
- SameSite=Strict: CSRF protection

#### 3. Generate JWT Token

**Endpoint**: `POST /api/auth/token/generate`

**Headers**: 
- `Cookie: SESSION_ID=f47ac10b-58cc-4372-a567-0e02b2c3d479`

**Request**:
```json
{
  "expirationMinutes": 43200
}
```

**Available expiration options**:
- 1 day: `1440` minutes
- 3 days: `4320` minutes
- 1 week: `10080` minutes
- 2 weeks: `20160` minutes
- 1 month: `43200` minutes
- 3 months: `129600` minutes
- 6 months: `259200` minutes (maximum)

**Response**:
```json
{
  "success": true,
  "message": "Token generated successfully",
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjUwN2YxZjc3YmNmODZjZDc5OTQzOTAxMSIsImV4cGlyZXNBdCI6MTczMzE2MDAwMDAwMH0.signature",
    "expirationMinutes": "43200"
  }
}
```

**Important Notes**:
- The JWT token is **returned only once** and **never stored** on the server
- User must save/download the token immediately
- If lost, user can generate a new token (which invalidates all previous tokens)
- Each new token generation invalidates **all previous tokens** for that user

#### 4. Logout (Optional)

**Endpoint**: `POST /api/auth/logout`

**Headers**: 
- `Cookie: SESSION_ID=f47ac10b-58cc-4372-a567-0e02b2c3d479`

**Response**:
```json
{
  "success": true,
  "message": "Logged out successfully"
}
```

---

## Approach 2: JWT Renewal (Token-to-Token)

This approach allows users to renew their JWT token without exposing their username/password. Perfect for automated systems, mobile apps, or when credentials should not be stored.

### Flow

```
User → Provide Current JWT → Validate → Generate New JWT → Old JWT Invalidated
```

### Usage

**Endpoint**: `POST /api/auth/token/renew`

**Headers**: 
- `Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...` (current JWT)

**Alternative** (token in body):
```json
{
  "currentToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expirationMinutes": 43200
}
```

**Request**:
```json
{
  "expirationMinutes": 43200
}
```

**Response**:
```json
{
  "success": true,
  "message": "Token renewed successfully. Old token is now invalid.",
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.NEW_TOKEN_CONTENT.signature",
    "expirationMinutes": "43200"
  }
}
```

### Key Features

1. **No Credential Exposure**: Password is never transmitted
2. **Automatic Invalidation**: Old JWT becomes invalid immediately
3. **Seamless Renewal**: Client can refresh tokens before expiration
4. **Security**: Even if old token is leaked, it's already invalid

### Example Workflow

```bash
# Initial JWT (obtained from web login)
CURRENT_JWT="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."

# Renew token before expiration
curl -X POST https://proxy.example.com/api/auth/token/renew \
  -H "Authorization: Bearer $CURRENT_JWT" \
  -H "Content-Type: application/json" \
  -d '{"expirationMinutes": 43200}'

# Response contains NEW_JWT
# CURRENT_JWT is now invalid
# Use NEW_JWT for future renewals
```

---

## Comparison

| Feature | Web Login Approach | JWT Renewal Approach |
|---------|-------------------|---------------------|
| **Authentication** | Username + Password | Current JWT |
| **Session** | Yes (24 hours) | No |
| **Credential Exposure** | Yes (during login) | No |
| **Use Case** | Web interface, initial setup | Automated systems, mobile apps |
| **Token Invalidation** | Generates new, old still valid until new one created | Old token immediately invalid |
| **Revocation** | Requires session | Uses current JWT |

---

## Security Considerations

### Token Storage

**Server-side**:
- JWT string is **NEVER** stored in the database
- Only metadata is stored: `tokenId`, `userId`, `expiresAt`
- User's `currentTokenId` field tracks the active token

**Client-side**:
- Store JWT securely (encrypted storage, keychain, etc.)
- Never expose JWT in logs or URLs
- Treat JWT like a password

### Token Invalidation

When a new token is generated (either approach):
1. New `tokenId` (UUID) is generated
2. User's `currentTokenId` is updated to the new value
3. **All previous tokens are immediately invalid**, even if not expired
4. Token validation checks:
   - JWT signature valid
   - Token not expired (`expiresAt`)
   - `tokenId` matches user's `currentTokenId` in database

### Best Practices

1. **Regular Renewal**: Renew tokens before expiration
2. **Short Expiration**: Use shorter expiration for sensitive operations
3. **Revoke on Compromise**: If token is compromised, generate new one immediately
4. **Monitor Usage**: Track token generation/renewal in logs
5. **Rate Limiting**: Implement rate limiting on token endpoints

---

## API Reference

### Registration
- **Method**: POST
- **Path**: `/api/auth/register`
- **Auth**: None
- **Body**: `{username, password, email}`

### Login
- **Method**: POST
- **Path**: `/api/auth/login`
- **Auth**: None
- **Body**: `{username, password}`
- **Returns**: Session cookie

### Logout
- **Method**: POST
- **Path**: `/api/auth/logout`
- **Auth**: Session cookie
- **Body**: None

### Generate JWT (Web)
- **Method**: POST
- **Path**: `/api/auth/token/generate`
- **Auth**: Session cookie
- **Body**: `{expirationMinutes}`
- **Returns**: JWT token (one-time)

### Renew JWT
- **Method**: POST
- **Path**: `/api/auth/token/renew`
- **Auth**: Bearer token (current JWT)
- **Body**: `{expirationMinutes}` or `{currentToken, expirationMinutes}`
- **Returns**: New JWT token (old one invalidated)

### Revoke Tokens
- **Method**: POST
- **Path**: `/api/auth/token/revoke`
- **Auth**: Session cookie
- **Body**: None
- **Effect**: Invalidates all user's tokens

---

## Error Handling

### Common Errors

**401 Unauthorized**:
```json
{
  "success": false,
  "message": "Authentication required. Please login first."
}
```

**401 Unauthorized** (expired session):
```json
{
  "success": false,
  "message": "Session expired. Please login again."
}
```

**401 Unauthorized** (invalid JWT):
```json
{
  "success": false,
  "message": "Invalid or expired token. Please login again."
}
```

**400 Bad Request**:
```json
{
  "success": false,
  "message": "Invalid expiration time. Must be between 1 minute and 6 months"
}
```

---

## Integration Example

### JavaScript/Node.js Client

```javascript
class IlliadClient {
  constructor(baseUrl) {
    this.baseUrl = baseUrl;
    this.jwt = null;
  }

  // Approach 1: Login and generate JWT
  async loginAndGenerateToken(username, password, expirationMinutes) {
    // Login
    const loginRes = await fetch(`${this.baseUrl}/api/auth/login`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ username, password }),
      credentials: 'include' // Important: include cookies
    });
    
    const loginData = await loginRes.json();
    if (!loginData.success) throw new Error(loginData.message);

    // Generate JWT
    const tokenRes = await fetch(`${this.baseUrl}/api/auth/token/generate`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ expirationMinutes }),
      credentials: 'include'
    });
    
    const tokenData = await tokenRes.json();
    if (!tokenData.success) throw new Error(tokenData.message);
    
    this.jwt = tokenData.data.token;
    return this.jwt;
  }

  // Approach 2: Renew existing JWT
  async renewToken(expirationMinutes) {
    if (!this.jwt) throw new Error('No JWT available');

    const res = await fetch(`${this.baseUrl}/api/auth/token/renew`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${this.jwt}`
      },
      body: JSON.stringify({ expirationMinutes })
    });
    
    const data = await res.json();
    if (!data.success) throw new Error(data.message);
    
    // Update to new JWT (old one is now invalid)
    this.jwt = data.data.token;
    return this.jwt;
  }

  // Auto-renew before expiration
  async autoRenew(checkIntervalMs = 3600000) { // Check every hour
    setInterval(async () => {
      try {
        await this.renewToken(43200); // Renew for 1 month
        console.log('JWT renewed successfully');
      } catch (err) {
        console.error('Failed to renew JWT:', err);
      }
    }, checkIntervalMs);
  }
}

// Usage
const client = new IlliadClient('https://proxy.example.com');

// First time: login
const jwt = await client.loginAndGenerateToken('alice', 'password123', 43200);
console.log('JWT:', jwt);

// Later: renew without credentials
await client.renewToken(43200);

// Or: auto-renew
client.autoRenew();
```

---

## Conclusion

The dual-approach authentication system provides:
- **Flexibility**: Choose the right approach for your use case
- **Security**: Session-based for web, token-based for automation
- **Convenience**: Renew tokens without exposing credentials
- **Control**: Automatic invalidation of old tokens

For web users, use Approach 1 (login → session → generate JWT).
For automated systems, use Approach 2 (JWT renewal) for seamless token refresh.

