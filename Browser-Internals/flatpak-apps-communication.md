# Investigating IPC Boundaries and Sandboxing in Flatpak: The LibreWolf & KeePassXC Case

## Executive Summary (TL;DR)
* **Problem:** Strict OS-level isolation (Flatpak/bubblewrap) breaks essential browser functionality, specifically Native Messaging IPC (KeePassXC) and hardware-based authentication (WebAuthn/USB Security Keys).
* **Action:** Investigated Flatpak mount namespaces, Unix Domain Sockets, and hardware permissions. Engineered granular sandbox overrides and analyzed browser engine configurations (`about:config`) to restore functionality. 
* **Result:** Successfully mapped the trade-off between OS-level sandboxing, functionality, and browser fingerprinting. Demonstrated that exposing hardware for WebAuthn fundamentally breaks fingerprinting resistance (RFP), highlighting a critical attack/detection surface.

---

## 1. Theoretical Background
Modern Linux systems increasingly rely on containerized application formats like Flatpak to enhance security. Flatpak uses features like namespaces and `bubblewrap` to isolate applications into separate sandboxes. 

However, this strict isolation breaks traditional Inter-Process Communication (IPC). Many applications rely on IPC to function together:
* **Password Managers (KeePassXC):** Need to communicate with browser extensions via Native Messaging.
* **Hardware Security Keys (USB/YubiKey):** Browsers need access to `/dev/bus/usb/` or local PC/SC daemons to authenticate.

These communications often happen over **Unix Domain Sockets**. When two applications run in separate Flatpak sandboxes, they cannot see or access each other's sockets by default.

## 2. The Investigation: Locating the IPC Socket
To understand why the LibreWolf browser extension cannot communicate with the KeePassXC desktop app, we first need to locate the communication channel. By inspecting the KeePassXC directory on the host, we identify the active socket:

```bash
shay0129@fedora:~$ ls -l /run/user/1000/app/org.keepassxc.KeePassXC/
srwx------. 1 shay0129 shay0129 0 Mar 28 21:21 org.keepassxc.KeePassXC.BrowserServer
```

## 3. Proving the Sandbox Isolation
To verify the boundary, we inject an `ls` command directly into the LibreWolf sandbox to see if it can access the socket:

```bash
shay0129@fedora:~$ flatpak run --command=ls io.gitlab.librewolf-community -l /run/user/1000/app/org.keepassxc.KeePassXC/
F: Not sharing "/dev/bus/usb" with sandbox: Path "/dev" is reserved by Flatpak
ls: cannot access '/run/user/1000/app/org.keepassxc.KeePassXC/': No such file or directory
```

**Findings:**
1. **Mount Namespaces (IPC Blocked):** The sandbox receives an isolated view of the filesystem. The host directory simply does not exist within its namespace.
2. **Device Isolation (Hardware Blocked):** The warning `Not sharing "/dev/bus/usb"` highlights that raw hardware access is restricted.

## 4. Architectural Solutions
To resolve these isolation barriers without compromising the entire sandbox, granular permissions must be applied. We punch a specific hole in the filesystem boundary using a Flatpak override:
```bash
flatpak override --user --filesystem=/run/user/1000/app/org.keepassxc.KeePassXC io.gitlab.librewolf-community
```
* **Impact:** Restored KeePassXC-LibreWolf communication without compromising core sandbox isolation.

## 5. Resolving the Hardware Token (USB Key) & The Fingerprinting Trade-off
Modern FIDO2/WebAuthn hardware keys rely on direct USB HID communication. To restore this:

**Step 1: Sandbox Hardware Override**
```bash
flatpak override --user --device=all io.gitlab.librewolf-community
```

**Step 2: Browser Engine Configuration**
LibreWolf's strict anti-fingerprinting (RFP) actively disables USB token support because hardware access exposes a high-entropy fingerprinting signal. We modified internal engine flags (`about:config`):
* `security.webauth.webauthn = true`
* `security.webauth.u2f = true`
* `security.webauth.webauthn_enable_usbtoken = true`

**The Security vs. Privacy Conflict:**
This bridges the gap between the OS-level sandbox and the browser's security matrix. However, it demonstrates a critical trade-off: **exposing hardware access to restore WebAuthn fundamentally breaks the browser's fingerprinting resistance.** By overriding the sandbox, we allow the runtime environment to expose identifiable hardware signals to web scripts.

## 6. Comparative Attack Surface Analysis: LibreWolf vs. Chrome & Brave
To contextualize LibreWolf's posture, we extracted active IPC sockets and hardcoded Flatpak permissions (`flatpak info --show-permissions`) to assess the potential impact of a Remote Code Execution (RCE) inside the sandbox.

1. **Google Chrome (Data Exposure):** Defaults request extensive filesystem access (`xdg-documents`, etc.). An RCE grants immediate read/write access to personal files.
2. **Brave Browser (System Integration Risk):** Requests write access to host application directories and exposes a massive DBus Session Bus surface, increasing the risk of persistent host compromise.
3. **LibreWolf (Strict Least Privilege):** Restricts filesystem access solely to `xdg-download` and severely limits the DBus surface. 

## 7. The Native Messaging Dilemma & Detection Surface
The browser extension relies on a JSON manifest (`"type": "stdio"`) pointing to an executable host path. 
* **The Deadlock:** A strictly sandboxed application cannot execute host binaries unless granted `flatpak-spawn --host`.
* **The Compromise:** Granting this permission destroys the sandbox. An attacker achieving RCE could execute arbitrary commands on the host.

### Conclusion
Legacy IPC mechanisms relying on `stdio` execution are fundamentally incompatible with strict modern sandboxes. 

Furthermore, **OS-level sandboxing directly influences the browser's fingerprinting surface**. Strict isolation suppresses environment signals, while "punching holes" for legacy IPC or hardware authentication (USB keys) exposes unique host attributes. Applications must transition to modern isolation-aware mechanisms, such as **DBus WebExtensions Portals**, to maintain both sandbox integrity and privacy.
