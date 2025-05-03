---
title: Interfaces
nav_order: 3
parent: Networking
---

# Interfaces

The `iface` module is the **central coordinator** for packet dispatching and reception. It bridges low-level packet parsing with high-level socket handling and protocol behavior. Each supported protocol (Ethernet, IPv4, TCP, etc.) has its own dedicated **send** and **receive** function. These functions are invoked either directly by the interface (e.g., from a device callback) or internally after parsing a lower-level protocol.

---

## Responsibilities

* Parsing incoming frames at each protocol layer
* Dispatching packets to appropriate socket handlers (e.g., TCP sockets, UDP ports)
* Generating ICMP responses as needed
* Managing device callbacks and entry points (e.g., `ethernet_recv`)

---

## Packet Flow

```
[Device] → ethernet_recv_frame
          ↓
     [Ethernet Frame]
          ↓
     match EtherType:
     - IPv4 → ipv4_recv_packet
     - ARP  → arp_recv_packet
     ↓
[IPv4 Packet]
     match Protocol:
     - TCP → tcp_recv_packet
     - UDP → udp_recv_packet
     - ICMP → icmp_recv_packet
     ↓
[Sockets / Responses]
```

```
[Application Layer]
      ↓
[Socket Layer]
      ↓
[Protocol Send (e.g., tcp_send, udp_send)]
      ↓
[ipv4_send_packet]
  - Lookup destination IP
  - Determine next hop:
      if dest_ip in local_subnet:
          next_hop_ip = dest_ip
      else:
          next_hop_ip = gateway_ip
      ↓
  - Check ARP Table:
      if ARP[next_hop_ip] exists:
          mac = ARP[next_hop_ip]
          ↓
          [ethernet_send_frame]
            - Build Ethernet frame with dest MAC
            - Send over device
      else:
          ↓
      [enqueue to pending ARP queue]
          ↓
      [send_arp_request(next_hop_ip)]
```

---

## Key Components

### `ethernet_recv`

* Entry point for incoming packets.
* Registered as a **callback for USB device reception** or virtual device (e.g., TAP).
* Parses the raw buffer into an `EthernetFrame` and routes based on `EtherType`.

### `ipv4_recv`

* Parses an `Ipv4Packet`.
* Dispatches to:

  * `tcp_recv` if protocol is TCP
  * `udp_recv` if protocol is UDP
  * `icmp_recv` if protocol is ICMP
* May drop invalid packets or send ICMP errors if unrecognized.

### `tcp_recv`, `udp_recv`

* Responsible for:

  * Validating TCP/UDP headers and checksum
  * Locating a **bound socket** for the destination port
  * Enqueuing data for socket-level processing
* If no socket is bound, TCP may send a **RST** and UDP may silently drop or trigger **ICMP Port Unreachable**.

### `icmp_recv`

* Handles ICMP packet parsing.
* May produce:

  * Echo replies to echo requests (ping)
  * Destination unreachable messages
  * Time exceeded messages (for TTL expiration)
* ICMP responses are constructed and sent using `icmp_send`.

---

## Sending Functions

Each protocol layer also defines a `*_send` function that:

* Accepts a preconstructed packet or payload
* Adds necessary headers (e.g., IP, UDP, Ethernet)
* Computes checksums
* Sends the result via the registered `Device`

Sending cascades **upward**: e.g., `tcp_send → ipv4_send → ethernet_send`.

---

## Socket Integration

TCP and UDP layers are integrated with the **socket subsystem**, which tracks active ports and connection state:

* When a packet is received, the `iface` module checks whether a matching socket exists.
* If so, the packet is forwarded to the socket’s receive buffer.
* If not, the packet is dropped or responded to (e.g., TCP RST).

---

## Notes

* The stack assumes the interface's underlying `Device` is already initialized and able to transmit/receive.
* Future improvements will add statistics, error tracking, and potentially async/await support where applicable.

