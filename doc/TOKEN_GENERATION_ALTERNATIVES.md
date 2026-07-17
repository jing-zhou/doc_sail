# Token Generation - 3 Authentication Alternatives

## Overview

The JWT token generation endpoint supports **3 different authentication methods**. Each method serves different use cases and security requirements.

**Important:** Only **ONE** alternative should be provided per request. The server will process the first valid alternative found in this order: sessionId → currentToken → username/password.

---

## Alternative 1: Username + Password (For Apps/Client Software)

### Use Case
- **Primary scenario:** Mobile apps, desktop clients, or client software
- User generates token directly without visiting the web interface
- One-step authentication and token generation
- User does NOT need to login to the server's web page
- Perfect for standalone applications that only need proxy access

### Request
```json
{
  "username": "alice",
  "password": "SecurePassword123!",
  "expirationMinutes": 43200
}
```

### Security Notes
- ✅ Most straightforward for client applications
- ✅ No browser or web session required
- ⚠️ App must securely store credentials
- ⚠️ Not recommended for web browsers (use Alternative 2 instead)

### Server Logic
1. Validate username and password
2. Look up user in database
3. Generate new JWT with new tokenId
4. Update user's tokenId field (invalidates all old tokens)
5. Return new JWT

---

## Alternative 2: SessionId (For Browser/Web Landing Page)

### Use Case
- **Primary scenario:** User visits the server's landing page using a web browser
- User logs in via web interface and receives a sessionId
- User then generates token through the web page UI
- Best UX for web-based token management
- No need to re-enter password for token generation

### Workflow
1. User opens browser and navigates to `https://your-server.com`
2. User enters username and password on the login page
3. Server validates credentials and returns a sessionId
4. User clicks "Generate Token" button on the web page
5. Server generates token using the sessionId (no password needed)
6. User downloads or copies the JWT token

### Request
```json
{
  "sessionId": "a1b2c3d4-e5f6-7890-1234-567890abcdef",
  "expirationMinutes": 43200
}
```

### Security Notes
- ✅ No password transmission after initial login
- ✅ SessionId has expiration (e.g., 1 hour)
- ✅ SessionId can be invalidated on logout
- ✅ Best practice for web browser interactions
- ✅ User-friendly for non-technical users

### Server Logic
1. Validate sessionId exists and not expired
2. Look up userId from session
3. Generate new JWT with new tokenId
4. Update user's tokenId field (invalidates all old tokens)
5. Return new JWT

---

## Alternative 3: Current Valid Token (For Token Renewal)

### Use Case
- **Primary scenario:** User wants to renew an existing token
- User has a valid token but wants to extend its expiration
- **Important:** Apps typically do NOT auto-renew because they cannot decode/verify JWT
- Apps treat JWT as an opaque string - they don't know when it expires
- Token renewal is usually a **manual user action** or triggered by server rejection

### How Apps Handle Token Renewal
Apps treat JWT as an **opaque string** - they don't decode, parse, or check expiration. Instead, apps use a **periodic renewal pattern** with health monitoring:

#### Recommended Pattern: Periodic Renewal with Health Monitoring

**Key Strategy:**
1. **Set a short renewal interval** (e.g., 10-30 minutes)
2. **Renew token periodically** regardless of actual expiration
3. **Monitor renewal health** - track success/failure
4. **Alert user only when problems occur** (multiple consecutive failures)

**Benefits:**
- ✅ Token always stays fresh (never expires)
- ✅ No need to decode JWT or check expiration
- ✅ Doubles as health monitor for server/network
- ✅ Users only alerted when there's an actual problem
- ✅ Seamless, silent renewal in background

**Example Workflow:**
1. User generates token with 30-day expiration
2. App sets 10-minute renewal timer
3. Every 10 minutes, app silently renews token in background
4. If renewal succeeds → continue proxy service normally
5. If renewal fails once → retry, no user alert
6. If renewal fails 3 times in a row → alert user: "Connection to server may be unstable"
7. If renewal fails 5+ times → alert user: "Cannot reach server, please check connection"

