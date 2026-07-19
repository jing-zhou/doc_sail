# 📖 Glossary of Terms

A quick reference guide defining the core components and concepts used across the Sail Proxy Service documentation.

---

### ⛵ Sail

The official brand name of this privacy proxy service network.

### 🚗 Troad Protocol

Sail's proprietary, standalone connection standard. It is a highly optimized alteration of older proxy styles that completely replaces static passwords with dynamic security tokens inside the data headers to achieve commercial scalability and advanced anti-tracking. It is entirely incompatible with standard Trojan proxies.

### 📜 JWT RSA Token (Anonymity Mode)

An advanced, self-contained connection credential that uses asymmetric encryption. It carries its own validation footprint and data limits natively. It allows the server to verify your connection completely in-memory without checking an underlying user account database, ensuring 100% user anonymity.

### 🔑 JWT SHA256 Token (Account-Managed Mode)

A digital connection key generated when a user signs up via the central web portal. It syncs with a standard registration account profile to allow easy multi-device usage tracking and subscription tier renewals.

### 🏗️ Local Client App

The dedicated software application installed directly on your equipment (Android phone or desktop computer). It acts as a local gate keeper, safely intercepting your internet traffic and packing it into the Troad encryption layer before sending it down the pipeline.

### 🌐 Remote Proxy Server

The secure server machine hosted on the internet listening on Port 443. It receives your client's encrypted request, checks token credentials state-lessly, strips away your home location identifiers, and fetches web targets on your behalf.

### 🎭 Web Disguise Layer

The safety feature that multiplexes all traffic over standard secure HTTPS **Port 443**. To an outside inspector or censor, the server looks like an ordinary public website with a standard welcome page. It only reveals the proxy pathway when a valid token header is presented.

### 🧮 Accounting Proofs

Granular historical database entries (capturing start times, stop times, and precise byte transfers) written to a persistent server disk. These records are mathematically required to build up and calculate the remaining aggregated balance of a nameless token, protecting users from quota discrepancies.
