# Authentication API Guide

Recent changes (2025-12-06):
- Privacy improvements: the server no longer exposes internal database identifiers (`userId` / Mongo `ObjectId`) or the user's `username` in any API responses that also contain a `sessionId` or a proxy `token` (JWT).
- `sessionId` is a UUID returned to the client and set as an HTTP-only Secure cookie. The server stores the mapping between session UUID and internal `ObjectId` only on the backend.
- JWTs contain only a random `id` (UUID) and `expiresAt` timestamp — they do NOT include `username` or `userId`.
- APIs that previously returned both `username` and `sessionId` now return only `sessionId` (or minimal status objects) to avoid leaking identifying information.

Complete guide for user registration, login, logout, and JWT token generation.

## Base URL

All endpoints are prefixed with `/api/auth`

---

## 1. User Registration

Register a new user account.

### Endpoint
```
POST /api/auth/register
```

### Request Body
```json
{
  "username": "alice",
  "password": "SecurePassword123!",
  "email": "alice@example.com"
}
```

### Response (Success - 200 OK)
```json
{
  "success": true,
  "message": "User registered successfully",
  "data": {
    "username": "alice",
    "email": "alice@example.com"
  }
}
```

### Response (Error - 400 Bad Request)
```json
{
  "success": false,
  "message": "Username already exists",
  "data": null
}
```

### Notes
- Username must be unique
- Email must be unique
- Both username and email are immutable after registration
- A billing account is automatically created (free plan)
- User is activated by default (`valid: true`)

---

## 2. User Login

Login to get a session ID for web-based token generation.

### Endpoint
```
POST /api/auth/login
```

### Request Body
```json
{
  "username": "alice",
  "password": "SecurePassword123!"
}
```

**OR** you can login with email:
```json
{
  "username": "alice@example.com",
  "password": "SecurePassword123!"
}
```

### Response (Success - 200 OK)
```json
{
  "success": true,
  "message": "Login successful",
  "data": {
    "sessionId": "a1b2c3d4-e5f6-7890-1234-567890abcdef"
  }
}
```

### Response Headers
```
Set-Cookie: sessionId=a1b2c3d4-e5f6-7890-1234-567890abcdef; Path=/; HttpOnly; Secure; Max-Age=86400; SameSite=Strict
```

### Response (Error - 401 Unauthorized)
```json
{
  "success": false,
  "message": "Invalid username/email or password",
  "data": null
}
```

### Notes
- Session expires after 24 hours
- SessionId is returned in both the response body AND as an HTTP-only cookie
- The cookie is used for web-based interactions
- Apps can extract sessionId from the response body

---

## 3. User Logout

Invalidate the current session.

### Endpoint
```
POST /api/auth/logout
```

### Request
- SessionId can be provided via cookie (automatic in browsers)
- Or via `Cookie` header: `Cookie: sessionId=a1b2c3d4-e5f6-7890-1234-567890abcdef`

### Response (Success - 200 OK)
```json
{
  "success": true,
  "message": "Logout successful",
  "data": null
}
```

### Response Headers
```
Set-Cookie: sessionId=; Path=/; HttpOnly; Secure; Max-Age=0
```

### Notes
- Session is immediately invalidated
- Cookie is cleared
- Safe to call even if not logged in

---

## 4. Token Generation (3 Alternatives)

Generate a JWT token for proxy authentication. Supports 3 authentication methods.

### Endpoint
```
POST /api/auth/token/generate
```

### Alternative 1: Username + Password

**Use Case:** Mobile/desktop apps, initial token generation

**Request:**
```json
{
  "username": "alice",
  "password": "SecurePassword123!",
  "expirationMinutes": 43200
}
```

**Can also use email:**
```json
{
  "username": "alice@example.com",
  "password": "SecurePassword123!",
  "expirationMinutes": 43200
}
```

### Alternative 2: SessionId

**Use Case:** Web browser users after login

**Request:**
```json
{
  "sessionId": "a1b2c3d4-e5f6-7890-1234-567890abcdef",
  "expirationMinutes": 43200
}
```

**Note:** If sessionId cookie is present, you can omit the sessionId field:
```json
{
  "expirationMinutes": 43200
}
```

### Alternative 3: Current Token (Renewal)

**Use Case:** Periodic background token renewal by apps

**Request:**
```json
{
  "currentToken": "eyJhbGciOiJIUzI1NiJ9.eyJpZCI6ImExYjJjM2Q0...",
  "expirationMinutes": 43200
}
```

### Response (Success - 200 OK)

