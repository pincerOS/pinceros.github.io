---
title: Development
nav_order: 4
parent: Networking
---

# Interfaces

In low-level networking stacks, packet structures must be defined explicitly and serialized into raw byte buffers that can be transmitted over the wire. The reverse—deserialization—is also essential for parsing incoming network traffic.

This section discusses how packets are structured, and how serialization and deserialization are implemented for Ethernet frames and DHCP messages.

---

## ✳️General Structure

Packets are typically defined as `struct`s, where each field corresponds to a region in the binary layout. Here's a typical Ethernet frame structure:
