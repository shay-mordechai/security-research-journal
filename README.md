# Security Research Journal

**Author:** Shay Mordechai  
**Focus:** Vulnerability Research, Reverse Engineering, OS & Browser Internals, Network Security  

Welcome to my central research repository. This space serves as a living journal for my theoretical analyses, vulnerability disclosures, and architectural security studies. While my other repositories focus on hands-on coding, exploit development, and malware reversing labs, this journal focuses on the underlying mechanics of execution flows, protocol logic, and mitigating complex attack vectors.

---

## 📚 Research Index

### 🌐 1. Browser Internals & Memory Exploitation
**[The Era of FullStack Security: The Collapse of Isolation Walls from V8 to Next.js](./Browser-Internals/v8-jit-rwx-exploitation.md)**
* **Topics:** V8 Engine, JIT Compilation, RWX Memory, Control Flow Integrity (CFI) Bypasses, Inline Hooking.
* **Summary:** A deep-dive analysis into how modern Just-In-Time (JIT) compilers in browsers bypass traditional OS memory isolation (W^X / NX bit). The paper explores how allocating RWX memory pages for dynamic execution opens the door for runtime code manipulation and detours, bridging the gap between web logic and low-level instruction pointers.

### 🛡️ 2. Network Protocol Security & Architecture
**[Deterministic TTL Enforcement: Mitigating DNS Rebinding (LANJack) at the Network Edge](./Network-Protocols/dns-rebinding-ttl-mitigation.md)**
* **Topics:** DNS Rebinding, SSRF, TTL Limiting, iptables, Vulnerability Disclosure.
* **Status:** Acknowledged by TP-Link PSIRT & Palo Alto Networks PSIRT.
* **Summary:** Following the "LANJack" campaign, this architectural proposal details a network-layer mitigation against DNS Rebinding exfiltration targeting edge routers. By enforcing a deterministic `TTL=1` limit on management interfaces, the defense breaks the exfiltration chain at the ISP gateway. The paper includes industry feedback on the trade-offs between zero-trust security and L3 VPN usability.

### ⚙️ 3. Compiler Security & Language Internals
**[Macros vs. Functions: Execution Flow & Security Implications](./Compiler-Security/macros-vs-functions-execution-flow.md)**
* **Topics:** Meta-Programming, AST Manipulation, LISP (SBCL) vs. Raku, Assembly Footprints.
* **Summary:** An analytical look at the differences between runtime function calls (Call Stack, JMP instructions) and compile-time macro expansions (Inline expansion, AST integration). This note explores how heavy reliance on macros shifts the binary landscape, neutralizing traditional stack-based overflows while introducing new attack surfaces like compiler encapsulation breakage and macro hygiene collisions.

---

## 🔗 Connect & Learn More
* **Portfolio & Full Write-ups:** [shay-mordechai.github.io](https://shay-mordechai.github.io)
* **LinkedIn:** [Shay Mordechai](https://linkedin.com/in/shay-mordechai)