**Why This Works Better:**
- Token practically never expires (renewed every 10 mins)
- App doesn't need to parse JWT or know expiration
- Failed renewals indicate real problems (server down, network issue, account suspended)
- Better UX - users only see alerts when action is needed

### Request
```json
{
  "currentToken": "eyJhbGciOiJIUzI1NiJ9.eyJpZCI6ImExYjJjM2Q0...",
  "expirationMinutes": 43200
}
```

### Security Notes
- ✅ No password or sessionId needed
- ✅ Old token is automatically invalidated when new token is generated
- ✅ Perfect for periodic background renewal (recommended pattern)
- ✅ Failed renewals serve as early warning system for service issues
- ⚠️ Current token must be valid and not expired
- ⚠️ If token is completely expired, user must use Alternative 1 (username/password)
- ⚠️ Apps should implement exponential backoff if renewals start failing

### Server Logic
1. Decode and validate currentToken
2. Check if token is expired - if expired, reject (user should use Alternative 1)
3. Extract tokenId from JWT
4. Verify tokenId matches user's current tokenId in database
5. Generate new JWT with new tokenId and new expiration
6. Update user's tokenId field (invalidates the old token)
7. Return new JWT **with expiresAt timestamp**

**Note:** With periodic renewal (every 10-30 mins), tokens should never actually expire in normal operation. Expired tokens indicate the app hasn't been running or network issues.

---

## Self-Connection Loop Prevention

### The Problem
When a user with a valid JWT tries to generate a new token via the SOCKS5 proxy, the request might include an Illiad header with the JWT. This causes:

```
Client → [Illiad Header + JWT] → Server
Server → Decodes JWT → Routes to SOCKS5
SOCKS5 → Detects destination is itself (same IP:port)
SOCKS5 → Potential infinite loop!
```

### The Solution: Alternative 3 (Token Renewal)

**Alternative 3 is specifically designed for this scenario:**

1. User sends token renewal request through HTTP (no Illiad header needed)
2. Server provides a **dedicated HTTP endpoint** for token renewal
3. No SOCKS5 involvement - pure HTTP/REST API
4. Old token invalidated, new token returned
5. No self-connection loop

### Recommended Pattern for Clients

**Periodic Renewal with Health Monitoring:**

```javascript
class TokenManager {
  constructor() {
    this.token = localStorage.getItem('jwt_token');
    this.renewalInterval = 10 * 60 * 1000; // 10 minutes
    this.consecutiveFailures = 0;
    this.maxFailuresBeforeAlert = 3;
    this.renewalTimer = null;
  }
  
  // Start periodic renewal when app launches
  startPeriodicRenewal() {
    this.renewalTimer = setInterval(() => {
      this.renewToken();
    }, this.renewalInterval);
    
    // Also renew immediately on start
    this.renewToken();
  }
  
  // Renew token silently in background
  async renewToken() {
    try {
      const response = await fetch('https://server.com/api/auth/token/generate', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          currentToken: this.token,
          expirationMinutes: 43200  // 30 days (doesn't matter, we renew every 10 mins)
        })
      });
      
      const data = await response.json();
      
      if (data.success) {
        // Success - update token and reset failure counter
        this.token = data.data.token;
        localStorage.setItem('jwt_token', this.token);
        this.consecutiveFailures = 0;
        console.log('Token renewed successfully (silent)');
      } else {
        // Renewal failed
        this.handleRenewalFailure();
      }
    } catch (error) {
      // Network error or server unreachable
      this.handleRenewalFailure();
    }
  }
  
  // Handle renewal failure with progressive alerts
  handleRenewalFailure() {
    this.consecutiveFailures++;
    
    if (this.consecutiveFailures >= 5) {
      // Critical: Multiple failures, likely server down or account issue
      this.showAlert('error', 
        'Cannot connect to server. Please check your internet connection or contact support.');
    } else if (this.consecutiveFailures >= this.maxFailuresBeforeAlert) {
      // Warning: A few failures, might be temporary
      this.showAlert('warning', 
        'Connection to server may be unstable. Monitoring...');
    }
    // else: Single failure, no alert (might be temporary network blip)
  }
  
  // Stop renewal (when app closes)
  stopPeriodicRenewal() {
    if (this.renewalTimer) {
      clearInterval(this.renewalTimer);
    }
  }
  
  // Optional: Manual renewal (if user wants to force it)
  async forceRenewal() {
    await this.renewToken();
    if (this.consecutiveFailures === 0) {
      this.showAlert('success', 'Token renewed successfully!');
    }
  }
  
  showAlert(type, message) {
    // Implement your UI alert mechanism
    console.log(`[${type.toUpperCase()}] ${message}`);
  }
}

// Usage in app
const tokenManager = new TokenManager();
tokenManager.startPeriodicRenewal();

// When app closes
// tokenManager.stopPeriodicRenewal();
```

