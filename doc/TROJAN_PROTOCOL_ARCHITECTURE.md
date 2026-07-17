# Enhanced Trojan Proxy Protocol - Architecture

## Overview

This project implements an **enhanced Trojan-like proxy protocol** with support for **multiple encryption/decryption headers**. It disguises itself as a legitimate HTTPS web server while providing SOCKS5 proxy services to authorized clients.

## Key Enhancement Over Standard Trojan

**Standard Trojan:** Single SHA224 password hash header  
**This Implementation:** Multiple encryption algorithms with variable-length signatures

Supported encryption types: `0x10, 0x20, 0x30, 0x40, 0x50, 0x60, 0x70, 0x80, 0x90, 0xA0, 0xB0, 0xC0, 0xD0, 0xE0, 0xF0, 0x01, 0x11, 0x21, 0x31, 0x41, 0x51, 0x61, 0x71, 0x81, 0x91, 0xA1, 0xB1, 0xC1, 0xD1, 0xE1`

---

## Architecture Diagram

```
                        Internet
                           ↓
                    ┌──────────────┐
                    │  Port 2080   │
                    │   SSL/TLS    │
                    └──────┬───────┘
                           ↓
                  ┌────────────────────┐
                  │   HeaderDecoder    │
                  │  (Protocol Switch) │
                  └────────┬───────────┘
                           │
              ┌────────────┴────────────┐
              │                         │
        Valid Header              Invalid Header
              │                         │
              ▼                         ▼
    ┌─────────────────┐      ┌──────────────────────┐
    │  SOCKS5 Proxy   │      │   RerouteHandler     │
    │   Pipeline      │      │  (HTTPS Disguise)    │
    └─────────────────┘      └──────────┬───────────┘
                                        │
                             ┌──────────┴──────────┐
                             │                     │
                             ▼                     ▼
                    ┌─────────────────┐   ┌──────────────────┐
                    │ Welcome Page    │   │ Let's Encrypt    │
                    │ "Welcome"       │   │ API & ACME       │
                    └─────────────────┘   └──────────────────┘
```

---

## Protocol Flow

### 1. Client Connection
```
Client → SSL/TLS Handshake → Server (Port 2080)
```

### 2. Header Detection
```
Client sends first bytes after TLS:
┌────────┬──────────┬────────────┬────────┬──────┐
│ Length │  Crypto  │ Signature  │ Offset │ CRLF │
│ (2B)   │  Type(1B)│ (Variable) │ (Var)  │ (2B) │
└────────┴──────────┴────────────┴────────┴──────┘
```

**HeaderDecoder Logic:**
1. Read first 3 bytes: `length (2) + cryptoType (1)`
2. Validate `cryptoType` against whitelist
3. Verify signature length matches expected
4. Search for CRLF terminator
5. Verify secret/password
6. **If valid** → SOCKS5 proxy pipeline
7. **If invalid** → HTTP server pipeline (disguise)

### 3A. Valid Header → SOCKS5 Proxy
```
HeaderDecoder
    ↓
V5ServerEncoder
    ↓
V5CmdReqDecoder
    ↓
V5CommandHandler
    ↓
(CONNECT/BIND/UDP_ASSOCIATE handling)
```

### 3B. Invalid Header → HTTPS Disguise
```
HeaderDecoder
    ↓
HttpServerCodec
    ↓
HttpObjectAggregator
    ↓
RerouteHandler
    ↓
┌─────────────────────────────────────────┐
│ Route based on URI:                     │
│ - /                  → Welcome page     │
│ - /.well-known/acme-challenge/* → ACME │
│ - /api/letsencrypt/* → Certificate API │
└─────────────────────────────────────────┘
```

---

## Custom Header Format

### Fixed-Length Signature
```
┌───────────┬──────────┬─────────────────┬─────────┬──────┐
│  Length   │  Crypto  │   Signature     │ Offset  │ CRLF │
│  (2 bytes)│  (1 byte)│   (N bytes)     │ (1+ B)  │ (2B) │
└───────────┴──────────┴─────────────────┴─────────┴──────┘
                                                           
Length field = total length including CRLF
Example: 0x0025 (37 bytes total)
```

