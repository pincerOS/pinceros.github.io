---
title: Development
nav_order: 1
parent: Networking
---

# Development

---

## Initial Planning and Inspiration

Without device drivers and interfaces, a networking stack is ultimately limited. While it can process packets and manage protocol logic, it lacks utility without a way to interface with actual network hardware or emulation. To avoid this during early development, I built the stack on top of Linux user-space, using `tun`/`tap` interfaces to simulate the environment.

### Packet Representation 

One of the most influential references was [smoltcp](https://github.com/smoltcp-rs/smoltcp), a minimalist TCP/IP stack written in Rust. They are one of the most well-maintained/fleshed-out implementations for a user networking stack in Rust. They separate protocol logic and with memory-safe packet parsing, which I stole. In particular, I used their design setup of `Packet` structs that wrap raw byte buffers and provide accessor methods using bit-level manipulation. It ensures safety without sacrificing performance, and it cleanly abstracts protocol headers like IPv4, TCP, and UDP.

### Interface and State Management

[rust-user-net](https://github.com/ykskb/rust-user-net) informed how I organized global state. Rather than having scattered socket and address state across modules, rust-user-net encapsulates everything inside a central `Interface` structure. I followed a similar idea: all sockets, local addresses, and the routing table are managed within a single `Interface`, making it easier to pass around and manage shared state. This structure became the core API that applications interact with.

### TAP Setup and Protocol Flow

[microps](https://github.com/pandax381/microps) provided clarity on the basic structure of a network stack. Its use of the `tap0` device for input/output, layering of protocols, and socket management provided a clean mental model. I borrowed their step-by-step packet handling pipeline to shape how incoming packets are demultiplexed (e.g., based on EtherType, IP protocol, and ports) and how outbound packets are constructed before writing to the interface. Although written in C, the general setup on Linux was very useful to getting me started.

---

## Learning Rust and Locking Challenges

This was my first project using Rust, and learning the language alongside systems-level design presented a steep learning curve. One of the major hurdles was dealing with **concurrent access to sockets**, especially when it came to blocking operations like waiting for data to arrive.

At first, I used a **single global lock** on the socket table (inside `Interface`). This worked for simple cases, but quickly led to deadlocks: for example, a thread waiting to `recv()` on a socket would hold the lock, preventing another thread from delivering incoming packets to that same socket.

To solve this, the design had to use **finer-grained locking**. Each socket must have its own internal mutex, allowing `recv()` to block independently without preventing packet delivery. The global socket table uses a read-write lock only for insertion/removal, not for regular access.

```
+-------------------------------------------+
|              Original Design              |
+-------------------------------------------+
| Global Mutex over all interface.sockets   |
|                                           |
| [recv()] --> holds lock                   |
|  meanwhile...                             |
| [packet arrives] --> tries to lock --> ❌ |
| DEADLOCK: recv holds the global lock      |
+-------------------------------------------+


+-------------------------------------------+
|              Improved Design              |
+-------------------------------------------+
| RwLock on socket table (for add/remove)   |
| Each socket has its own Mutex             |
|                                           |
| [recv()] --> locks socket only            |
| [packet arrives] --> locks same socket    |
| Both proceed independently                |
+-------------------------------------------+
```

This taught me a lot about concurrency in Rust, the `Arc<Mutex<_>>` pattern, and the importance of designing for lock granularity in systems with blocking operations.


