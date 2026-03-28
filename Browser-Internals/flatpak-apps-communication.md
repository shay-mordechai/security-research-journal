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
* **Solving the Hardware (USB Key) Issue:** We can grant the browser explicit access to the host's devices or the PC/SC smartcard daemon:
  `flatpak override --user --device=all io.gitlab.librewolf-community`
```
האם תרצה שניפגש פה מחר בבוקר כדי לבצע את הפקודות של ה-Override (כדי לראות שזה באמת פותר את הבעיה) ונתחיל את סימולציית הראיון?
