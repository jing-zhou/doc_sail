# 🔍 How It Works: The "Two-Face" Disguise

To ensure your internet activity is never flagged or throttled by network censorship, the Troad protocol hides your proxy traffic in plain sight by blending seamlessly into standard global web infrastructure:

```
                    Client connects to port 443
                              ↓
                    TLS/SSL handshake completes
                              ↓
                    Client sends first bytes after TLS
                              ↓
                    ┌─────────┴─────────────────┐
                    │                           │
         Troad Header + SOCKS5         Standard HTTPS Request
                    │                           │
         (Header verified ✓)         (No valid header / failed)
                    ▼                           ▼
          ┌──────────────────────┐  ┌──────────────────────┐
          | Obfuscation response |  |                      |
          |         ↓            |  |                      |
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

1. **The Public Face (For Network Monitors) 🌐**  
   If an internet provider or an automated firewall tries to inspect the Sail server on **Port 443**, it behaves exactly like a harmless, normal public HTTPS website. It displays a standard "Welcome" page and handles regular secure web requests. It looks entirely innocent to outside scanners because it mimics authentic HTTPS web server behavior.

2. **The Hidden Face (For Your Private Token) 🔒**  
   When your client app sends a valid access token in its header, the server instantly unlocks a high-speed, secure proxy tunnel. Your real internet traffic passes through undetected, safely hidden beneath the appearance of standard web browsing over the regular web traffic port.
