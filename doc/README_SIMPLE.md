# Enhanced Trojan Proxy - Simple Explanation

## What This Is

A **SOCKS5 proxy disguised as an HTTPS web server**.

## How It Works

```
                    Client connects to port 2080
                              ↓
                    TLS/SSL handshake completes
                              ↓
                    Client sends first bytes after TLS
                              ↓
                    ┌─────────┴─────────────────┐
                    │                           │
         illiad Header + SOCKS5         Standard HTTPS Request
                    │                           │
         (Header verified ✓)         (No valid header / failed)
                    ▼                           ▼
          ┌──────────────────────┐  ┌──────────────────────┐
          │ Process SOCKS5       │  │ Route to HTTPS       │
          │ CONNECT request      │  │ Handler              │
          │         ↓            │  │                      │
          │ Connect to           │  │ Serves:              │
          │ destination          │  │ - Welcome page       │
          │         ↓            │  │ - Let's Encrypt      │
          │ Relay traffic        │  │ - REST API           │
          │ Client ↔ Destination │  │                      │
          └──────────────────────┘  └──────────────────────┘
              Proxy Success            Looks like web server
```

## Complete Request Flow

### True illiad Client (Authorized Proxy User)
```
1. Client establishes TCP connection to server:2080
2. Client and server complete TLS handshake
3. Client sends: [illiad header] + [SOCKS5 CONNECT command]
   Example: [encrypted header with signature] + [0x05 0x01 0x00 ...]
4. Server verifies illiad header signature
5. ✅ If valid:
   - Server processes SOCKS5 CONNECT request
   - Server establishes TCP connection to destination
   - Server relays traffic: Client ↔ Server ↔ Destination
   - All traffic encrypted in TLS tunnel
6. ❌ If invalid:
   - Traffic rerouted to HTTPS handler
   - Server responds as normal HTTPS server would
   - Returns HTTP responses (Welcome page, APIs, etc.)
```

### Regular HTTPS Client (Browser, Probe, Let's Encrypt)
```
1. Client establishes TCP connection to server:2080
2. Client and server complete TLS handshake
3. Client sends: Standard HTTP request (GET / HTTP/1.1)
   No illiad header - just normal HTTP
4. Server examines first bytes
5. Server sees: Not a valid illiad header
6. Server routes to HTTPS handler (RerouteHandler)
7. Server responds: HTTP/1.1 200 OK + content
   - "/" → "Welcome" page
   - "/.well-known/acme-challenge/*" → ACME token
   - "/api/letsencrypt/*" → JSON API responses
```

## The Critical Difference

### illiad Protocol Structure
```
After TLS:
┌────────────────┬──────────────────────┐
│ illiad Header  │ SOCKS5 Request       │
│ (auth/crypto)  │ (CONNECT example.com)│
└────────────────┴──────────────────────┘
     ↑                    ↑
  Verified first    Then processed

If header valid → SOCKS5 proxy activated
If header invalid → HTTPS server activated
```

### Standard HTTPS Request
```
After TLS:
┌─────────────────────────────────┐
│ HTTP Request                    │
│ GET / HTTP/1.1                  │
│ Host: proxy.example.com         │
└─────────────────────────────────┘
          ↑
   No illiad header → HTTPS handler
```

## The Disguise Mechanism

### Header Detection (HeaderDecoder)
```java
After SSL/TLS handshake:
1. Read first bytes from client
2. Parse potential illiad header:
   - Check length field (2 bytes)
   - Check crypto type (1 byte)  
   - Extract signature
   - Find CRLF terminator
3. Verify signature against secret
   ├─ Valid → Process remaining bytes as SOCKS5
   └─ Invalid → Route entire stream to HTTP handler
```

