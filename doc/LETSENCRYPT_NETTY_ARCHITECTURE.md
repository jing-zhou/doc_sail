# Let's Encrypt Netty HTTP Server - Architecture

## Overview

The Let's Encrypt integration now uses **pure Netty** for HTTP handling instead of Spring WebFlux. This provides better integration with your existing Netty-based SOCKS5 proxy server.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  Application Startup                         │
└────────────────────┬────────────────────────────────────────┘
                     │
          ┌──────────┴──────────┐
          │                     │
          ▼                     ▼
┌──────────────────┐   ┌─────────────────────┐
│  SOCKS5 Proxy    │   │ Let's Encrypt HTTP  │
│  Server (SSL)    │   │ Server (Port 80)    │
│  Port 2080       │   │                     │
└──────────────────┘   └──────────┬──────────┘
                                  │
                     ┌────────────┴────────────┐
                     │                         │
                     ▼                         ▼
          ┌─────────────────────┐   ┌──────────────────────┐
          │ ACME HTTP-01         │   │ REST API Endpoints   │
          │ Challenge Handler    │   │ /api/letsencrypt/*   │
          │ /.well-known/...     │   │                      │
          └─────────────────────┘   └──────────────────────┘
```

## Components

### 1. LetsEncryptHttpServer
**Purpose:** Netty HTTP server for Let's Encrypt API and ACME challenges

**Features:**
- Runs on port 80 (configurable via `letsencrypt.challenge-port`)
- Separate EventLoopGroup from main SOCKS5 proxy
- Automatically starts when `letsencrypt.enabled=true`
- Handles both ACME challenges and REST API endpoints

**Lifecycle:**
- `@PostConstruct` - Starts server in background thread
- `@PreDestroy` - Gracefully shuts down EventLoopGroups

### 2. LetsEncryptHttpHandler
**Purpose:** Netty ChannelHandler for HTTP request routing

**Endpoints:**
```
GET  /.well-known/acme-challenge/{token}  - ACME HTTP-01 challenge
GET  /api/letsencrypt/status              - Certificate status
GET  /api/letsencrypt/certificate         - Certificate info
POST /api/letsencrypt/renew               - Manual renewal
```

**Features:**
- JSON serialization via Jackson ObjectMapper
- Proper HTTP response codes
- Keep-alive support
- Error handling and logging

## Request Flow

### ACME Challenge Request
```
1. Let's Encrypt → http://yourdomain.com/.well-known/acme-challenge/abc123
2. LetsEncryptHttpServer receives request
3. LetsEncryptHttpHandler routes to handleAcmeChallenge()
4. Looks up token in LetsEncryptService.challengeTokens map
5. Returns authorization string as plain text
6. Let's Encrypt validates and issues certificate
```

### API Request (Status)
```
1. Client → http://localhost:80/api/letsencrypt/status
2. LetsEncryptHttpServer receives request
3. LetsEncryptHttpHandler routes to handleStatus()
4. Calls LetsEncryptService.needsRenewal() and getCertificateInfo()
5. Serializes response to JSON
6. Returns HTTP 200 with certificate status
```

## Why Netty Instead of Spring WebFlux?

### Advantages of Netty HTTP

1. **Consistency** - Same framework as SOCKS5 proxy
2. **Performance** - No Spring WebFlux overhead
3. **Control** - Direct control over HTTP handling
4. **Simplicity** - Fewer dependencies
5. **Integration** - Easier to share EventLoopGroups if needed

### Comparison

| Feature | Spring WebFlux | Netty HTTP |
|---------|---------------|------------|
| Framework | Reactive Spring | Pure Netty |
| Dependencies | spring-boot-starter-webflux | netty-codec-http |
| EventLoop | Internal WebFlux | Shared with SOCKS5 proxy |
| Customization | Limited | Full control |
| Memory | Higher | Lower |
| Startup Time | Slower | Faster |

## Configuration

### Enable Netty HTTP Server (Default)
```properties
letsencrypt.enabled=true
letsencrypt.challenge-port=80
letsencrypt.use-spring-webflux=false  # Use Netty (default)
```

### Use Spring WebFlux (Deprecated)
```properties
letsencrypt.enabled=true
letsencrypt.use-spring-webflux=true  # Use Spring WebFlux (not recommended)
```

## Pipeline Configuration

### Netty HTTP Server Pipeline
```
ServerBootstrap
  ↓
ChannelInitializer<SocketChannel>
  ↓
  ├─ HttpServerCodec          (HTTP request/response codec)
  ├─ HttpObjectAggregator     (Aggregate HTTP chunks)
  └─ LetsEncryptHttpHandler   (Custom handler for routing)
```

### Handler Pipeline
```
FullHttpRequest
  ↓
channelRead0()
  ↓
Route based on URI:
  ├─ /.well-known/acme-challenge/* → handleAcmeChallenge()
  ├─ /api/letsencrypt/status       → handleStatus()
  ├─ /api/letsencrypt/certificate  → handleCertificateInfo()
  ├─ /api/letsencrypt/renew        → handleRenewal()
  └─ *                             → sendNotFound()
```

## JSON Serialization

Uses Jackson ObjectMapper for JSON serialization:

```java
ObjectMapper objectMapper = new ObjectMapper();
String json = objectMapper.writeValueAsString(data);
```

**Response Format:**
```json
{
  "enabled": true,
  "domains": ["example.com"],
  "staging": false,
  "needsRenewal": false,
  "certificateInfo": {
    "subject": "CN=example.com",
    "issuer": "CN=Let's Encrypt Authority X3",
    "notBefore": "2024-01-01T00:00:00Z",
    "notAfter": "2024-04-01T00:00:00Z",
    "serialNumber": "abc123",
    "domains": ["example.com"],
    "daysUntilExpiration": 60
  }
}
```

## Error Handling

### HTTP Error Responses
```json
{
  "error": "Not Found",
  "status": 404,
  "message": "Endpoint not found",
  "path": "/api/unknown"
}
```

### Exception Handling
- `exceptionCaught()` logs and closes connection
- All endpoints wrapped in try-catch
- Errors returned as JSON with appropriate status codes

## Threading Model

### EventLoopGroups
```
LetsEncryptHttpServer:
  - bossGroup: NioEventLoopGroup(1)      // Accept connections
  - workerGroup: NioEventLoopGroup()     // Handle I/O (default: 2 * cores)

SOCKS5 Proxy Server:
  - bossGroup: Separate from Let's Encrypt
  - workerGroup: Separate from Let's Encrypt
```

**Isolation:** Each server has its own EventLoopGroups for complete isolation.

## Testing

### Test ACME Challenge
```bash
curl http://localhost:80/.well-known/acme-challenge/test-token
```

### Test Status Endpoint
```bash
curl http://localhost:80/api/letsencrypt/status
```

### Test Certificate Info
```bash
curl http://localhost:80/api/letsencrypt/certificate
```

### Test Manual Renewal
```bash
curl -X POST http://localhost:80/api/letsencrypt/renew
```

## Monitoring

### Logs
```
INFO  LetsEncryptHttpServer - Let's Encrypt HTTP server started on port 80
INFO  LetsEncryptHttpServer - ACME challenge endpoint: http://example.com/.well-known/acme-challenge/{token}
INFO  LetsEncryptHttpServer - API endpoints available at http://localhost:80/api/letsencrypt/*
DEBUG LetsEncryptHttpHandler - Received HTTP request: GET /api/letsencrypt/status
INFO  LetsEncryptHttpHandler - Serving ACME challenge for token: abc123
```

### Metrics
- Track request count
- Track response times
- Monitor error rates
- Certificate expiration alerts

## Security Considerations

1. **Port 80 Required** - Let's Encrypt requires HTTP port 80
2. **No Authentication** - ACME challenges are public
3. **API Security** - Consider adding authentication for API endpoints
4. **Rate Limiting** - Consider adding rate limiting for API
5. **Firewall** - Ensure port 80 is open for ACME validation

## Migration from Spring WebFlux

The old Spring WebFlux implementation is deprecated but still available:

```properties
# Enable deprecated Spring WebFlux (not recommended)
letsencrypt.use-spring-webflux=true
```

**Deprecated Classes:**
- `LetsEncryptController` (use `LetsEncryptHttpHandler` instead)
- `AcmeHttpChallengeHandler` (integrated into `LetsEncryptHttpHandler`)

## Future Enhancements

1. **HTTPS Support** - Add SSL/TLS to HTTP server
2. **Metrics** - Prometheus metrics endpoint
3. **Rate Limiting** - Request rate limiting
4. **Authentication** - API key authentication
5. **WebSocket** - Real-time certificate status updates
6. **Dashboard** - Web-based management interface

## Troubleshooting

### Server doesn't start
```
Check logs for:
- Port 80 already in use
- Permission denied (need root/sudo for port 80)
- EventLoopGroup initialization errors
```

### Challenges fail
```
- Verify port 80 is accessible from internet
- Check firewall rules
- Verify DNS points to correct server
- Check challenge token is being stored
```

### API returns errors
```
- Check LetsEncryptService is running
- Verify certificate files exist
- Check file permissions
- Review error logs
```

---

**Status:** ✅ **Production Ready**

The Netty-based HTTP server provides a robust, performant solution for Let's Encrypt certificate management integrated seamlessly with your Netty SOCKS5 proxy server.

