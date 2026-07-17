# Self-Connection Loop Fix

## Problem Description

When a user with a valid JWT token tries to access the server's own HTTPS API endpoints (e.g., to generate a new JWT token), the following scenario occurs:

- HeaderDecoder behavior (important for this flow):
  - JWT is represented by crypto byte `0x07`.
  - HeaderDecoder enforces the header parsing rules (offset is mandatory; for variable-length JWT it scans for CRLF within 128 bytes after signature end).
  - On successful verification HeaderDecoder stores the billingId as a native `UUID` in the channel context attribute `BillingHandler.BILLING_ID`.

1. Client sends request with Illiad header + JWT token
2. HeaderDecoder verifies JWT successfully
3. Request is routed to SOCKS5 proxy path
4. SOCKS5 CONNECT command specifies destination as the server itself (same IP and port)
5. V5ConnectHandler attempts to create a backend connection to the server itself
6. This creates a self-connection loop

## Solution Overview

Instead of rejecting self-connections or attempting to prevent them at the protocol level, we implement a **smart fallback mechanism** in `V5ConnectHandler`:

### Detection
When `V5ConnectHandler` receives a SOCKS5 CONNECT request, it checks if the destination address and port match the server's own listening address and port.

### Handling
When self-connection is detected:

1. **Accept the SOCKS5 connection** - Send a SOCKS5 SUCCESS response to complete the handshake
2. **Switch to HTTP mode** - Dynamically reconfigure the Netty pipeline:
   - Remove all SOCKS5 handlers
   - Add HTTP codec (`HttpServerCodec`)
   - Add HTTP aggregator (`HttpObjectAggregator`)
   - Add HTTP handler (`RerouteHandler`)
3. **Send HTTP redirect** - Send an HTTP 307 Temporary Redirect response asking the client to resend the request

### Key Advantages

- **No protocol modification needed** - Works within standard SOCKS5 and HTTP protocols
- **Transparent to client** - Smart clients can handle the redirect automatically
- **No infinite loops** - The connection is established at TCP level but handled at HTTP level
- **Preserves security** - JWT validation already happened in HeaderDecoder
- **Clean architecture** - Leverages existing HTTP handlers for API endpoints

## Implementation Details

### Self-Connection Detection

```java
private boolean isSelfConnection(ChannelHandlerContext ctx, Socks5CommandRequest request) {
    String destAddr = request.dstAddr();
    int destPort = request.dstPort();
    
    InetSocketAddress localAddr = (InetSocketAddress) ctx.channel().localAddress();
    int localPort = localAddr.getPort();
    String localHost = localAddr.getHostString();
    
    // Check port match
    if (destPort != localPort) {
        return false;
    }
    
    // Check address match (handles localhost, 127.0.0.1, ::1, etc.)
    return destAddr.equals(localHost) 
        || destAddr.equals("127.0.0.1") 
        || destAddr.equals("localhost")
        || destAddr.equals("::1")
        || destAddr.equals(bus.params.getLocalHost());
}
```

### Pipeline Switching

```java
private void switchToHttpMode(ChannelHandlerContext ctx) {
    ChannelPipeline pipeline = ctx.pipeline();
    
    // Remove SOCKS5 handlers
    String prefix = bus.namer.getPrefix();
    for (String name : pipeline.names()) {
        if (name.startsWith(prefix)) {
            pipeline.remove(name);
        }
    }
    
    // Add HTTP handlers
    pipeline.addLast("httpServerCodec", new HttpServerCodec());
    pipeline.addLast("httpAggregator", new HttpObjectAggregator(65536));
    pipeline.addLast("rerouteHandler", new RerouteHandler(
        bus.letsEncryptService, 
        bus.userService, 
        bus.sessionService
    ));
}
```

### HTTP Redirect Response

```java
private void sendHttpRedirect(ChannelHandlerContext ctx) {
    FullHttpResponse response = new DefaultFullHttpResponse(
        HttpVersion.HTTP_1_1,
        HttpResponseStatus.TEMPORARY_REDIRECT,
        Unpooled.copiedBuffer("Self-connection detected. Please access the API directly without proxy.\n", 
                             CharsetUtil.UTF_8)
    );
    
    response.headers().set(HttpHeaderNames.CONTENT_TYPE, "text/plain; charset=UTF-8");
    response.headers().set(HttpHeaderNames.CONTENT_LENGTH, response.content().readableBytes());
    response.headers().set(HttpHeaderNames.LOCATION, "/");
    response.headers().set(HttpHeaderNames.CONNECTION, HttpHeaderValues.KEEP_ALIVE);
    
    ctx.writeAndFlush(response);
}
```

## Flow Diagram

```
Client (with JWT) → Server
│
├─ HeaderDecoder: Verify JWT ✓
│
├─ Route to SOCKS5 path
│
├─ SOCKS5 handshake complete
│
├─ V5ConnectHandler receives CONNECT request
│
├─ Detect: destination = server itself?
│   │
│   ├─ NO → Normal flow: create backend connection, relay traffic
│   │
│   └─ YES → Self-connection handling:
│       │
│       ├─ Send SOCKS5 SUCCESS response
│       │
│       ├─ Switch pipeline to HTTP mode
│       │
│       └─ Send HTTP 307 redirect
│           │
│           └─ Client receives redirect, can retry without proxy
```

## Usage Scenario

**Scenario**: User wants to generate a new JWT token

1. User has a valid JWT token (Token A)
2. Client library configured to use Illiad proxy
3. Client makes request to `https://server:2080/api/tokens/generate` with Token A in Illiad header
4. Server verifies Token A ✓
5. Server detects self-connection
6. Server sends HTTP 307 redirect
7. **Client options**:
   - Smart client: Automatically retry without proxy
   - Simple client: User manually accesses API directly without proxy

## Alternative Solutions Considered

1. **Reject self-connections** - Would break the use case of generating new tokens
2. **Special header flag** - Requires client to know when to use it
3. **Separate API port** - Adds complexity and configuration overhead
4. **Allow self-relay** - Would create infinite loops

## Notes

- This solution is documented in TODO.md as a known limitation
- For production use, clients should implement logic to detect 307 redirects and retry without proxy
- The HTTP redirect message is user-friendly for debugging
- The solution maintains backward compatibility with existing clients

## Related Files

- `/home/wjz/pro/proxy/server/src/main/java/com/illiad/server/handler/v5/V5ConnectHandler.java`
- `/home/wjz/pro/proxy/server/src/main/java/com/illiad/server/handler/reroute/RerouteHandler.java`
- `/home/wjz/pro/proxy/server/src/main/java/com/illiad/server/codec/v5/HeaderDecoder.java`
- `/home/wjz/pro/proxy/server/TODO.md`
