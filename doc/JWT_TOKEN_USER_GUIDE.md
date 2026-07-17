# JWT Token User Workflow Guide

## Complete User Journey: From Registration to Proxy Usage

This guide shows how users can register, login, generate a JWT token, and use it with the proxy client.

---

## Step 1: Register an Account

### Using cURL
```bash
curl -k -X POST "https://your-domain.com/api/auth/register" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "alice",
    "password": "SecurePassword123!",
    "email": "alice@example.com"
  }'
```

### Response (200 OK)
```json
{
  "success": true,
  "message": "User registered successfully",
  "data": {
    "username": "alice"
  }
}
```


---

## Step 2: Login to Get Session ID

After registration, login with your credentials to obtain a session ID.

### Using cURL
```bash
curl -k -X POST "https://your-domain.com/api/auth/login" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "alice",
    "password": "SecurePassword123!"
  }'
```

### Response (200 OK)
```json
{
  "success": true,
  "message": "Login successful",
  "data": {
    "username": "alice",
    "sessionId": "a1b2c3d4-e5f6-7890-1234-567890abcdef"
  }
}
```

**Save your sessionId:** The session ID is valid for a limited time (e.g., 1 hour) and allows you to generate tokens without re-entering your password.

---

## Step 3: Generate JWT Token

You have **3 alternatives** to generate a JWT token:

### Alternative 1: Using SessionId (Recommended after login)

This is the most convenient method after login - no need to re-enter password.

```bash
curl -k -X POST "https://your-domain.com/api/auth/token/generate" \
  -H "Content-Type: application/json" \
  -d '{
    "sessionId": "a1b2c3d4-e5f6-7890-1234-567890abcdef",
    "expirationMinutes": 43200
  }'
```

### Alternative 2: Using Username + Password (Direct)

Generate a token directly without login session.

```bash
curl -k -X POST "https://your-domain.com/api/auth/token/generate" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "alice",
    "password": "SecurePassword123!",
    "expirationMinutes": 43200
  }'
```

### Alternative 3: Using Existing Valid Token (Renewal)

Renew/refresh your token with an existing valid token. This invalidates the old token and creates a new one.

```bash
curl -k -X POST "https://your-domain.com/api/auth/token/generate" \
  -H "Content-Type: application/json" \
  -d '{
    "currentToken": "eyJhbGciOiJIUzI1NiJ9...",
    "expirationMinutes": 43200
  }'
```

**Note:** `expirationMinutes: 43200` = 30 days

### Response (200 OK) - Same for all alternatives
```json
{
  "success": true,
  "message": "Token generated successfully",
  "data": {
    "token": "eyJhbGciOiJIUzI1NiJ9.eyJpZCI6ImExYjJjM2Q0LWU1ZjYtNzg5MC1hYmNkLWVmMTIzNDU2Nzg5MCIsImV4cGlyZXNBdCI6MTczNTg2MjQwMDAwMH0.Xh7j9K2mN4pQ6rS8tU0vW1xY2zA3bC4dE5fF6gG7hH8",
    "expirationMinutes": "43200"
  }
}
```

---

## Step 4: Copy the JWT Token

The JWT token is a **plain text string** that looks like this:

```
eyJhbGciOiJIUzI1NiJ9.eyJpZCI6ImExYjJjM2Q0LWU1ZjYtNzg5MC1hYmNkLWVmMTIzNDU2Nzg5MCIsImV4cGlyZXNBdCI6MTczNTg2MjQwMDAwMH0.Xh7j9K2mN4pQ6rS8tU0vW1xY2zA3bC4dE5fF6gG7hH8
```

### Three Parts (separated by dots):
1. **Header:** `eyJhbGciOiJIUzI1NiJ9`
2. **Payload:** `eyJpZCI6ImExYjJjM2Q0LWU1ZjYtNzg5MC1hYmNkLWVmMTIzNDU2Nzg5MCIsImV4cGlyZXNBdCI6MTczNTg2MjQwMDAwMH0`
3. **Signature:** `Xh7j9K2mN4pQ6rS8tU0vW1xY2zA3bC4dE5fF6gG7hH8`

### How to Save the Token

#### Option 1: Copy to Text File
```bash
# Linux/Mac
echo "eyJhbGciOiJIUzI1NiJ9.eyJpZCI6ImExYjJjM2Q0LWU1ZjYtNzg5MC1hYmNkLWVmMTIzNDU2Nzg5MCIsImV4cGlyZXNBdCI6MTczNTg2MjQwMDAwMH0.Xh7j9K2mN4pQ6rS8tU0vW1xY2zA3bC4dE5fF6gG7hH8" > my_jwt_token.txt

# Windows
echo eyJhbGciOiJIUzI1NiJ9.eyJpZCI6ImExYjJjM2Q0LWU1ZjYtNzg5MC1hYmNkLWVmMTIzNDU2Nzg5MCIsImV4cGlyZXNBdCI6MTczNTg2MjQwMDAwMH0.Xh7j9K2mN4pQ6rS8tU0vW1xY2zA3bC4dE5fF6gG7hH8 > my_jwt_token.txt
```

