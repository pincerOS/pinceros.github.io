---
title: Packet Representations
nav_order: 1
parent: Networking
---

# Packet Representations

## To Build a Network Stack

In order to use and test a network stack, there has to be some sort of network
driver and interface in place in order to connect to. Early on in development,
this wasn't yet available. The solution to this was to perform early development
on Linux before porting back to PincerOS and connecting with the chosen
interface. For a more in depth talk about development, check out the 
[Development]({{ site.baseurl }}/docs/networking/development) page.

But what's in a network stack? From the perspective of the user, we want to be
able to make simple, Unix-style calls to send and receive packets from our
network interface. While there are many approaches to implementing this, I took
a modual approach. First we start off with packet representation.

## Representations

Many systems opt to go for a "builder-style" packet system. Since the user would
not be directly manipulating packets, I took a coarser approach and just kept
all the packets as simple structs with `new` methods and no setters nor getters.

The main goal was to create `serialize` and `deserialize` methods for each type of packet that could hierarchically process incoming or outgoing packets. This meant that each method had to account for dynamic structure, checksums, or malformations. I personally found this to more intuitive than the sequential processing style that seeks to check packets in-line while processing protocol logic as well.

---

## General Structure

Packets are typically defined as `struct`s, where each field corresponds to a region in the binary layout. Here's a typical Ethernet frame structure:

| Field              | Size (Bytes) | Description                        |
|--------------------|--------------|------------------------------------|
| **Destination MAC**| 6            | MAC address of the destination     |
| **Source MAC**     | 6            | MAC address of the sender          |
| **EtherType**      | 2            | Type of payload (e.g., IPv4 = 0x0800) |
| **Payload**        | 46–1500      | Encapsulated data (e.g., IP packet) |
| **FCS** (Frame Check Sequence) | 4 | CRC for error checking             |
