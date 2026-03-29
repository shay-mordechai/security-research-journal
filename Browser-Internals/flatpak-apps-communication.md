# Investigating IPC Boundaries and Sandboxing in Flatpak: The KeePassXC & LibreWolf Case

## 1. Theoretical Background
Modern Linux systems increasingly rely on containerized application formats like Flatpak to enhance security. Flatpak uses features like namespaces and `bubblewrap` to isolate applications into separate sandboxes. 

However, this strict isolation breaks traditional Inter-Process Communication (IPC). Many applications rely on IPC to function together. For example:
* **Password Managers (KeePassXC):** Need to communicate with browser extensions via Native Messaging.
* **Hardware Security Keys (USB/YubiKey):** Browsers need access to `/dev/bus/usb/` or local PC/SC daemons to authenticate.

These communications often happen over **Unix Domain Sockets** — local data channels represented as files in the filesystem (usually under `/run/user/`). When two applications run in separate Flatpak sandboxes, they cannot see or access each other's sockets by default.

## 2. The Investigation: Locating the IPC Socket
To understand why the LibreWolf browser extension cannot communicate with the KeePassXC desktop app, we first need to locate the communication channel (the Unix Domain Socket).

Flatpak applications store their runtime data in isolated directories under `/run/user/1000/app/`. By inspecting the KeePassXC directory on the host, we can identify the active socket:

```bash
shay0129@fedora:~$ ls -l /run/user/1000/app/org.keepassxc.KeePassXC/
srwx------. 1 shay0129 shay0129 0 Mar 28 21:21 org.keepassxc.KeePassXC.BrowserServer
```
*(Note the `s` at the beginning of the permissions string, indicating it is a Socket file).*

## 3. Proving the Sandbox Isolation
To verify that this is a sandbox boundary issue, we must simulate the browser's perspective. We can inject an `ls` command directly into the LibreWolf sandbox to see if it can access the KeePassXC socket:

```bash
shay0129@fedora:~$ flatpak run --command=ls io.gitlab.librewolf-community -l /run/user/1000/app/org.keepassxc.KeePassXC/
F: Not sharing "/dev/bus/usb" with sandbox: Path "/dev" is reserved by Flatpak
ls: cannot access '/run/user/1000/app/org.keepassxc.KeePassXC/': No such file or directory
```

### Findings
This output perfectly demonstrates two critical isolation mechanisms enforced by Flatpak:
1. **Mount Namespaces (IPC Blocked):** The sandbox receives a completely isolated view of the filesystem. It does not get a "Permission Denied" error; rather, the host directory simply does not exist within its namespace. This breaks the Unix Domain Socket communication required by the Native Messaging host.
2. **Device Isolation (Hardware Blocked):** The warning `Not sharing "/dev/bus/usb"` highlights that raw hardware access is restricted, which actively breaks authentication mechanisms like hardware Security USB keys.

## 4. Architectural Solutions
To resolve these isolation barriers without compromising the entire sandbox, granular permissions must be applied:

* **Solving the IPC (KeePassXC) Issue:** We can punch a specific hole in the filesystem boundary using a Flatpak override. This allows the browser to map the specific socket path into its namespace:
  `flatpak override --user --filesystem=/run/user/1000/app/org.keepassxc.KeePassXC io.gitlab.librewolf-community`

## 5. Resolving the Hardware Token (USB Key) Limitation & The Fingerprinting Trade-off
During the `ls` injection, Flatpak explicitly warned about blocking `/dev/bus/usb`. While some smartcards use the PC/SC daemon, modern FIDO2/WebAuthn hardware keys rely on direct USB HID (Human Interface Device) communication. 

To restore hardware token functionality, two steps are required:

**Step 1: Sandbox Hardware Override**
We explicitly grant the sandbox access to the host's devices to allow USB HID communication:
```bash
flatpak override --user --device=all io.gitlab.librewolf-community
```

**Step 2: Browser Engine Configuration**
LibreWolf's strict anti-fingerprinting (RFP - Resist Fingerprinting) defaults actively disable USB token support because hardware access exposes a **high-entropy fingerprinting signal** (אות זיהוי ייחודי). To re-enable it, the following internal engine flags must be modified via `about:config`:
* `security.webauth.webauthn = true`
* `security.webauth.u2f = true`
* `security.webauth.webauthn_enable_usbtoken = true`

