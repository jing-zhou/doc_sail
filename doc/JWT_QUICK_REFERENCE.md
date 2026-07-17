# JWT Authentication Quick Reference

## Two Approaches

### 🌐 Approach 1: Web Login (Session-based)
**When to use**: Browser access, initial token generation, user management

```bash
# Login → Session → Generate JWT
POST /api/auth/login
Body: {"username":"alice","password":"pass123"}
Returns: Session cookie

# Generate JWT (requires session cookie)
POST /api/auth/token/generate
Body: {"expirationMinutes":43200}
Returns: {"token":"eyJ..."}
```

### 🔄 Approach 2: JWT Renewal (Token-to-Token)
**When to use**: Automated systems, mobile apps, token refresh

```bash
# Renew JWT (requires current JWT)
POST /api/auth/token/renew
Header: Authorization: Bearer eyJ...
Body: {"expirationMinutes":43200}
Returns: {"token":"NEW_eyJ..."}  ← Old token now invalid!
```

---

## All Endpoints

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| POST | `/api/auth/register` | None | Create account |
| POST | `/api/auth/login` | None | Login (get session) |
| POST | `/api/auth/logout` | Session | Logout |
| POST | `/api/auth/token/generate` | Session | Generate JWT |
| POST | `/api/auth/token/renew` | JWT | Renew JWT |
| POST | `/api/auth/token/revoke` | Session | Revoke all tokens |

---

## Expiration Times

```javascript
{
  "1 day":    1440,
  "3 days":   4320,
  "1 week":   10080,
  "2 weeks":  20160,
  "1 month":  43200,
  "3 months": 129600,
  "6 months": 259200  // Maximum
}
```

---

## Code Examples

### JavaScript/TypeScript

```typescript
// Approach 1: Web login
async function loginAndGetToken(username: string, password: string) {
  // Login
  const loginRes = await fetch('/api/auth/login', {
    method: 'POST',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify({username, password}),
    credentials: 'include' // Important!
  });
  
  // Generate token
  const tokenRes = await fetch('/api/auth/token/generate', {
    method: 'POST',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify({expirationMinutes: 43200}),
    credentials: 'include'
  });
  
  const {data} = await tokenRes.json();
  return data.token; // Save this!
}

// Approach 2: Renew token
async function renewToken(currentJwt: string) {
  const res = await fetch('/api/auth/token/renew', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${currentJwt}`
    },
    body: JSON.stringify({expirationMinutes: 43200})
  });
  
  const {data} = await res.json();
  return data.token; // Old token is now invalid!
}
```

### Python

```python
import requests

# Approach 1: Web login
def login_and_get_token(username, password):
    session = requests.Session()
    
    # Login
    session.post('https://proxy.example.com/api/auth/login', json={
        'username': username,
        'password': password
    })
    
    # Generate token
    resp = session.post('https://proxy.example.com/api/auth/token/generate', json={
        'expirationMinutes': 43200
    })
    
    return resp.json()['data']['token']

# Approach 2: Renew token
def renew_token(current_jwt):
    resp = requests.post('https://proxy.example.com/api/auth/token/renew',
        headers={'Authorization': f'Bearer {current_jwt}'},
        json={'expirationMinutes': 43200}
    )
    
    return resp.json()['data']['token']  # Old token is now invalid!
```

### cURL

```bash
# Approach 1: Web login
curl -X POST https://proxy.example.com/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"pass123"}' \
  -c cookies.txt

curl -X POST https://proxy.example.com/api/auth/token/generate \
  -H "Content-Type: application/json" \
  -d '{"expirationMinutes":43200}' \
  -b cookies.txt

# Approach 2: Renew token
curl -X POST https://proxy.example.com/api/auth/token/renew \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJ..." \
  -d '{"expirationMinutes":43200}'