### Variable-Length Signature
```
┌───────────┬──────────┬─────────────────┬─────────┬──────┐
│  Length   │  Crypto  │   Signature     │ Offset  │ CRLF │
│  (2 bytes)│  (1 byte)│   (N bytes)     │ (Var)   │ (2B) │
└───────────┴──────────┴─────────────────┴─────────┴──────┘
                                                           
Length field = signature length only
CRLF must be found by scanning after signature
```

---

## Disguise Mechanism

### Purpose
Make the proxy indistinguishable from a normal HTTPS website to:
- Bypass deep packet inspection (DPI)
- Avoid detection by firewalls
- Look like legitimate web traffic

### Implementation
1. **SSL/TLS Encryption**: All traffic encrypted (just like HTTPS)
2. **Fallback to HTTP Server**: Invalid headers → serve web pages
3. **Let's Encrypt Integration**: Legitimate SSL certificates
4. **Standard HTTP Responses**: Proper HTTP/1.1 responses

### Endpoints Served by Disguise

| Endpoint | Response | Purpose |
|----------|----------|---------|
| `GET /` | "Welcome" | Default landing page |
| `GET /.well-known/acme-challenge/{token}` | Challenge token | Let's Encrypt validation |
| `GET /api/letsencrypt/status` | JSON status | Certificate management |
| `GET /api/letsencrypt/certificate` | JSON cert info | Certificate info |
| `POST /api/letsencrypt/renew` | JSON response | Manual renewal |

---

## Security Features

### 1. Multiple Encryption Algorithms
Supports 30 different encryption types, making it harder to fingerprint

### 2. Variable Signature Lengths
Both fixed and variable-length signatures supported

### 3. Secret Verification
Cryptographic verification of client credentials

### 4. Traffic Obfuscation
- Encrypted within TLS tunnel
- Custom protocol not matching known signatures
- Falls back to legitimate HTTP traffic