**Key Features:**
1. **Silent background renewal** every 10 minutes
2. **Health monitoring** - tracks consecutive failures
3. **Progressive alerts** - only alert user when there's a real problem
4. **No JWT parsing** - app treats token as opaque string
5. **Resilient** - tolerates temporary network issues

---

## API Endpoint

```
POST /api/auth/token/generate
Content-Type: application/json
```

### Request Body (Choose ONE alternative)

```typescript
{
  // Alternative 1: Username + Password
  username?: string,
  password?: string,
  
  // Alternative 2: SessionId
  sessionId?: string,
  
  // Alternative 3: Current Token
  currentToken?: string,
  
  // Required for all
  expirationMinutes: number
}
```

### Response (Success)

```json
{
  "success": true,
  "message": "Token generated successfully",
  "data": {
    "token": "eyJhbGciOiJIUzI1NiJ9...",
    "expirationMinutes": 43200,
    "expiresAt": "2025-12-08T10:30:00Z"
  }
}
```

**Note:** The `expiresAt` field is critical for apps because:
- Apps cannot decode/parse JWT tokens
- Apps treat tokens as opaque strings
- Apps need to know when to warn users about expiration
- Apps can display "Token valid until Dec 8, 2025" to users

### Response (Error)

```json
{
  "success": false,
  "message": "Invalid credentials / session / token",
  "data": null
}
```

---

## Security Considerations

### userId Never Exposed
- All alternatives avoid exposing internal userId (MongoDB ObjectId)
- SessionId, username, and tokenId are public identifiers
- userId only used internally for database correlation

### Token Invalidation Strategy
- **Lazy invalidation:** When new token generated, old tokenId replaced
- **No token revocation list needed:** Just check if JWT's tokenId matches user's current tokenId
- **One active token per user:** Ensures security and simplicity

### Best Practices
1. **Apps/Client Software (Initial):** Use username/password (Alternative 1) - no web login needed
2. **Web Browser/Landing Page:** Use sessionId (Alternative 2) - best UX for web users
3. **Automatic Renewal by Apps:** Use currentToken (Alternative 3) - seamless background renewal
4. **Never expose:** userId, password hashes, or internal database IDs

### Use Case Summary
| Scenario | Alternative | When to Use |
|----------|-------------|-------------|
| Mobile/Desktop App (First time) | Alternative 1 | User enters credentials in app, gets token directly |
| Web Browser User | Alternative 2 | User visits landing page, logs in, generates token via web UI |
| App Periodic Renewal | Alternative 3 | App automatically renews token every 10-30 mins in background |

**Important:** Apps should:
1. Implement **periodic renewal** (e.g., every 10 minutes) regardless of token expiration
2. **Silently renew in background** - no user interruption
3. **Track consecutive failures** - use as health monitor
4. **Alert users only when needed** - after multiple consecutive failures
5. This provides seamless service AND early warning of server/network issues

---

## Implementation Notes

The server should implement the token generation endpoint to:

1. Check which alternative is provided (priority: sessionId → currentToken → username/password)
2. Authenticate using the appropriate method
3. Generate new JWT with new UUID tokenId
4. Update user's tokenId field in database
5. Return the new JWT token

All three alternatives result in the **same behavior**: old tokens are invalidated, and a new token is issued.