**All alternatives return the same response format:**
```json
{
  "success": true,
  "message": "Token generated successfully",
  "data": {
    "token": "eyJhbGciOiJIUzI1NiJ9.eyJpZCI6IjEyMzQ1Njc4LTkwYWItY2RlZi0xMjM0LTU2Nzg5MGFiY2RlZiIsImV4cGlyZXNBdCI6MTczMzY1NDQwMDAwMH0.abc123...",
    "expirationMinutes": 43200,
    "expiresAt": "2025-12-08T10:30:00Z"
  }
}
```

### Response (Error - 401 Unauthorized)

**Invalid credentials (Alternative 1):**
```json
{
  "success": false,
  "message": "Invalid username or password",
  "data": null
}
```

**Invalid session (Alternative 2):**
```json
{
  "success": false,
  "message": "Invalid or expired session",
  "data": null
}
```

**Invalid/expired token (Alternative 3):**
```json
{
  "success": false,
  "message": "Token is expired, please use username+password to generate a new token",
  "data": null
}
```

### Response (Error - 400 Bad Request)

**No valid alternative provided:**
```json
{
  "success": false,
  "message": "Must provide one of: sessionId, currentToken, or username+password",
  "data": null
}
```

### Authentication Priority

If multiple alternatives are provided, the server processes them in this order:
1. **SessionId** (from body or cookie) - highest priority
2. **CurrentToken** (token renewal)
3. **Username + Password** - lowest priority

### Important Notes

1. **Token Security:**
   - Only ONE active token per user at any time
   - Generating a new token automatically invalidates ALL previous tokens
   - Tokens are NEVER stored in the database (only the tokenId UUID is stored)

2. **expiresAt Field:**
   - Critical for apps that treat JWT as opaque string
   - Apps can display "Token valid until Dec 8, 2025" to users
   - Apps can warn users when token is about to expire

3. **Expiration Minutes:**
   - Common values: 1440 (1 day), 10080 (1 week), 43200 (30 days)
   - Max recommended: 129600 (90 days) due to Let's Encrypt certificate rotation

---

## Complete Workflow Examples

### Example 1: Mobile App (First-Time User)

1. User installs app and opens it
2. User enters username/password in app
3. App calls `/api/auth/token/generate` with username+password
4. App stores the JWT token
5. App sets up periodic renewal (every 10-30 minutes)
6. App uses JWT as Illiad header for proxy requests

```javascript
// Step 3: Generate token
const response = await fetch('https://server.com/api/auth/token/generate', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    username: 'alice',
    password: 'SecurePassword123!',
    expirationMinutes: 43200  // 30 days
  })
});

const data = await response.json();
if (data.success) {
  localStorage.setItem('jwt_token', data.data.token);
  localStorage.setItem('token_expires_at', data.data.expiresAt);
  startPeriodicRenewal();  // Renew every 10 minutes
}
```

### Example 2: Web Browser User

1. User visits `https://server.com` in browser
2. User clicks "Login" and enters username/password
3. Browser calls `/api/auth/login` → receives sessionId cookie
4. User clicks "Generate Token" button
5. Browser calls `/api/auth/token/generate` (sessionId from cookie)
6. User downloads/copies the JWT token
7. User configures client app with the JWT token

```javascript
// Step 3: Login
const loginResponse = await fetch('https://server.com/api/auth/login', {
  method: 'POST',
  credentials: 'include',  // Send cookies
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    username: 'alice',
    password: 'SecurePassword123!'
  })
});

// Step 5: Generate token (sessionId from cookie)
const tokenResponse = await fetch('https://server.com/api/auth/token/generate', {
  method: 'POST',
  credentials: 'include',  // Send sessionId cookie
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    expirationMinutes: 43200  // SessionId from cookie is used
  })
});

const data = await tokenResponse.json();
if (data.success) {
  // Display token for user to copy
  displayToken(data.data.token);
  displayExpiration(data.data.expiresAt);
}
```

### Example 3: Periodic Token Renewal (App Background)

```javascript
class TokenManager {
  constructor() {
    this.token = localStorage.getItem('jwt_token');
    this.renewalInterval = 10 * 60 * 1000;  // 10 minutes
    this.consecutiveFailures = 0;
  }
  
  startPeriodicRenewal() {
    setInterval(() => this.renewToken(), this.renewalInterval);
  }
  
  async renewToken() {
    try {
      const response = await fetch('https://server.com/api/auth/token/generate', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          currentToken: this.token,
          expirationMinutes: 43200
        })
      });
      
      const data = await response.json();
      if (data.success) {
        this.token = data.data.token;
        localStorage.setItem('jwt_token', this.token);
        this.consecutiveFailures = 0;
        console.log('Token renewed silently');
      } else {
        this.handleFailure();
      }
    } catch (error) {
      this.handleFailure();
    }
  }
  
  handleFailure() {
    this.consecutiveFailures++;
    if (this.consecutiveFailures >= 5) {
      alert('Cannot connect to server. Please check your connection.');
    }
  }
}
```

