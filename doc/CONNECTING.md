# 🚀 Connecting Your Devices to Sail

Configuring your Sail client app is a straightforward process. Once you have acquired your token string, follow the configuration steps detailed below for your operating system.

---

## 🤖 Android Setup Guide

1. Download and launch the official **Sail Client App** on your Android device.
2. Select the **Profiles** tab and tap **Add New Profile**.
3. In the input configuration block, paste your raw access token string (accepts either a **JWT RSA** anonymous voucher or a **JWT SHA256** account token).
4. Return to the main home interface and toggle the **Connect** switch.
5. When the Android system prompts you with a _VPN Connection Request_, tap **Allow/OK** to grant system permission.
6. Your entire phone's outbound application and web traffic is now safely routed through Sail.

---

## 💻 Desktop Setup Guide (Windows, Mac, Linux)

1. Launch the **Sail Desktop Client** on your computer.
2. Navigate to **Profiles** and paste your token string directly into the connection profile box.
3. Save your changes and click **Start Service**.
4. The desktop client will automatically apply system-level proxy routing parameters:
   - **Web Browsers:** Your operating system's global HTTP/HTTPS proxy rules will point natively to the client.
   - **Advanced Application Support:** Specialized tools, terminal environments, or cryptocurrency node wallets can be configured manually to output through the local **SOCKS5 Interface** provided by the client app.

---

## 📊 Checking Your Quota Balance

Because we prioritize user privacy, checking your remaining data footprint or time allowance is handled securely depending on your mode:

- **Anonymous Tokens (JWT RSA):** Your remaining data limit is tracked locally by your client app based on the self-contained contract inside the voucher. It operates entirely on-device without phoning home to log your usage history.
- **Account Tokens (JWT SHA256):** You can view your real-time data sync consumption by logging directly into your personal dashboard on the **Sail Gateway Web Portal**.
