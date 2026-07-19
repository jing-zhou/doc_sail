# 🔑 Available Connection Methods

To accommodate different types of users while maintaining excellent performance across thousands of concurrent connections, Sail natively supports two streamlined connection token formats:

## 🛡️ Method 1: JWT RSA (100% Absolute Anonymity Mode) — _Recommended_

- **Best For:** Privacy-centric scenarios (like cryptocurrency trading or secure communication) where complete isolation from the service provider is mandatory.
- **How it Works:** Uses advanced asymmetric cryptography to validate your prepaid quota. It bypasses central user databases completely, ensuring that there is zero data on the server to link back to a physical person.

## 🔑 Method 2: JWT SHA256 (Account-Managed Mode)

- **Best For:** Traditional account-based setup, user want to manage their profile via a web dashboard, or need to easily sync access across multiple devices.
- **How it Works:** You log into the **Sail Gateway Web Portal** to generate this token. The proxy server decodes it instantly using a secure shared key, keeping your connection fast even during peak internet hours.
