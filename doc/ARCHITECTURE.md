# 🏗️ System Architecture: How Sail Works on Your Devices

To understand how Sail protects your privacy, it helps to look at the two main components that work together: the **Local Client App** installed on your device, and the **Remote Proxy Server** hosted on the internet.

```text
          [ Device Apps/Traffic ]
                    │
                    ▼
┌────────────────────────────────────────┐
│ 🛡️ Local Client App                    │
│   • Android: Native OS VPN Interface   │
│   • Desktop: SOCKS5 & HTTP Interfaces  │
└───────────────────┬────────────────────┘
                    │
        (🔒 Encrypted Troad Tunnel)
                    │
                    ▼
┌────────────────────────────────────────┐
│ ⛵ Remote Proxy Server (Port 443)       │
│   • Zero-Knowledge / RAM Verification  │
│   • Fetches Requested Target Web Assets│
└───────────────────┬────────────────────┘
                    │
                    ▼
        [ Destination Website ]
```

## 1. The Local Client (On Your Device)

The client app acts as a secure local gateway on your computer or phone. It intercepts your device’s outbound internet traffic and bundles it into the encrypted **Troad** protocol before sending it safely to the internet.

Depending on your operating system, the client integrates directly at the system level to capture traffic seamlessly:

- **🤖 On Android:** The app creates a native **VPN Interface** at the Android OS level. Once activated, the Android operating system automatically routes all outbound traffic from your web browsers, crypto wallets, and apps directly through the Sail client.
- **💻 On Desktop (Windows, Mac, Linux):** The client provides two flexible integration points at the operating system level:
  - A **SOCKS5 Proxy Interface** for handling raw, high-speed layer 3 traffic (TCP, UDP Relay).
  - An **HTTP/HTTPS Proxy Interface** for seamless, universal compatibility with standard desktop web browsers and internet applications.

## 2. The Remote Proxy Server (On the Internet)

The Sail Proxy Server is hosted on a remote internet node, listening on standard HTTPS **Port 443**. It acts as a shield between your device and the websites you visit.

When your local client transmits data to the server, the server performs two critical functions:

- **Decouples Your Identity:** In line with our commitment to user privacy as our first priority, if you connect using an anonymous token, the server validates your access entirely in temporary memory using cryptographic math. It uses no central database and stores no logs, leaving your real-world identity entirely disconnected from your traffic.
- **Fetches Your Requests:** Once validated, the server decrypts the traffic, forwards your request to your target website (such as a Bitcoin exchange), receives the website's response, and passes it securely back to your local device. To the destination website, the request looks like it came directly from the Sail server, completely masking your true IP address and location.