### 5. Legitimate HTTPS Appearance
- Real SSL/TLS certificates (via Let's Encrypt)
- Standard HTTP responses
- Looks like a normal web server to observers

---

## Let's Encrypt Integration

### Why Integrated into Main HTTPS Port?

**NOT separate port 80 HTTP server!**

Reasons:
1. **Better Disguise**: Everything on HTTPS (port 2080)
2. **No Additional Ports**: Simpler firewall configuration
3. **Encrypted Challenges**: Even ACME challenges go through TLS
4. **Trojan Protocol**: Maintains the Trojan principle of single port

### How ACME Challenges Work

```
1. Let's Encrypt makes request to:
   https://yourdomain.com/.well-known/acme-challenge/token123

2. Request arrives at port 2080 (SSL/TLS)

3. TLS handshake completes (using current certificate)

4. Client sends HTTP GET request (not valid illiad header)

5. HeaderDecoder detects invalid header → switches to HTTP

6. RerouteHandler receives HttpRequest

7. Routes /.well-known/acme-challenge/* to handleAcmeChallenge()

8. Returns challenge token

9. Let's Encrypt validates → issues new certificate
```

**Note**: Initial setup may require:
- Temporary self-signed certificate, OR
- Port 80 forwarding during first certificate acquisition, OR
- DNS-01 challenge instead of HTTP-01

---

## Comparison with Standard Trojan

| Feature | Standard Trojan | This Implementation |
|---------|----------------|---------------------|
| **Protocol** | SHA224 hash | Multiple encryption types |
| **Header** | Fixed SHA224 (28 bytes) | Variable (28+ bytes) |
| **Disguise** | TLS traffic | TLS + HTTP server |
| **Fallback** | Close connection | Serve HTTP content |
| **Certificate** | Manual | Automatic (Let's Encrypt) |
| **Port** | Typically 443 | Configurable (2080) |
| **Signature Types** | 1 (SHA224) | 30+ algorithms |
| **API** | None | REST API for management |

---

## Configuration

### Main Server
```properties
# Port for the proxy (both SOCKS5 and HTTPS disguise)
params.local-port=2080

# SSL Certificate (for initial HTTPS)
server.ssl.key-store=classpath:server.p12
server.ssl.key-store-password=changeit
```

### Let's Encrypt
```properties
# Enable automatic certificate management
letsencrypt.enabled=true
letsencrypt.email=admin@example.com
letsencrypt.domains=proxy.example.com

# Staging for testing
letsencrypt.staging=true

# Integrated into main HTTPS port (not standalone port 80)
letsencrypt.use-standalone-http-server=false
```

### Security
```properties
# Secret for header verification
params.secret=your-secret-password
```

---

## Client Configuration

Clients must:
1. Connect to port 2080 with TLS
2. Send valid illiad header after TLS handshake
3. Use SOCKS5 protocol after header accepted

**Header must include:**
- Valid length field
- Valid crypto type (from whitelist)
- Correct signature for the secret
- CRLF terminator (for variable-length JWT the CRLF must immediately follow the token)

---

## Traffic Analysis Resistance

### What Observer Sees

**Without valid header:**
```
1. TLS handshake to port 2080
2. HTTPS traffic (encrypted HTTP requests)
3. Normal web server responses
4. Let's Encrypt certificate validation
```
→ Looks like a normal HTTPS website

**With valid header (authorized client):**
```
1. TLS handshake to port 2080  
2. Custom encrypted data (SOCKS5 within TLS)
3. Proxied traffic (encrypted)
```
→ Still encrypted, custom protocol hidden in TLS

### DPI Resistance

- **No plaintext headers**: Everything encrypted within TLS
- **No fixed patterns**: Variable-length signatures, multiple algorithms
- **Fallback behavior**: Serves legitimate HTTP content
- **Certificate validation**: Uses real Let's Encrypt certificates

---

## Advantages of This Design

1. **Stealth**: Indistinguishable from HTTPS web server
2. **Flexibility**: 30+ encryption types
3. **Automation**: Automatic certificate renewal
4. **Management**: REST API for administration
5. **Single Port**: Everything on one port (simpler firewall)
6. **Legitimate**: Real SSL certificates, proper HTTP responses
7. **Robust**: Falls back gracefully to HTTP server

---

## Implementation Details

### HeaderDecoder (Packet Inspector)
- Examines first bytes after TLS handshake
- Validates header format and signature
- Switches pipeline based on validation result

### RerouteHandler (HTTP Disguise)
- Serves welcome page for `/`
- Handles Let's Encrypt ACME challenges
- Provides REST API for certificate management
- Looks like a legitimate web application

### SOCKS5 Pipeline
- V5ServerEncoder: Encodes SOCKS5 responses
- V5CmdReqDecoder: Decodes SOCKS5 commands
- V5CommandHandler: Handles CONNECT/BIND/UDP_ASSOCIATE

---

## Security Considerations

### Strengths
✅ Traffic analysis resistant  
✅ DPI resistant (encrypted + variable patterns)  
✅ Active probing resistant (serves HTTP)  
✅ Multiple encryption algorithms  
✅ Automatic certificate renewal

### Limitations
⚠️ TLS fingerprinting still possible  
⚠️ Timing analysis might detect proxy behavior  
⚠️ Active connection attempts can be logged  
⚠️ Certificate transparency logs reveal domains

### Mitigations
- Use common TLS configurations
- Vary response times
- Rate limit connections
- Use generic domain names
- Serve actual web content on fallback

---

## Future Enhancements

1. **DNS-01 Challenge**: Avoid any HTTP-01 requirements
2. **Decoy Content**: Serve realistic website content
3. **Traffic Shaping**: Match normal HTTPS patterns
4. **Multiple Domains**: Host multiple sites on same IP
5. **CDN Integration**: Use CDN for additional obfuscation

---

**Status**: ✅ Production-ready enhanced Trojan protocol with Let's Encrypt integration

The proxy successfully disguises itself as an HTTPS web server while providing SOCKS5 proxy services to authorized clients with multiple encryption header support.
