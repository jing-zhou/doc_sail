# 🛡️ Core Priority: 100% Absolute Anonymity with JWT RSA

At Sail, **user privacy is our first priority**. The system uses secure, industry-standard **JWT RSA (Anonymity Mode)** authentication. This mode is explicitly designed for privacy-critical scenarios—such as **Bitcoin and cryptocurrency trading**, secure financial routing, and navigating restrictive network filters—where protecting your real-world identity is just as important as encrypting your data traffic.

Instead of forcing you to create an account, register an email address, or link a credit card, Sail’s Anonymity Mode allows for a completely zero-knowledge network footprint:

- **Independent Token Acquisition:** You can get your access token completely outside of the Sail network infrastructure (for example, purchasing a voucher using cryptocurrency, trading retail gift cards via a third-party distributors/ payment processors). The Sail proxy network never processes your name, payment details, or personal identity.
- **Database-Free Authentication:** Traditional proxies check your credentials against a central user database, which can be vulnerable to leaks. Sail resolves this by verifying your access token using secure cryptographic math directly in the server's temporary memory. The server **performs zero database lookups**, has no user account table, keeps no logs connecting your IP address to a personal profile; nor does Sail server save/store the token itself.
- **Self-Contained Data Contracts:** Your anonymous token functions exactly as a digital prepaid voucher. It contains a temporary ID pre-loaded with usage limits—such as a specific number of gigabytes or a fixed expiration window. The proxy honors this limit, tracks the remaining balance against the ont-time temporary ID. an user might share/gift/resell tokens.

## ⚠️ Important Disclaimer: Treat Your Token Like Cash

User privacy is our first priority, Sail completely decouples itself from your token once it is issued. This means you must manage your token with strict operational security:

- **Total User Responsibility:** Treat this token exactly like physical cash paper in your wallet. Its security, storage, and backup are entirely up to you.
- **Completely Unrecoverable & Untrackable:** Because our system does not map tokens to names, emails, or hardware IDs, we cannot track your token across the web. If you accidentally delete it or overwrite it, it is gone forever.
- **Impossible to Prove Ownership:** There is no "forgot password" form or database profile to look up. If you lose the token, you cannot prove ownership to customer support because we have no technical means to verify who originally held it, or who originally bought it.
- **Strictly Non-Refundable:** Due to the zero-knowledge nature of the system, it is technically impossible to audit token ownership or usage patterns to process a refund. Safeguard your keys accordingly.

## 🔒 Operational Transparency & Historical Accounting Proofs

To maintain network integrity, ensure fair usage compliance, and provide indisputable mathematical validation of service delivery, Sail maintains structured data records within a persistent database. These logs remain archived on file even after an anonymous token expires.

### 1. What Exactly is Kept in the Database?

When you use a JWT RSA (Anonymity Mode) token, the server writes precise transactional ledger entries to its database. These records consist of:

- The **One-Time Temporary ID** associated with the token.
- The **Detailed Usage Log** for every connection window, capturing the exact **Start Time** and **Stop Time**.
- The **Exact Bytes Transported** during each specific session.

### 2. Why are Detailed Accounting Proofs Retained?

These granular logs serve as immutable **accounting proofs**. Because Sail operates a completely decentralized server infrastructure, these logs are mathematically required to build up, calculate, and prove the total aggregated balance of a JWT RSA token. This ledger protects both the user and the platform, serving as an undisputable record that verifies exactly how much data quota was consumed, when it was consumed, and what balance remains available on the prepaid token contract.

### 3. How Your Identity Remains Protected Under a Court Order or Subpoena

User privacy is our first priority, the architectural separation between the **Prepaid Token** and your **Real Identity** ensures that even highly detailed session ledgers cannot be used to unmask you:

```text
[ Persistent Database Ledger ] ──► [ Decodes Session Log for Temporary ID ]
                                           │
       ┌───────────────────────────────────┼───────────────────────────────────┐
       ▼                                   ▼                                   ▼
【 What Can Be Disclosed 】    【 What Does NOT Exist 】          【 Financial Separation 】
 • Token ID Session Ledgers     • Real Name / Email Address        • FastSpring MOR Processes Payments
 • Start & Stop Timestamps      • Inbound Home IP Addresses        • No Credit Card/Wallet Trail
 • Precise Bytes Transported    • Device / Hardware IDs            • No Identity Linkage in Proxy DB
```

- **Nameless Transaction Ledgers:** The database rows are completely decoupled from human parameters. The table logs that a specific string of characters (the Temporary Token ID) initiated a connection at time A, disconnected at time B, and transferred a specific volume of packets. There are no fields connecting these timelines to a real name, email account, or physical user identity.
- **No Inbound Connection Logging:** While the server tracks detailed duration and byte counts to calculate voucher balances, it does not log or store your home internet service provider's inbound IP address in these persistent database ledger rows.
- **Absolute Financial Decoupling:** Because all commercial transaction compliance, automated tax accounting, and global payment reconciliation are managed outside our core infrastructure via our US Merchant of Record (**FastSpring**), a court order or civil subpoena demanding Sail's database logs will yield zero financial paths or real-world banking trails.

### 4. Legal Compliance Reality

If an attorney or law enforcement agency serves a legally binding court order or subpoena targeting a specific token ID, Sail will comply by presenting the relevant entries from the audit ledger database.

However, because of our zero-knowledge structural design, the only information that factually exists to be handed over is the **isolated data-consumption metrics and session durations of a nameless ID**. A third party will receive proof that a token was used, but they will remain completely unable to prove _who_ used it or _where_ the user is physically located.
