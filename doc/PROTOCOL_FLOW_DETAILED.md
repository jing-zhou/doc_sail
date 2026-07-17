# Enhanced Trojan Proxy - Complete Protocol Flow

## Overview

This document details the complete request flow for both authorized (illiad) clients and unauthorized (HTTPS) clients.

---

## Authorized Client Flow (illiad Protocol)

### Step-by-Step: Successful Proxy Connection

```
┌─────────────┐                                    ┌─────────────┐
│   Client    │                                    │   Server    │
└──────┬──────┘                                    └──────┬──────┘
       │                                                  │
       │  1. TCP SYN                                     │
       │─────────────────────────────────────────────────>│
       │                                                  │
       │  2. TCP SYN-ACK                                 │
       │<─────────────────────────────────────────────────│
       │                                                  │
       │  3. TCP ACK                                     │
       │─────────────────────────────────────────────────>│
       │                                                  │
       │  4. TLS ClientHello                             │
       │─────────────────────────────────────────────────>│
       │                                                  │
       │  5. TLS ServerHello + Certificate + ...         │
       │<─────────────────────────────────────────────────│
       │                                                  │
       │  6. TLS Key Exchange + ChangeCipherSpec         │
       │─────────────────────────────────────────────────>│
       │                                                  │
       │  TLS Tunnel Established ✓                       │
       │═════════════════════════════════════════════════│
       │                                                  │
       │  7. [illiad Header] + [SOCKS5 CONNECT]          │
       │     ┌──────────────────────────────┐            │
       │     │ Length (2B): 0x0025          │            │
       │     │ Crypto (1B): 0x10            │            │
       │     │ Signature (28B): [encrypted] │            │
       │     │ Offset (1B): 0x42            │            │
       │     │ CRLF (2B): 0x0D 0x0A         │            │
       │     │ SOCKS5: 0x05 0x01 0x00       │            │
       │     │ ATYP: 0x03 (domain)          │            │
       │     │ Domain: example.com:443      │            │
       │     └──────────────────────────────┘            │
       │─────────────────────────────────────────────────>│
       │                                                  │
       │                                  8. HeaderDecoder│
       │                                     reads bytes  │
       │                                     ↓            │
       │                                  Verify header:  │
       │                                   - Check length │
       │                                   - Check crypto │
       │                                   - Extract sig  │
       │                                   - Find CRLF    │
       │                                   - Verify secret│
       │                                     ↓            │
       │                                  ✅ Valid!       │
       │                                     ↓            │
       │                                  Switch pipeline:│
       │                                   + V5Encoder    │
       │                                   + V5Decoder    │
       │                                   + V5Handler    │
       │                                     ↓            │
       │                                  9. Process      │
       │                                     SOCKS5       │
       │                                     CONNECT      │
       │                                     ↓            │
       │  10. [SOCKS5 Response: Connected]               │
       │     0x05 0x00 0x00 0x01 ...                     │
       │<─────────────────────────────────────────────────│
       │                                                  │
       │                                  11. Establish   │
       │                                      TCP to      │
       │                                      example.com │
       │                                      ↓           │
┌──────┴──────┐                          ┌───────────────┴────────┐
│   Client    │                          │ Server ↔ example.com   │
└──────┬──────┘                          └───────────┬────────────┘
       │                                              │
       │  12. Data: HTTP GET / HTTP/1.1              │
       │─────────────────────────────────────────────>│
       │                                              │─────────>
       │                                              │ Forward to
       │                                              │ example.com
       │                                              │
       │  13. Data: HTTP/1.1 200 OK + HTML           │<────────
       │<─────────────────────────────────────────────│ Response
       │                                              │
       │  Traffic relay continues...                  │
       │═════════════════════════════════════════════>│═════════>
       │<═════════════════════════════════════════════│<═════════
       │                                              │
```

### What Happens
1. **TCP + TLS**: Normal connection establishment
2. **illiad Header + SOCKS5**: Client sends both in one packet
3. **Header Verification**: Server validates signature
4. **Pipeline Switch**: Changes to SOCKS5 handlers
5. **Proxy Established**: Relays traffic to destination
6. **Transparent Relay**: All data flows through encrypted tunnel

---

## Unauthorized Client Flow (Standard HTTPS)

### Step-by-Step: Web Browser Access