```

---

## Security Notes

✅ **DO**:
- Save JWT securely (encrypted storage)
- Renew tokens before expiration
- Use HTTPS in production
- Use Approach 2 for automation (no credentials in code)

❌ **DON'T**:
- Store JWT in server database
- Expose JWT in logs or URLs
- Hard-code credentials
- Share JWT between users

---

## Token Invalidation

When you generate a new token:
- ✅ New token is created
- ❌ **ALL old tokens become invalid immediately**
- ⚠️ Even if old tokens haven't expired!

This is a feature, not a bug! It prevents:
- Multiple active tokens per user
- Leaked tokens remaining valid
- Token accumulation over time

---

## Error Handling

```javascript
async function handleTokenRenewal(jwt) {
  try {
    return await renewToken(jwt);
  } catch (error) {
    if (error.status === 401) {
      // Token invalid/expired → Login again
      return await loginAndGetToken(username, password);
    }
    throw error;
  }
}
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    CLIENT APPLICATION                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Approach 1              │           Approach 2             │
│  (Web Login)             │          (JWT Renewal)           │
│                          │                                  │
│  Username + Password ────┼──────→ Current JWT              │
│         ↓                │              ↓                   │
│    POST /login           │    POST /token/renew            │
│         ↓                │              ↓                   │
│   Session Cookie ────────┼──────→ Validate JWT             │
│         ↓                │              ↓                   │
│  POST /token/generate ───┼──────→ Generate New JWT         │
│         ↓                │              ↓                   │
│     New JWT  ←───────────┴──────── New JWT                 │
│                                                              │
│         (Old JWT is now invalid)                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                      SERVER (Netty)                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  RerouteHandler (HTTPS Disguise)                            │
│  ├─ handleLogin() ──────────→ SessionService                │
│  ├─ handleTokenGenerate() ──→ UserService.generateToken()   │
│  └─ handleTokenRenew() ──────→ UserService.validateToken()  │
│                                  + generateToken()           │
│                                                              │
│  HeaderDecoder (Illiad Protocol)                            │
│  └─ Validates JWT from Illiad header                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    DATABASE (MongoDB)                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  users: {                      sessions: {                  │
│    _id,                           _id (sessionId),          │
│    username,                      userId,                   │
│    password (hashed),             expiresAt                 │
│    currentTokenId ← Key!       }                            │
│  }                                                           │
│                                                              │
│  tokens: {                                                   │
│    tokenId,                    ⚠️ JWT string NOT stored!    │
│    userId,                                                   │
│    expiresAt                                                │
│  }                                                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Files Modified/Created

**New Files**:
- `model/Session.java` - Session model
- `repository/SessionRepository.java` - Session DB access
- `service/SessionService.java` - Session management
- `JWT_AUTHENTICATION_GUIDE.md` - User guide
- `JWT_AUTHENTICATION_IMPLEMENTATION.md` - Implementation details
- `JWT_QUICK_REFERENCE.md` - This file

**Modified Files**:
- `RerouteHandler.java` - Added session auth & JWT renewal
- `HeaderDecoder.java` - Pass SessionService
- `ParamBus.java` - Added SessionService dependency

---

## Testing Checklist

- [ ] Register new user
- [ ] Login with valid credentials
- [ ] Login with invalid credentials (should fail)
- [ ] Generate JWT with session
- [ ] Generate JWT without session (should fail)
- [ ] Renew JWT with valid token
- [ ] Renew JWT with invalid token (should fail)
- [ ] Verify old JWT is invalid after renewal
- [ ] Logout
- [ ] Generate JWT after logout (should fail)
- [ ] Use JWT in Illiad header (proxy connection)

---

## Production Deployment

1. **Enable HTTPS**: Use Let's Encrypt certificates
2. **MongoDB**: Ensure MongoDB is running and accessible
3. **Environment**: Set production environment variables
4. **Monitoring**: Track token generation/renewal rates
5. **Backup**: Regular MongoDB backups
6. **Rate Limiting**: Implement on authentication endpoints

---

For detailed information, see:
- **`JWT_AUTHENTICATION_GUIDE.md`** - Complete user guide
- **`JWT_AUTHENTICATION_IMPLEMENTATION.md`** - Implementation details