---

## Security Best Practices

### For App Developers

1. **Never expose userId** - use username, email, or sessionId
2. **Store tokens securely** - use encrypted storage on mobile
3. **Implement periodic renewal** - every 10-30 minutes
4. **Monitor renewal failures** - detect server/network issues early
5. **Use HTTPS only** - never send credentials over HTTP

### For Web Developers

1. **Use HTTP-only cookies** for sessionId - prevents XSS attacks
2. **Never store passwords** - only store sessionId temporarily
3. **Display token expiration** - show users when renewal is needed
4. **Implement logout** - always provide a logout button
5. **Use SameSite cookies** - prevents CSRF attacks

### For Server Administrators

1. **Rotate JWT secret** regularly (but carefully - invalidates all tokens)
2. **Monitor failed login attempts** - implement rate limiting
3. **Log authentication events** - audit trail for security
4. **Use strong password policies** - enforce minimum complexity
5. **Implement account lockout** - after N failed attempts

---

## Error Handling

### HTTP Status Codes

- `200 OK` - Success
- `307 Temporary Redirect` - Server asks client to resend the request (occurs when client tries to access server itself through proxy - self-connection loop detected)
- `400 Bad Request` - Invalid request (missing required fields)
- `401 Unauthorized` - Authentication failed (wrong credentials, expired session/token)
- `500 Internal Server Error` - Server error (check logs)

**Note on 307 Temporary Redirect:**
This status code is returned whenever the proxy service detects that a client is trying to connect to the server itself (destination IP and port match the server's listening address). This can occur with ANY request - registration, login, token generation, or any other API call when routed through the proxy. The server responds with 307 to ask the client to resend the request directly to the HTTPS server without the Illiad proxy header, preventing an infinite self-connection loop.

### Common Error Messages

| Error Message | Cause | Solution |
|--------------|-------|----------|
| "307 Temporary Redirect" (HTTP Status) | Self-connection loop detected - client trying to access server through proxy | Client should resend request directly to HTTPS server without Illiad proxy header |
| "Username already exists" | Username taken during registration | Choose different username |
| "Email already exists" | Email taken during registration | Use different email or login |
| "Invalid username/email or password" | Wrong credentials | Check username/email and password |
| "Invalid or expired session" | SessionId expired or invalid | Login again |
| "Token is expired" | JWT token expired | Generate new token with username+password |
| "Must provide one of: sessionId, currentToken, or username+password" | No valid auth method | Provide at least one authentication method |

---

## Testing with cURL

### Register
```bash
curl -X POST https://server.com/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"SecurePass123!","email":"alice@example.com"}'
```

### Login
```bash
curl -X POST https://server.com/api/auth/login \
  -H "Content-Type: application/json" \
  -c cookies.txt \
  -d '{"username":"alice","password":"SecurePass123!"}'
```

### Generate Token (with sessionId from cookie)
```bash
curl -X POST https://server.com/api/auth/token/generate \
  -H "Content-Type: application/json" \
  -b cookies.txt \
  -d '{"expirationMinutes":43200}'
```

### Generate Token (with username+password)
```bash
curl -X POST https://server.com/api/auth/token/generate \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"SecurePass123!","expirationMinutes":43200}'
```

### Logout
```bash
curl -X POST https://server.com/api/auth/logout \
  -b cookies.txt
```

---

## FAQ

**Q: Can I have multiple active tokens?**  
A: No, only ONE active token per user. Generating a new token invalidates all previous tokens.

**Q: What happens if my token expires?**  
A: You must generate a new token using username+password (Alternative 1).

**Q: Can I renew an expired token?**  
A: No, expired tokens cannot be renewed. Use username+password to generate a new one.

**Q: How often should I renew tokens?**  
A: Apps should renew every 10-30 minutes regardless of expiration (periodic renewal pattern).

**Q: Is sessionId the same as JWT token?**  
A: No. SessionId is for web-based token generation. JWT token is for proxy authentication.

**Q: Can I login with email instead of username?**  
A: Yes, both login and token generation support email as username.

**Q: What's the recommended token expiration?**  
A: 30 days (43200 minutes) is common. With periodic renewal, actual expiration doesn't matter much.

**Q: Can I use the API without a browser?**  
A: Yes, use Alternative 1 (username+password) for apps. SessionId/cookie is only for web browsers.