```
┌─────────────┐                                    ┌─────────────┐
│   Browser   │                                    │   Server    │
└──────┬──────┘                                    └──────┬──────┘
       │                                                  │
       │  1-6. TCP + TLS Handshake (same as above)      │
       │═════════════════════════════════════════════════│
       │                                                  │
       │  7. HTTP GET / HTTP/1.1                         │
       │     ┌──────────────────────────────┐            │
       │     │ GET / HTTP/1.1               │            │
       │     │ Host: proxy.example.com      │            │
       │     │ User-Agent: Mozilla/5.0 ...  │            │
       │     │ Accept: text/html,*/*        │            │
       │     │                              │            │
       │     └──────────────────────────────┘            │
       │─────────────────────────────────────────────────>│
       │                                                  │
       │                                  8. HeaderDecoder│
       │                                     reads bytes  │
       │                                     ↓            │
       │                                  Parse header:   │
       │                                   First bytes:   │
       │                                   0x47 0x45 0x54 │
       │                                   ("GET")        │
       │                                     ↓            │
       │                                  ❌ Not illiad!  │
       │                                     ↓            │
       │                                  Switch pipeline:│
       │                                   + HttpCodec    │
       │                                   + HttpAggr     │
       │                                   + RerouteHndlr │
       │                                     ↓            │
       │                                  9. Process HTTP │
       │                                     Route: "/"   │
       │                                     ↓            │
       │  10. HTTP/1.1 200 OK                            │
       │      Content-Type: text/plain                   │
       │      Content-Length: 7                          │
       │                                                  │
       │      Welcome                                    │
       │<─────────────────────────────────────────────────│
       │                                                  │
```

### What Happens
1. **TCP + TLS**: Normal HTTPS connection
2. **HTTP Request**: Standard GET request (no illiad header)
3. **Header Detection Fails**: First bytes are "GET", not illiad format
4. **Pipeline Switch**: Changes to HTTP handlers
5. **HTTP Response**: Serves content like normal web server

---

## Let's Encrypt ACME Challenge Flow

### Step-by-Step: Domain Validation

```
┌──────────────┐                                   ┌─────────────┐
│ Let's Encrypt│                                   │   Server    │
└──────┬───────┘                                   └──────┬──────┘
       │                                                  │
       │  1-6. TCP + TLS Handshake                       │
       │═════════════════════════════════════════════════│
       │                                                  │
       │  7. GET /.well-known/acme-challenge/abc123      │
       │─────────────────────────────────────────────────>│
       │                                                  │
       │                                  8. HeaderDecoder│
       │                                     ↓            │
       │                                  Not illiad      │
       │                                     ↓            │
       │                                  HTTP Pipeline   │
       │                                     ↓            │
       │                                  RerouteHandler  │
       │                                     ↓            │
       │                                  Route matches:  │
       │                               /.well-known/acme..│
       │                                     ↓            │
       │                                  Lookup token:   │
       │                                  "abc123"        │
       │                                     ↓            │
       │                                  From service:   │
       │                           challengeTokens.get()  │
       │                                     ↓            │
       │  9. HTTP/1.1 200 OK                             │
       │     Content-Type: text/plain                    │
       │                                                  │
       │     [authorization string]                      │
       │<─────────────────────────────────────────────────│
       │                                                  │
       │  10. Validates authorization                    │
       │      ✓ Domain ownership confirmed               │
       │                                                  │
```

---

## Failed illiad Header Flow

### Step-by-Step: Invalid Authentication

```
┌─────────────┐                                    ┌─────────────┐
│ Bad Client  │                                    │   Server    │
└──────┬──────┘                                    └──────┬──────┘
       │                                                  │
       │  1-6. TCP + TLS Handshake                       │
       │═════════════════════════════════════════════════│
       │                                                  │
       │  7. [Malformed illiad header]                   │
       │     ┌──────────────────────────────┐            │
       │     │ Length: 0x0025               │            │
       │     │ Crypto: 0x10                 │            │
       │     │ Signature: [WRONG PASSWORD]  │            │
       │     │ ...                          │            │
       │     └──────────────────────────────┘            │
       │─────────────────────────────────────────────────>│
       │                                                  │
       │                                  8. HeaderDecoder│
       │                                     ↓            │
       │                                  Verify:         │
       │                                   - Length ✓     │
       │                                   - Crypto ✓     │
       │                                   - Signature ❌  │
       │                                     ↓            │
       │                                  Verification    │
       │                                  FAILED!         │
       │                                     ↓            │
       │                                  Switch to HTTP: │
       │                                   + HttpCodec    │
       │                                   + HttpAggr     │
       │                                   + RerouteHndlr │
       │                                     ↓            │
       │  9. HTTP/1.1 200 OK (Welcome page)              │
       │<─────────────────────────────────────────────────│
       │                                                  │
       │  Attacker sees: "Just a web server"             │
       │  ✓ No indication proxy exists                   │
       │  ✓ No error message revealing purpose           │
       │  ✓ Looks completely normal                      │
```

