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
