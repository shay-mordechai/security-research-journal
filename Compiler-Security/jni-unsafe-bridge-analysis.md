# JNI as an (Un-)Safe Bridge: Memory Isolation and Execution Boundaries

**Author:** Shay Mordechai
**Date:** March 26, 2026

## The Paradox of High-Level to Low-Level Programming
A programming language is essentially a way to manage memory smartly so the CPU can execute it. The memory space the CPU knows is just the RAM. Every command and variable in a program gets a virtual address from the operating system, which maps it to a physical address in the RAM.

In C, the programmer has almost direct access to the RAM using pointers to these virtual addresses, and can move forward or backward in memory. Java, on the other hand, prevents this by managing memory through an intermediary—the JVM. It uses Metadata labels for each variable and gives the programmer only a limited reference, preventing them from freely traversing the memory.

So supposedly, High-Level languages are very secure against Buffer Overflow. But this isolation comes at a high cost: every variable carries Metadata along with the data itself, requiring much more memory. This hurts the efficiency of the language's memory management. Therefore, many High-Level languages "cheat" a bit and allow bridging via FFI (Foreign Function Interface) to Low-Level languages (especially C) to regain the lost performance (ביצועים שאבדו).

## The Illusion of Security
In Java, for example, this FFI bridge is called JNI (Java Native Interface). It is the protocol that translates Java's structured memory management to C's economical memory management.

The JNI essentially sheds all the protections of the high-level language, like the Garbage Collector (as a result of removing the Metadata). It does not provide alternative protections; instead, it relies on the programmer to manually insert C-language protections (such as Canary values to prevent stack smashing).

Thus, the High-Level developer thinks they are programming in a safe language with built-in protections and "falls asleep on watch" (נרדם בשמירה) at the most critical time—when the language strips its protections, drops to Low-Level via C, and is left completely exposed. This demonstrates that today, programming in high-level languages is driven more by development convenience than by the inherent security of the language itself.

## The Semantic Contradiction in Passing Strings via JNI
The problem goes deeper than just losing the Garbage Collector. When data crosses the JNI bridge, it suffers from semantic dissonance (דיסוננס סמנטי).

The JVM might validate a string and approve it as safe because it checks its explicit length. But when that exact same data crosses over to C, C interprets it differently—it stops reading as soon as it identifies a Null character (`\0`). 

We are left with a massive architectural flaw: two systems are looking at the exact same bits in memory, but understanding them differently. Attackers specifically target this blind spot (שטח מת) on the Native side.

## The Compilation Paradox
If C is so dangerous, why not simply add a sanitization stage during compilation? In that scenario, the compiler would automatically insert bounds-checking before every memory access to ensure the program isn't writing outside allocated memory.

But here lies the developer's paradox: they crossed the JNI bridge to C specifically to escape Java's performance penalty. If they activate sanitization mechanisms in C to remain safe, they essentially reintroduce the exact same CPU performance hit they tried to escape! They lose on both fronts.

## A Full Transition to Rust? Not Necessarily.
We recently saw reports of internal research at Microsoft aiming to rewrite infrastructure code (like in Windows 11) in Rust. However, it seems a full transition of such scale is neither strictly worthwhile nor feasible. While it would prevent Buffer Overflow and memory leak vulnerabilities, it could introduce other execution flow issues. As discovered recently with a vulnerability in the Windows kernel's Rust component, the code might simply crash (Panic / DoS) whenever a memory issue occurs. Sometimes the usability, convenience, and stability of the system outweigh the "security at all costs" approach.

## My Proposed Solution: Building Secure Tunnels
The more I researched this, the more I concluded that the industry shouldn't rush to throw away decades of C development. The solution isn't necessarily a full transition from C to Rust, but rather keeping the C libraries—and securing the bridges leading to them.

My proposal is that every transition from Java to C should pass through a Rust Tunnel. Rust will offer memory safety at compile-time and sanitize the data, and only then pass it safely to C.

I implemented an identical architectural concept in a project I recently built: **Data Hop Firewall**. It is a WebAssembly filter I wrote in Rust for Envoy Proxy, positioned as a network buffer between the Backend and Frontend containers. 

Its goal is to act as a smart gatekeeper at the application layer (Layer 7): it scans the traffic, sanitizes it, and prevents sensitive data leakage. For example, it prevents API Over-fetching that leaks through RSC (React Server Components) or the exposure of ASP.NET Stack traces to the user's browser. It achieves this by redacting sensitive data right before it leaves the server boundaries, enforcing a Zero-Trust model.

Just as the Envoy filter protects network boundaries from the logical errors of Web developers (who pull excess data and rely on the Client to hide it), so should Rust 'tunnels' in JNI protect memory boundaries from the management errors of C developers.

## The Future
The JNI bridge isn't going anywhere. We still need High-Level languages for fast and convenient development. But the future of the security world isn't about relying on the dark tunnels of C; it is about building secure, Rust-based boundaries that do not compromise on speed.

Ultimately, should security not be part of the secured entity itself, but rather an accompanying component? Will the day come when security is an integral part of the language, or will it always be an external layer attached to the product?