### Why This Matters
- **Standard Trojan**: Closes connection → suspicious
- **This Implementation**: Serves HTTP → looks normal
- **Result**: Failed authentication is indistinguishable from normal web browsing

---

## Header Format Details

```
┌─────────────┬──────────┬────────────────┬────────┬──────┐
│   Length    │  Crypto  │   Signature    │ Offset │ CRLF │
│  (2 bytes)  │ (1 byte) │  (N bytes)     │ (1B)   │ (2B) │
└─────────────┴──────────┴────────────────┴────────┴──────┘
      ↓            ↓            ↓              ↓       ↓
   0x0025        0x10      [28 bytes]        0x01   0x0D0A
  (total or     (crypto)   [encrypted]      (offset) (CRLF)
   signature)
```

Important rules (latest design):

- Offset is MANDATORY for all headers. It encodes the padding length between the end of the signature and the CRLF marker.

- For fixed-length signatures (crypto types that imply a fixed digest/signature length), the 2-byte Length field indicates the length of the *entire header* up to and including the CRLF. In other words:
  Length = 2 (length) + 1 (crypto) + signature_length + 1 (offset) + offset_padding + 2 (CRLF)
  The decoder validates the length, reads the specified signature bytes, reads the offset byte, then seeks the CRLF at (end-of-signature + offset). If the CRLF is not present at that position, the header is considered invalid and the stream falls back to HTTP.

- For variable-length signatures (for example, JWT where the signature length is not fixed), the 2-byte Length field indicates the length *up to the end of the signature only* (i.e., it covers length field + crypto + signature). In this case the decoder MUST scan forward from the end of signature and locate the CRLF within a 128-byte window. The offset byte still exists and is used to indicate padding between signature end and where CRLF is expected, but the decoder should accept any CRLF found within 128 bytes after the signature; if no CRLF is found within 129 bytes (signature end + 128) the header is invalid and the stream reverts to HTTP.

- The crypto identifier byte for JWT has been assigned `0x07`. The `Secret` verification will treat `0x07` as JWT and perform JWT parsing/validation.

- On successful verification, the HeaderDecoder sets the billing identifier (a real `UUID` object) into the channel context attribute (`BillingHandler.BILLING_ID`). This is stored as a native `UUID` (not a string) for efficiency.

- The `verify()`/`verifyAndGetBillingId()` flow was updated: the verification method returns a `UUID` (billingId) on success or a special system-wide `VERIFICATION_FAILURE` UUID to indicate verification failure. The `HeaderDecoder` checks this value and only proceeds to add SOCKS5 handlers when verification succeeded and the returned UUID is not the `VERIFICATION_FAILURE` sentinel.

- If verification fails or the header cannot be parsed according to the above rules, `HeaderDecoder` removes itself and the pipeline switches to HTTP handlers (serves the disguised HTTPS site).

---

## Pipeline Switching

### Before Header Verification
```
ChannelPipeline:
├─ SslHandler (TLS encryption/decryption)
├─ LoggingHandler
└─ HeaderDecoder (reads first bytes)
```

### After Valid illiad Header
```
ChannelPipeline:
├─ SslHandler
├─ LoggingHandler
├─ V5ServerEncoder (SOCKS5 response encoder)
├─ V5CmdReqDecoder (SOCKS5 request decoder)
└─ V5CommandHandler (SOCKS5 command processor)

HeaderDecoder removed ✓
Buffer advanced past illiad header ✓
Remaining bytes = SOCKS5 protocol ✓
```

### After Invalid/No illiad Header
```
ChannelPipeline:
├─ SslHandler
├─ LoggingHandler
├─ HttpServerCodec (HTTP request/response codec)
├─ HttpObjectAggregator (combines HTTP chunks)
└─ RerouteHandler (HTTP request router)

HeaderDecoder removed ✓
Entire stream treated as HTTP ✓
```

---

## Summary

### The Brilliance of This Design

1. **Single Request**: illiad header + SOCKS5 sent together
2. **Header First**: Authentication verified before processing SOCKS5
3. **Transparent Relay**: Once authenticated, pure traffic relay
4. **Perfect Disguise**: Failed auth → normal HTTPS response
5. **No Indication**: Attackers see a web server, nothing suspicious

### Traffic Characteristics

**Authorized (illiad) Client:**
- Encrypted TLS tunnel
- Custom first packet (illiad + SOCKS5)
- Then pure relay traffic
- Indistinguishable from HTTPS to observer

**Unauthorized (HTTP) Client:**
- Standard HTTPS traffic
- Normal HTTP requests/responses
- Fully functional web server
- Legitimately serves Let's Encrypt validation

---

**Result**: A proxy that's impossible to detect because it actually IS a functional HTTPS web server! 🎭
