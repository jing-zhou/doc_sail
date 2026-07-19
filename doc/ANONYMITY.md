# 🛡️ Core Priority: 100% Absolute Anonymity with JWT RSA

At Sail, **user privacy is our first priority**. The system uses secure, industry-standard **JWT RSA (Anonymity Mode)** authentication. This mode is explicitly designed for privacy-critical scenarios—such as **Bitcoin and cryptocurrency trading**, secure financial routing, and navigating restrictive network filters—where protecting your real-world identity is just as important as encrypting your data traffic.

Instead of forcing you to create an account, register an email address, or link a credit card, Sail’s Anonymity Mode allows for a completely zero-knowledge network footprint:

- **Independent Token Acquisition:** You can get your access token completely outside of the Sail network infrastructure (for example, purchasing a voucher using cryptocurrency or trading retail gift cards via a third-party distributor). The Sail proxy network never processes your name, payment details, or personal identity.
- **Database-Free Authentication:** Traditional proxies check your credentials against a central user database, which can be vulnerable to leaks. Sail resolves this by verifying your access token using secure cryptographic math directly in the server's temporary memory. The server **performs zero database lookups**, has no user account table, and keeps no logs connecting your IP address to a personal profile.
- **Self-Contained Data Contracts:** Your anonymous token functions exactly like a digital prepaid voucher. It contains a temporary ID pre-loaded with your exact usage limits—such as a specific number of gigabytes or a fixed expiration window. The proxy honors this limit, tracks your remaining balance temporarily during your active session, and leaves no footprints behind once the token expires.