### The Security vs. Privacy Conflict
This combination bridges the gap between the OS-level isolated filesystem and the browser's internal security matrix. However, it demonstrates a critical trade-off: **exposing hardware access to restore WebAuthn functionality fundamentally breaks the browser's fingerprinting resistance.** By overriding the sandbox (`--device=all`), we allow the runtime environment to expose identifiable hardware signals to web scripts.

## 6. Comparative Attack Surface Analysis: LibreWolf vs. Chrome & Brave
To contextualize the security posture of LibreWolf, a comparative analysis of both active IPC sockets and hardcoded Flatpak permissions was conducted against Google Chrome and Brave Browser. The goal was to assess the potential impact of a Remote Code Execution (RCE) vulnerability inside the browser sandbox.

### Methodology
To extract the active IPC sockets from within each sandbox, we injected a `cat` command to read the kernel's Unix socket routing table:
```bash
flatpak run --command=cat com.google.Chrome /proc/net/unix > chrome_sockets.txt
flatpak run --command=cat com.brave.Browser /proc/net/unix > brave_sockets.txt
flatpak run --command=cat io.gitlab.librewolf-community /proc/net/unix > librewolf_sockets.txt
```

To extract the hardcoded sandbox permissions (Manifest Analysis), we queried Flatpak:
```bash
flatpak info --show-permissions com.google.Chrome
flatpak info --show-permissions com.brave.Browser
flatpak info --show-permissions io.gitlab.librewolf-community
```

### Findings:
1. **Google Chrome (Data Exposure):**
   Chrome defaults request extensive filesystem access (`xdg-music;xdg-pictures;xdg-videos;xdg-documents`). In the event of an RCE, the attacker gains immediate read/write access to the user's personal files.
2. **Brave Browser (System Integration Risk):**
   Brave requests write access to host application directories (`~/.local/share/applications:create;~/.local/share/icons:create`) and exposes a massive DBus Session Bus surface (including local wallets like `kwalletd6` and screen savers). This increases the risk of persistent host compromise via malicious desktop entries or IPC manipulation.
3. **LibreWolf (Strict Least Privilege):**
   LibreWolf proved to be the most isolated environment. By default, it restricts filesystem access solely to `xdg-download` and severely limits the DBus surface. 

By utilizing LibreWolf and explicitly engineering granular overrides for necessary identity services (KeePassXC IPC and FIDO2 USB tokens), we achieve a functional browsing environment that strictly adheres to the principle of least privilege.


## 7. The Native Messaging Dilemma: Legacy IPC vs. Modern Sandboxing
While attempting to finalize the KeePassXC browser integration, a fundamental architectural conflict was discovered regarding the Native Messaging API.

The browser extension relies on a JSON manifest (`org.keepassxc.keepassxc_browser.json`) to locate the password manager. However, the manifest uses `"type": "stdio"` and points to an executable path on the host (`/var/lib/flatpak/exports/bin/org.keepassxc.KeePassXC`).

### The Architectural Deadlock
1. **Execution Blocked:** A strictly sandboxed application (like our hardened LibreWolf) is structurally prohibited from executing binaries on the host system.
2. **The Security Compromise:** The only way to force this legacy `stdio` execution to work is by granting the sandbox the `flatpak-spawn --host` permission (or `--talk-name=org.freedesktop.Flatpak`). 
3. **Sandbox Destruction:** Granting this permission completely destroys the strict isolation we verified in Section 6. An attacker achieving Remote Code Execution (RCE) within the browser could easily use `flatpak-spawn --host` to execute arbitrary commands on the host machine, bypassing the sandbox entirely.

### Conclusion & The Detection Surface
This research demonstrates that legacy IPC mechanisms relying on standard input/output (stdio) execution are fundamentally incompatible with modern, strict sandbox environments. 

Furthermore, this highlights how **OS-level sandboxing directly influences the browser's fingerprinting surface**. Strict isolation suppresses environment signals, while "punching holes" for legacy IPC or hardware authentication (USB keys) exposes unique host attributes. To maintain both sandbox integrity and privacy, applications must transition to modern isolation-aware mechanisms, such as **DBus WebExtensions Portals**, which facilitate message passing without requiring host execution privileges or broad device exposure.
