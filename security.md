# Breaking the Sandbox
## Web-to-Native Bridge & Security

> [!WARNING]
> **Engineering Hazard:** Orion is built with React Native/Web technologies.
> Standard web apps cannot install APKs, uninstall system apps, or revoke permissions.
> **We cheated.**

---

### 1. The Worker Core (Thread Offloading)

**The Bottleneck:**
Parsing 10,000 JSON objects and version strings in Javascript on the UI thread kills performance. The app would freeze at 60fps.

**The Solution:**
The Main UI thread is strictly for **Rendering**.
We spawn a dedicated **Web Worker** context for logic.

*   **UI Thread:** "Here is a list of data. Draw it."
*   **Worker Thread:** "I am sorting 5,000 apps by date, comparing version hashes against the binary manifest, and filtering by 'Games'. Here is the result."

**Result:**
Orion runs at 120Hz on flagship devices while doing heavy crypto-verification in the background.

---

### 2. Shizuku Integration (The System Guardian)

**The Problem:**
Android restricts standard apps. You can't just "Delete Facebook" (System App) or "Grant Permission X" without Root.

**The Architecture:**
Orion implements a Bridge to **Shizuku**.
Shizuku is a tool that allows normal apps to access system APIs via the `adb` (Android Debug Bridge) process.

*   **Silent Install:** Install APKs in the background without user prompts.
*   **Permission Revoke:** Revoke sensitive permissions from other apps instantly.
*   **System App Debloater:** Disable or uninstall pre-installed carrier bloatware.

**Result:**
Orion behaves like a Rooted app manager, allowing silent installs and system debloating, without the user actually needing to root their phone.

> [!CAUTION]
> **Security Implication:**
> Shizuku grants `adb` level permissions (`com.android.shell`). 
> - **Can:** Install/Uninstall apps, Grant runtime permissions, Clear app data.
> - **Cannot:** Touch `/system` partition, modify Kernel, brick the device.
> This makes it 10x safer than Root, but still powerful enough to be dangerous if misused. Orion sandboxes these commands strictly to Package Management operations.


---

### 3. Orion Sentinel (Distributed Intelligence)

**The Privacy Dilemma:**
We want to scan apps for malware/trackers.
*   *Usually:* You upload the file hash to a central server (VirusTotal).
*   *Privacy Risk:* The server knows what every user has installed.

**The Fix:**
We download the Threat DB to the user.

1.  **The Compilation:**
    We aggregate tracker signatures and malware hashes into huge JSON shards (`shard_a.json` ... `shard_z.json`).
2.  **The Distro:**
    Orion downloads the specific shard relevant to the app being scanned.
3.  **The Local Scan:**
    Comparison happens **On Device**.

**Impact:**
Orion can tell you if an app has 50 trackers or a known malware signature **without ever telling our servers what apps you have.**
Your app list never leaves your device.
