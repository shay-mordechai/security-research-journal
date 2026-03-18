# Deterministic TTL Enforcement: Mitigating DNS Rebinding (LANJack) at the Network Edge

**Author:** Shay Mordechai  
**Focus:** Network Security, Vulnerability Disclosure, OS Internals  
**Status:** Acknowledged by TP-Link PSIRT & Palo Alto Networks PSIRT (Feature Request / Hardening Proposal)

## 📝 Overview
Following the exposure of the "LANJack" campaign, which highlighted how attackers exploit unsecured DNS servers to turn victim browsers into proxies (targeting Mobile and IoT devices), I began researching architectural mitigations. 

In cloud environments, AWS successfully mitigated similar Server-Side Request Forgery (SSRF) vectors in their IMDSv2 standard by requiring a token and restricting incoming packet Time-To-Live (TTL) to `1`. This research proposes applying a similar deterministic network-layer defense to consumer and enterprise edge routers to neutralize DNS Rebinding data exfiltration.

## 🔬 The Threat: DNS Rebinding Exfiltration
DNS Rebinding allows a remote attacker to bypass the Same-Origin Policy (SOP). By rapidly changing the IP address associated with a malicious domain from an external IP to a local one (e.g., `192.168.1.1`), the attacker's JavaScript can force the victim's browser to send requests to the local router's management interface. 

With default Linux kernel configurations, the outbound HTTP response from the router has a standard `TTL=64`. This allows the exfiltrated sensitive data to traverse multiple routing hops across the internet back to the attacker.

## 🛡️ The Proposed Mitigation: TTL=1 Enforcement
The proposal introduces an `iptables`/`nftables` rule on the router's `OUTPUT` chain specifically for the Web Management Interface (Ports 80/443), setting the outbound `TTL=1`.

**How it works:**
1. The victim's browser (Hop 1) requests data from the router.
2. The router responds with `TTL=1`.
3. The browser successfully receives the data (since it is physically/logically adjacent).
4. If the malicious script attempts to exfiltrate this response payload to an external C2 server, the packet must pass through the ISP gateway (Hop 2).
5. The gateway decrements the TTL to `0`, immediately dropping the packet and returning an ICMP Time Exceeded message. The exfiltration chain is deterministically broken.

## ⚙️ Architectural Trade-offs & Industry Feedback
I submitted this proposal and accompanying PCAP proofs to major networking vendors. Both **TP-Link** and **Palo Alto Networks** validated the technical merit of the mitigation, but highlighted the classic tension between Security and Usability.

### The VPN & Mesh Routing Conundrum
Setting `TTL=1` creates challenges for complex topologies:
* **L3 VPNs (TUN Interfaces):** When an administrator connects home via a VPN (TUN mode), they exist on a different virtual subnet. When the router replies to the VPN client, it performs a subnet routing operation, decrementing the TTL. A packet starting with `TTL=1` will drop before reaching the remote client.
* **L2 Bridges (TAP Interfaces):** Switching the VPN to TAP mode (Bridging) avoids the subnet hop by resolving directly to the MAC address. However, TAP mode is often unsupported on modern mobile devices (Android/iOS) due to strict OS-level security constraints.
* **Mesh Networks / Range Extenders:** Multi-level routing scenarios require more than one hop internally.

### PSIRT Responses
> **TP-Link Product Security Team:**
> *"We've concluded that while setting TTL=1 for web server traffic effectively prevents session exfiltration, the impact on core user scenarios—like multi-level routing and mesh topologies—is high. Therefore, we will hold off on implementation for now. However, we'll keep this under consideration as a future opt-in security hardening feature."*

> **Palo Alto Networks PSIRT:**
> *"We agree that providing customers with a toggle to customize TTL values could be a useful feature for specific edge-case deployments... If you are an active customer, we encourage you to open a support ticket so our product team can track this as a formal feature request."*

## 📊 Proof of Concept (PCAP Analysis)
During the disclosure process, three network states were captured to demonstrate the defense mechanism (available upon request):
1. **`baseline_vulnerable.pcap`**: Default behavior (`TTL=64`). Management interface packets successfully traverse multiple routing hops.
2. **`mitigation_ttl1.pcap`**: Enforced `TTL=1`. L2-adjacent communication remains functional, but routing beyond the immediate segment fails.
3. **`dns_rebinding_mitigation_chain.pcap`**: An exfiltration attempt where the packet is sent with `TTL=1` and is immediately dropped by the gateway, breaking the DNS Rebinding attack chain.

## 💡 Conclusion
Security is rarely about simply "locking the door"; it requires continuous risk management to ensure the product remains functional for the end-user. While deterministic TTL limiting is highly effective against exfiltration, its impact on L3 routing necessitates its implementation as an **"Isolated Management Mode"** or an "Opt-in" hardening feature for advanced users, rather than a default standard.