### Why This Works
- ✅ **Stealth**: Outsiders only see HTTPS traffic
- ✅ **Legitimacy**: Has real SSL certificates (Let's Encrypt)
- ✅ **Functionality**: Serves actual HTTP content
- ✅ **Security**: Hidden SOCKS5 only accessible with secret
- ✅ **Single Port**: Everything on port 2080

## Example Scenarios

### Scenario 1: Random Person Visits
```
Browser connects to: https://proxy.example.com/
  ↓ TLS handshake completes
  ↓ Browser sends: GET / HTTP/1.1
Server examines first bytes: "GET / HTTP/1.1..." (not illiad header)
Server routes to: HTTP handler (RerouteHandler)
Server responds: HTTP/1.1 200 OK\r\n...\r\nWelcome
Observer sees: "Just a regular HTTPS website"
```

### Scenario 2: Let's Encrypt Validates Domain
```
Let's Encrypt connects to: https://proxy.example.com/.well-known/acme-challenge/token123
  ↓ TLS handshake completes
  ↓ Sends: GET /.well-known/acme-challenge/token123 HTTP/1.1
Server examines: Standard HTTP GET (not illiad header)
Server routes to: HTTP handler → ACME challenge endpoint
Server responds: HTTP/1.1 200 OK\r\n...\r\n[challenge token]
Let's Encrypt: Validates domain ownership ✓
Observer sees: "Legitimate certificate validation"
```

### Scenario 3: Authorized User Proxies (illiad Client)
```
Client connects to: proxy.example.com:2080
  ↓ TLS handshake completes
  ↓ Client sends: [illiad header (length+crypto+signature+CRLF)] + 
                  [SOCKS5 CONNECT request: 0x05 0x01 0x00 0x03 0x0B example.com 0x01BB]
Server examines first bytes:
  ↓ Reads length field (2 bytes)
  ↓ Reads crypto type (1 byte)
  ↓ Extracts signature
  ↓ Finds CRLF terminator
  ↓ Verifies signature against secret ✓
Server processing:
  ↓ Header valid → Advances buffer past header
  ↓ Processes SOCKS5 CONNECT request
  ↓ Establishes connection to example.com:443
  ↓ Sends SOCKS5 response: 0x05 0x00 0x00 ...
  ↓ Starts relaying: Client ↔ Server ↔ example.com
Traffic flow: All encrypted in TLS tunnel
Observer sees: "Encrypted HTTPS traffic" (cannot determine it's a proxy)
```

### Scenario 4: Invalid Header Attempt
```
Malicious probe tries custom protocol:
  ↓ TLS handshake completes
  ↓ Sends: Random bytes or malformed header
Server examines:
  ↓ Length/crypto checks fail, OR
  ↓ Signature verification fails, OR  
  ↓ CRLF not found in expected range
Server routes to: HTTP handler (fallback)
Server responds: HTTP/1.1 200 OK\r\n...\r\nWelcome
Probe result: "Looks like a web server, nothing special"
```

## Key Innovation

**Standard Trojan**: 
- Client sends header
- Server verifies
- If invalid → close connection (suspicious!)

**This Implementation**:
- Client sends header (+ SOCKS5 request)
- Server verifies
- If valid → process SOCKS5, relay traffic
- If invalid → treat as HTTP, serve content (looks normal!)

**Result**: Failed authentication looks exactly like accessing a normal HTTPS website

## Traffic From Outside Perspective

### Without Secret (Casual Observer)
```
Port scan → Port 2080 open
TLS probe → Valid certificate, HTTPS
HTTP GET  → Valid HTML response
Conclusion: "It's a web server"
```

### With Secret (Authorized User)
```
Connect → TLS handshake
Send header → Custom encrypted authentication
SOCKS5 → Full proxy functionality
Encrypted traffic → All proxied connections
Conclusion: "Private proxy access"
```

## Architecture Layers

### Layer 1: Network
```
Internet ──→ Port 2080 (SSL/TLS)
```

### Layer 2: Protocol Detection (HeaderDecoder)
```
Inspect first bytes:
├─ Match custom format + valid signature → SOCKS5
└─ Anything else → HTTPS
```

### Layer 3A: SOCKS5 Mode
```
Pipeline: V5Encoder → V5Decoder → V5Handler
Function: Full SOCKS5 proxy service
```

### Layer 3B: HTTPS Mode  
```
Pipeline: HttpCodec → HttpAggregator → RerouteHandler
Function: Web server + Let's Encrypt + API
```

## Configuration

### Enable Both Modes
```properties
# Port for both SOCKS5 and HTTPS
params.local-port=2080

# Secret for SOCKS5 access
params.secret=your-strong-password

# SSL certificate (automatic via Let's Encrypt)
letsencrypt.enabled=true
letsencrypt.domains=proxy.example.com
```

## Benefits

1. **Undetectable**: Looks like normal HTTPS
2. **Legitimate**: Real SSL certificates
3. **Functional**: Actually serves web content
4. **Secure**: Authentication required for proxy
5. **Efficient**: Single port for everything
6. **Smart**: Auto-renewing certificates
7. **Flexible**: REST API for management

## Use Cases

### Public Face (HTTPS)
- Certificate validation (Let's Encrypt)
- Administration API
- Monitoring endpoints
- Decoy website

### Private Face (SOCKS5)
- Personal proxy server
- Bypass censorship
- Privacy protection
- Access restricted content

## Security Model

### For HTTPS Mode
- Anyone can access
- Serves public content
- No authentication needed
- Looks like normal website

### For SOCKS5 Mode
- Requires custom header
- Requires valid signature
- Requires secret password
- Hidden from outsiders

## Summary

**This is a chameleon server:**
- Acts like HTTPS web server to everyone
- Acts like SOCKS5 proxy to authorized users
- Uses same port, same IP, same certificate
- Completely indistinguishable to observers

**The disguise is perfect because it's functional** - it actually IS an HTTPS server, and it actually IS a SOCKS5 proxy. Which face you see depends on how you authenticate.

---

**Result**: A SOCKS5 proxy that's invisible to detection, protected by legitimate HTTPS, with automatic certificate management. 🎭