#### Option 2: Copy to Clipboard
```bash
# Linux (with xclip)
echo "eyJhbGciOiJIUzI1NiJ9..." | xclip -selection clipboard

# Mac
echo "eyJhbGciOiJIUzI1NiJ9..." | pbcopy

# Windows (PowerShell)
echo "eyJhbGciOiJIUzI1NiJ9..." | Set-Clipboard
```

#### Option 3: Save in Configuration File
Most proxy clients have a configuration file where you can paste the token.

**Example: config.json**
```json
{
  "server": "your-domain.com",
  "port": 443,
  "auth": {
    "type": "jwt",
    "token": "eyJhbGciOiJIUzI1NiJ9.eyJpZCI6ImExYjJjM2Q0LWU1ZjYtNzg5MC1hYmNkLWVmMTIzNDU2Nzg5MCIsImV4cGlyZXNBdCI6MTczNTg2MjQwMDAwMH0.Xh7j9K2mN4pQ6rS8tU0vW1xY2zA3bC4dE5fF6gG7hH8"
  }
}
```

#### Option 4: Manual Input
The token is plain text, so you can:
- Type it character by character (not recommended - easy to make mistakes)
- Copy-paste from the API response
- Store in password manager (recommended for security)

---

## Step 5: Use the Token with Proxy Client

### Python Client Example

```python
import socket
import struct

# Your JWT token (plain text string)
JWT_TOKEN = "eyJhbGciOiJIUzI1NiJ9.eyJpZCI6ImExYjJjM2Q0LWU1ZjYtNzg5MC1hYmNkLWVmMTIzNDU2Nzg5MCIsImV4cGlyZXNBdCI6MTczNTg2MjQwMDAwMH0.Xh7j9K2mN4pQ6rS8tU0vW1xY2zA3bC4dE5fF6gG7hH8"

# Server details
SERVER_HOST = "your-domain.com"
SERVER_PORT = 443

# Build Illiad JWT header
def build_illiad_jwt_header(jwt_token):
    crypto_type = 0x07  # JWT
    token_bytes = jwt_token.encode('utf-8')
    
    # Length = 2 (length field) + 1 (crypto) + len(token)
    length = 2 + 1 + len(token_bytes)
    
    # CRLF terminator must immediately follow token
    crlf = b"\r\n"
    
    # Build complete header
    header = struct.pack('>H', length)  # 2 bytes, big-endian
    header += bytes([crypto_type])       # 1 byte
    header += token_bytes                # Variable length
    header += crlf                       # Terminator
    
    return header

# Connect to proxy
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# Wrap with TLS
import ssl
context = ssl.create_default_context()
context.check_hostname = False
context.verify_mode = ssl.CERT_NONE  # For self-signed certs in dev

tls_sock = context.wrap_socket(sock, server_hostname=SERVER_HOST)
tls_sock.connect((SERVER_HOST, SERVER_PORT))

# Send Illiad header with JWT
illiad_header = build_illiad_jwt_header(JWT_TOKEN)
tls_sock.sendall(illiad_header)

# Now send SOCKS5 handshake
# ... (standard SOCKS5 protocol)

print("Connected to proxy with JWT authentication!")
```

### Using with Browser/Application

Many SOCKS5 clients allow you to configure authentication. You can:

1. **Create a configuration file** with your JWT token
2. **Paste the token** into the client's settings UI
3. **Load token from file** if the client supports it

---

## Token Properties

### Plain Text Format
✅ The JWT is a **plain text ASCII string**  
✅ Contains only safe characters: `A-Z`, `a-z`, `0-9`, `-`, `_`, `.`  
✅ No special encoding needed  
✅ Can be copied, pasted, or typed  
✅ Length: ~150-200 characters (varies based on claims)

### Token Structure
```
[Header].[Payload].[Signature]
```

Each part is Base64URL encoded, which means:
- Only alphanumeric characters plus `-` and `_`
- No spaces, no newlines, no special characters
- Safe for URLs, JSON, configuration files

### Token Example Breakdown
```
Original Token:
eyJhbGciOiJIUzI1NiJ9.eyJpZCI6ImExYjJjM2Q0LWU1ZjYtNzg5MC1hYmNkLWVmMTIzNDU2Nzg5MCIsImV4cGlyZXNBdCI6MTczNTg2MjQwMDAwMH0.Xh7j9K2mN4pQ6rS8tU0vW1xY2zA3bC4dE5fF6gG7hH8

Decoded Header:
{"alg":"HS256"}

Decoded Payload:
{"id":"a1b2c3d4-e5f6-7890-abcd-ef1234567890","expiresAt":1735862400000}
       ↑                                            ↑
  Random UUID - reveals NOTHING              Expiration timestamp
  about the user. Only correlates            (milliseconds since 1970)
  to user in server database.

Signature: (binary, verified by server)
```

**Privacy Protection:**
- The `id` is a **random UUID**, not your username or any personal data
- Only the server's database can correlate this UUID to your account
- Each new token gets a completely different random UUID
- Even if someone intercepts the token, they cannot identify you from it

---

## Using Web Browser

### Save Token via HTML Page

If you create a simple HTML page to manage tokens, users can:

```html
<!DOCTYPE html>
<html>
<head>
    <title>JWT Token Generator</title>
</head>
<body>
    <h1>Generate Your JWT Token</h1>
    
    <form id="tokenForm">
        <label>Session ID:</label>
        <input type="text" id="sessionId" required><br><br>
        
        <label>Expiration (days):</label>
        <select id="expiration">
            <option value="1440">1 day</option>
            <option value="10080">1 week</option>
            <option value="43200" selected>30 days</option>
            <option value="129600">90 days</option>
        </select><br><br>
        
        <button type="submit">Generate Token</button>
    </form>
    
    <div id="result" style="display:none;">
        <h2>Your JWT Token</h2>
        <textarea id="tokenDisplay" rows="5" cols="80" readonly></textarea><br>
        <button onclick="copyToken()">Copy to Clipboard</button>
        <button onclick="downloadToken()">Download as Text File</button>
    </div>
    
    <script>
        document.getElementById('tokenForm').addEventListener('submit', async (e) => {
            e.preventDefault();
            
            const sessionId = document.getElementById('sessionId').value;
            const expirationMinutes = document.getElementById('expiration').value;
            
            try {
                const response = await fetch('/api/auth/token/generate', {
                    method: 'POST',
                    headers: {'Content-Type': 'application/json'},
                    body: JSON.stringify({
                        sessionId: sessionId,
                        expirationMinutes: parseInt(expirationMinutes)
                    })
                });
                
                const data = await response.json();
                
                if (data.success) {
                    document.getElementById('tokenDisplay').value = data.data.token;
                    document.getElementById('result').style.display = 'block';
                } else {
                    alert('Error: ' + data.message);
                }
            } catch (error) {
                alert('Failed to generate token: ' + error);
            }
        });
        
        function copyToken() {
            const tokenText = document.getElementById('tokenDisplay');
            tokenText.select();
            document.execCommand('copy');
            alert('Token copied to clipboard!');
        }
        
        function downloadToken() {
            const token = document.getElementById('tokenDisplay').value;
            const blob = new Blob([token], {type: 'text/plain'});
            const url = URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = url;
            a.download = 'jwt_token.txt';
            a.click();
            URL.revokeObjectURL(url);
        }
    </script>
</body>
</html>
```

This page allows users to:
1. ✅ **Generate token** via form
2. ✅ **View token** in plain text textarea
3. ✅ **Copy to clipboard** with one click
4. ✅ **Download as .txt file** with one click

---

## Token Expiration Options

When generating a token, users can choose how long it should be valid:

| Duration | Minutes | Use Case |
|----------|---------|----------|
| 1 day | 1,440 | Testing/temporary access |
| 5 days | 7,200 | Short-term project |
| 1 week | 10,080 | Weekly rotation |
| 2 weeks | 20,160 | Bi-weekly rotation |
| 1 month | 43,200 | Monthly rotation (recommended) |
| 3 months | 129,600 | Maximum allowed |

**Note:** 3 months is the maximum due to Let's Encrypt certificate renewal requirements.

---

## Security Best Practices

### For Users

1. **Store Securely**
   - Use password manager (1Password, LastPass, Bitwarden)
   - Don't email the token to yourself
   - Don't post in public forums or chat

2. **Handle Like a Password**
   - Anyone with your token can use your proxy access
   - Never share with others
   - Revoke if compromised

3. **Rotate Regularly**
   - Generate new token every 30 days
   - Old token automatically invalidated
   - Update client configuration with new token

4. **Backup**
   - Save token in secure location
   - If lost, generate new one (old becomes invalid)

### Token in Transit

- ✅ Token sent via HTTPS (encrypted in transit)
- ✅ Token stored in MongoDB (server-side)
- ✅ Token includes HMAC signature (tamper-proof)
- ✅ Token validates expiration (time-limited)

---

## Troubleshooting

### Token Not Working?

1. **Check Expiration**
   - Use https://jwt.io to decode token
   - Check `expiresAt` timestamp
   - Generate new token if expired

2. **Check Format**
   - Token must be complete string (all 3 parts)
   - No extra spaces or newlines
   - Must include both dots (`.`)

3. **Token Revoked?**
   - Generating new token invalidates old one
   - Only one active token per user
   - Must use most recent token

4. **User Account Active?**
   - Account must be active in database
   - Contact admin if account disabled

---

## Summary

The JWT token is a **plain text string** that:

✅ Can be **copied** from API response  
✅ Can be **pasted** into configuration files  
✅ Can be **typed** manually (though not recommended)  
✅ Can be **downloaded** as text file  
✅ Can be **stored** in password managers  
✅ Is **portable** across all platforms  
✅ Contains **no binary data** (only ASCII characters)  

Users have complete flexibility in how they obtain, store, and use their JWT tokens. The token is designed to be user-friendly while maintaining security through cryptographic signatures and server-side validation.
