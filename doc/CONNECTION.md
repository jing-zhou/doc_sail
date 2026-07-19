# 🔑 Available Connection Methods

To accommodate different types of users while maintaining excellent performance across thousands of concurrent connections, Sail natively supports two streamlined connection token formats:

## 🛡️ Method 1: JWT RSA (100% Absolute Anonymity Mode) — _Utmost Privacy_

- **Best For:** Privacy-centric scenarios (cryptocurrency trading, secure communication) where complete isolation from the service provider is mandatory.
- **How it Works:** Uses advanced asymmetric cryptography to validate your prepaid quota. It bypasses central user databases, ensuring that there is zero user data on the server to link back to a physical person.

### 🟢 Pros

- **100% Absolute Anonymity:** Completely unlinked from your real-world identity, email, or credit cards.
- **Zero Log Footprint:** The proxy server performs no database lookups and stores no user logs, leaving no audit trail.
- **Safe for High-Risk Scenarios:** Ideal for cryptocurrency trading, confidential transactions, and bypassing aggressive network censorship.
- **Self-Contained Security:** Operates like a digital cash voucher; if a server node is compromised, it contains no central database of users to leak.

### 🔴 Cons

- **Cash-Like Risk & Total User Responsibility:** Treat this token exactly like physical cash. Once you receive the token string, its security is entirely your responsibility.
- **Completely Unrecoverable & Untrackable:** Because the service does not track the holder or log user assignments, there is no way to monitor where a token went or recover it if it is deleted.
- **Impossible to Prove Ownership:** If you lose your token string, you cannot prove ownership to customer support, as there are no registered accounts or matching logs to verify you bought it.
- **Strictly Non-Refundable:** Due to the zero-knowledge nature of the system, transaction histories cannot be cross-referenced to issue refunds.

## 🔑 Method 2: JWT SHA256 (Account-Managed Mode)

- **Best For:** Traditional account-based setup, user want to manage their profile via a web dashboard, or need to easily sync access across multiple devices.
- **How it Works:** You log into the **Sail Gateway Web Portal** to generate this token. The proxy server decodes it instantly using a secure shared key, keeping your connection fast even during peak internet hours.

### 🟢 Pros

- **Multi-Device Account Syncing:** Allows you to easily log in and share your data tier or subscription access across your phone, tablet, and computer.
- **Centralized Management:** You can easily manage your profile, change passwords, and monitor account statistics via a web portal dashboard.
- **Simple Recovery:** Offers automated account recovery features if you lose your credentials or connection configurations.

### 🔴 Cons

- **Reduced Anonymity:** Requires account registration (such as an email address) and payment information, meaning your real-world identity is linked to the service provider.
- **Central Database Dependency:** Relies on an underlying database infrastructure to verify credentials and synchronize states, creating a potential point of data tracking.
