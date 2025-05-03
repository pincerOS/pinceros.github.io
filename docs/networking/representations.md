---
title: Packet Representations
nav_order: 2
parent: Networking
---

# Packet Representations

This module defines the core **protocol-level packet structures** used by the networking stack. It provides a unified way to serialize and deserialize common protocols (Ethernet, IPv4, TCP, UDP, etc.), with fields and conventions inspired by existing stacks like [smoltcp](https://github.com/smoltcp-rs/smoltcp).

It is designed for modular use — each protocol module (`ethernet`, `ipv4`, `tcp`, etc.) exports its own strongly typed representations and conversion methods. Together, they form the backbone for encoding, decoding, and inspecting packet data throughout the stack.

---

## Design Goals

* **Protocol Isolation**: Each protocol is self-contained and exposes only the relevant fields and logic.
* **Structured Parsing**: Like smoltcp, packets are parsed from raw byte slices using type-safe structs with field-level accessors.
* **Flexible Composition**: Designed so higher-level protocols (e.g., TCP over IPv4 over Ethernet) can layer on top of one another cleanly.

---

## Error Handling

Usage for the user should return errors on a failed deserialization. The main
way users will interact with the representations is by calling
`serialize`/`deserialize`. This means that when we have a malformed packet,
`deserialize` should inform the user by returning an `Error`.

---

## Protocol Modules

Each protocol is defined in its own file and module, documented below:

### `ethernet`

Represents Ethernet II frames. Includes fields like:

* Destination and source MAC address
* EtherType (IPv4, ARP, etc.)
* Frame payload

### `arp`

Represents ARP packets. Fields include:

* Hardware and protocol type
* Operation (request/reply)
* Sender and target hardware/IP addresses

### `ipv4`

Represents IPv4 packets. Fields include:

* Version, IHL, total length
* Source and destination IP addresses
* Protocol (TCP, UDP, ICMP)
* Payload

### `icmp`

Represents ICMP messages such as:

* Echo request/reply (ping)
* Destination unreachable
* Time exceeded

Provides high-level enums to match against message types.

### `udp`

Represents UDP datagrams, with:

* Source and destination ports
* Length and checksum
* Payload

### `tcp`

Represents TCP segments. Contains:

* Port numbers, sequence/ack numbers
* Control flags (SYN, ACK, FIN, etc.)
* Window size, checksum
* Payload (after header and options)

### `dns`

Represents basic DNS packets, including:

* Header fields (ID, flags, question count, etc.)
* Question and answer sections
* No full parsing of all record types yet (TODO)

### `http`

Simple and incomplete representation of HTTP requests/responses. Useful for debugging or application-level testing.

* Method (GET, POST, etc.)
* Path, headers, body
* Minimal parsing

### `dhcp`

Represents DHCP messages, including:

* Message type (DISCOVER, OFFER, etc.)
* Client hardware address, transaction ID
* Option list

Used during IP address configuration over UDP.

---

## Device Module

The `dev` module defines the `Device` trait, which provides a consistent interface for sending and receiving raw bytes from network interfaces (e.g., `tap0`). It is used by higher-level stack logic to bridge between the OS interface and the packet representations defined here.


