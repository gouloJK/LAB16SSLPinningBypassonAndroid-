# LAB 16 – SSL Pinning Bypass on Android  
## *Objection + Frida + Burp Suite — Full Deep Dive*

> Omayma El Yamani   

---

## 🎯 Lab Objectives

This lab teaches you how to **defeat SSL Pinning** in Android applications using dynamic instrumentation.

By the end, you will be able to:
- Deploy **Frida** and **Objection** on a test environment
- Intercept HTTPS traffic from **pinned** Android apps
- Understand **why** SSL Pinning breaks traditional MITM
- Master both `spawn` and `attach` injection methods

---

##  Theoretical Background

### What is SSL Pinning?

Normally, an Android app trusts any certificate signed by a CA in the system store.  
**SSL Pinning** changes this: the app **hardcodes** the expected server certificate or public key.

If the server presents a different certificate (like Burp's), the app **rejects** the connection.

### How does Objection bypass it?

Objection uses Frida to inject JavaScript into the app's memory at runtime.  
It **hooks** the Java methods responsible for certificate validation:

| Class | Method Hooked | Effect |
|-------|---------------|--------|
| `TrustManager` | `checkServerTrusted()` | Always returns `true` |
| `OkHttp CertificatePinner` | `pin()` | Disables pinning logic |
| `X509TrustManagerExtensions` | `checkServerTrusted()` | Forces acceptance |

The app still sees a certificate — but now it **accepts any** certificate.

---

##  Required Tools

| Tool | Purpose |
|-------|---------|
| **Frida** | Dynamic instrumentation engine |
| **frida-server** | Runs on Android, bridges PC and device |
| **Objection** | Frida wrapper with pre-made commands |
| **Burp Suite** | HTTP/HTTPS proxy for traffic inspection |
| **ADB** | Android Debug Bridge for device communication |

---

##  Screenshot Walkthrough

### Verifying Frida Installation

<img width="1074" height="169" alt="image" src="https://github.com/user-attachments/assets/f6d7e026-3948-407f-91d1-87c4b643bb3a" />

Frida is correctly installed on the PC. The version is 17.9.6.
The Android device must run exactly the same version of frida-server, otherwise communication will fail.

### Testing Objection
<img width="567" height="332" alt="image" src="https://github.com/user-attachments/assets/ecb91e3f-3501-41eb-a653-c7741bcbf01c" />

```bash
objection --help
Usage: objection [OPTIONS] COMMAND [ARGS]...
Runtime Mobile Exploration by @leonjza from @sensepost
```

What this tells us:
- Objection is accessible from the command line.
- The help menu confirms it's ready to use.

### Listing Processes on Device
<img width="568" height="361" alt="image" src="https://github.com/user-attachments/assets/dba855ed-3881-4063-8826-6268879fb063" />

```bash
C:\Users\pc\Downloads> frida-ps -U
PID    Name
6989   Diva
6747   Messaging
4184   RoomMVVMDemo
```

What this tells us:
- The command `frida-ps -U` lists all running processes on the USB-connected Android device.
- We see Diva (PID 6989) — a vulnerable training app often used for SSL pinning exercises.
- This will be our target.

### Installed Certificates on Android
<img width="350" height="261" alt="image" src="https://github.com/user-attachments/assets/d5b00a3f-f078-4fbb-bcfd-e17a494c3574" />

What you see:
Under Settings → Security → Trusted Credentials → User:
- mitmproxy certificate
- PortSwigger CA (Burp Suite certificate)

Why this matters:
- For Burp to decrypt HTTPS traffic, Android must trust Burp's CA.
- These certificates are installed in the User store.
- *(Note: Apps targeting Android 7+ ignore user CAs by default — but Objection bypasses this restriction.)*

### Burp Suite CA Download Page
<img width="349" height="120" alt="image" src="https://github.com/user-attachments/assets/386e2171-9786-4dbd-824a-11a0a64123b6" />

```
Burp Suite Community Edition
CA Certificate
```

What this shows:
Burp's built-in web server at http://burp. From here, you download the cacert.der file to install on Android.

Step-by-step:
1. Open Chrome on Android
2. Go to http://burp
3. Click "CA Certificate"
4. Save and install via Android settings

### Manual Proxy Configuration
<img width="190" height="287" alt="image" src="https://github.com/user-attachments/assets/8b0f56f6-c1be-436b-b819-cbfc8b2a61d8" />

```yaml
Network: AndroidWifi
Proxy: Manual
Proxy hostname: 10.0.2.2
Proxy port: 8080
```

What this shows:
- The Android device is configured to route all HTTP/HTTPS traffic through 10.0.2.2:8080.

Why 10.0.2.2?
- On Android emulators, 10.0.2.2 is a special alias for the host machine's 127.0.0.1.
- On a physical device, you would enter your PC's local IP (e.g., 192.168.1.42).

### SSL Error Before Pinning Bypass
<img width="180" height="283" alt="image" src="https://github.com/user-attachments/assets/b0a512cb-7cd8-44eb-af16-51bcf0bcd009" />

```text
Your connection is not private
NET:ERR_CERT_AUTHORITY_INVALID
```

What this tells us:
- Even with Burp's CA installed and proxy configured, the connection is rejected.

Why?
- The app (or browser) has SSL Pinning enabled.
- It sees Burp's certificate instead of the real server's certificate → blocks the connection.

This is the problem we will solve with Objection.

## Step-by-Step Execution

### Phase 1: Deploy frida-server on Android

```bash
# Push the frida-server binary to the device
adb push frida-server /data/local/tmp/

# Make it executable
adb shell chmod 755 /data/local/tmp/frida-server

# Launch the daemon
adb shell "/data/local/tmp/frida-server -l 0.0.0.0 &"

# Forward ports for better stability
adb forward tcp:27042 tcp:27042
adb forward tcp:27043 tcp:27043
```

Explanation:
- `-l 0.0.0.0` : Listen on all network interfaces
- `&` : Run in background
- Ports 27042/27043 : Default Frida communication channels

### Phase 2: Configure Burp Suite

Open Burp Suite → Proxy tab → Options

Add a new listener:
- Bind to port: 8080
- Bind to address: 0.0.0.0 (accept connections from any IP)
- Ensure "Intercept" is off (or on, depending on your workflow)

### Phase 3: Set Proxy on Android Device

For emulator (10.0.2.2):
- Settings → Wi-Fi → Long-press AndroidWifi → Modify network → Proxy: Manual →
- Hostname: 10.0.2.2 | Port: 8080 → Save

For physical device (PC's local IP):
- Find your PC's IP: `ipconfig` (Windows) or `ifconfig` (Linux/Mac)
- Use that IP instead of 10.0.2.2

### Phase 4: Install Burp CA on Android

**Method 1 (easiest):**
1. Open Chrome on Android
2. Go to http://burp
3. Click "CA Certificate"
4. When prompted, name it BurpCA and confirm

**Method 2 (manual):**
```bash
adb push cacert.der /sdcard/Download/
# Then install via Settings → Security → Install from storage
```

### Phase 5: Launch Objection and Disable SSL Pinning

**🔹 Spawn Method (recommended)**

```bash
objection -g diva explore --startup-command "android sslpinning disable"
```

What happens:
- Objection starts the diva app from scratch
- Immediately injects Frida hooks
- Runs `android sslpinning disable` before the app's pinning code executes
- Opens an interactive console

**🔹 Attach Method (if spawn crashes)**

```bash
# First, manually open the app on your device
# Then run:
objection -g diva explore
```

Once inside the Objection console:

```
diva $ android sslpinning disable
```

Expected output:

```
[✓] SSL Pinning disabled for OkHTTP
[✓] SSL Pinning disabled for TrustManager
[✓] SSL Pinning disabled for X509TrustManagerExtensions
```

### Phase 6: Verify the Bypass

**Check 1 – Burp Suite:**
- Go to Proxy → HTTP History.
- You should see HTTPS requests from diva appearing.
- No SSL errors in the Alerts tab.

**Check 2 – Android device:**
- The app loads normally.
- No "certificate invalid" popups.
- Login and API calls work.

**Check 3 – Objection console:**
- You may see log messages confirming hooks are active.

---

## 🐞 Common Issues and Solutions

### Issue 1: frida-ps -U shows no devices

**Cause:** ADB not connected or USB debugging off.

**Fix:**
```bash
adb devices
```

If device not listed, check cable, enable USB debugging, approve RSA fingerprint.

### Issue 2: unable to connect to remote frida-server

**Possible causes:**
- frida-server not running
- Version mismatch between PC and Android
- Firewall blocking port 27042

**Fix:**
```bash
adb shell ps | grep frida   # Check if running
frida --version              # Compare with server version
adb forward tcp:27042 tcp:27042
```

### Issue 3: App crashes when using spawn method

**Cause:** Some apps implement anti-debugging or initialize pinning too early.

**Fix:** Use attach method instead:
```bash
# Open app manually, then:
objection -g diva explore
android sslpinning disable
```

### Issue 4: No traffic appears in Burp

**Possible causes:**
- Proxy not configured correctly
- App ignores system proxy (uses raw sockets or Cronet)
- Firewall blocking port 8080

**Fix:**
- Verify proxy settings on Android
- Test with browser: visit http://example.com — should appear in Burp
- If app bypasses proxy, use iptables on rooted device or VPN-based MITM

### Issue 5: Error persists after android sslpinning disable

**Cause:** App uses multiple HTTP clients, or native SSL (BoringSSL/OpenSSL).

**Fix:**
```bash
# Search for all networking classes
android hooking search classes okhttp
android hooking search classes trust
# Then target specific classes
```

For native SSL, Objection alone is insufficient — use a custom Frida script.

---

## 📊 Method Comparison: Spawn vs Attach

| Criteria | Spawn | Attach |
|----------|-------|--------|
| App starts automatically | ✅ Yes | ❌ No (must start manually) |
| Hooks before pinning init | ✅ Yes | ⚠️ Depends on timing |
| Works if app has anti-debug | ❌ Often fails | ✅ More reliable |
| Best for | Simple apps, early hooking | Complex apps, crash-prone targets |

**Recommendation:** Start with spawn. If it crashes → switch to attach.

---

## 🧹 Cleanup Instructions

After finishing the lab:

```bash
# Stop frida-server
adb shell killall frida-server

# Remove forwarded ports
adb forward --remove-all

# Remove Burp CA from Android (optional)
Settings → Security → User credentials → Select and remove

# Disable proxy
Settings → Wi-Fi → Modify network → Proxy → None

# Close Burp Suite or turn off interception
```

---

## 🎓 Key Takeaways

- SSL Pinning prevents classic MITM even with a trusted CA installed.
- Objection simplifies Frida-based bypasses with one command: `android sslpinning disable`.
- Spawn vs attach gives you flexibility depending on app behavior.
- The CA must still be installed — Objection disables pinning, not certificate validation.
- Native SSL implementations require custom Frida scripts (beyond Objection).
