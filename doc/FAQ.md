# ❓ Frequently Asked Questions (FAQ)

Find answers to common questions about using Sail, managing your access vouchers, and ensuring optimal privacy.

---

### 1. What happens when my JWT RSA token runs out of data or expires?

Once your token hits its data quota limit (gigabytes consumed) or reaches its chronological expiration date, the secure connection tunnel will close automatically.

- To resume service under **Anonymity Mode**, you must obtain a brand-new token string from a third-party voucher vendor and add it as a new profile in your client app.
- Because we hold user privacy as our first priority, there is no automatic billing, auto-renewal, or linked credit cards to charge.

### 2. Can I use the same anonymous token on my phone and computer at the same time?

No. Anonymous tokens (**JWT RSA**) are self-contained data vouchers intended for use on a single active device session. If you try to connect multiple devices using the exact same anonymous token string simultaneously, the server's database ledger will conflict during real-time balance accounting, resulting in connection drops or authorization rejections. For multiple devices, we recommend using a unique voucher for each device or utilizing **Account-Managed Mode (JWT SHA256)**.

### 3. How do I back up my anonymous token safely?

Since anonymous tokens are completely unrecoverable if lost, you must safeguard them like physical currency:

- **Encrypted Notes/Vaults:** Copy and paste your token string into a secure, password-protected password manager or an encrypted digital vault file on your hardware device.
- **Avoid Plaintext:** Do not store token strings in unencrypted text files, emails, or public messaging apps where malware or third parties could intercept them.
- **Remember:** If you delete the client app without backing up your token string elsewhere, your balance is permanently gone. Sail support cannot retrieve it.

### 4. Why does my web browser occasionally show a normal website when I try to connect?

If your client app attempts to connect using a corrupted, expired, or invalid token string, the server invokes the **Web Disguise Layer**. Instead of opening a proxy tunnel, it routes the connection to a normal, innocent public HTTPS landing page. If this happens, open your client app, verify your token string is pasted correctly, check your voucher balance, and try again.

### 5. Does Sail work for peer-to-peer (P2P) connections or heavy downloading?

Yes. The **Troad** protocol is optimized for high-volume, low-latency data transit. It handles standard web traffic, specialized peer-to-peer protocols, and crypto exchange network pipelines smoothly across all server nodes.
