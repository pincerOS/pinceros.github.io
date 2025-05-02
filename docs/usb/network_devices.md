---
title: Networking Devices
nav_order: 4
parent: USB
---

# Networking Devices
{: .no_toc }

---

Author: Aaron Lo [@22aronl](https://github.com/22aronl/)

# USB CDC Networking Device (CDC-ECM / CDC-NCM)

USB networking devices normally implement the **Communication Device Class (CDC)** with either the **Ethernet Control Model (ECM)** or **Network Control Model (NCM)** subclass.  
- **bDeviceClass**: `0x02` (CDC)  
- **bDeviceSubClass**: `0x06` (Ethernet Control Model) or `0x0D` (NCM)  
- **bDeviceProtocol**: `0x00`  

## Standard Endpoints

A CDC‑based network adapter usually exposes at least three endpoints:

1. **Interrupt IN**  
   - **Purpose**: Notify the host of asynchronous events (link up/down, speed change, etc.)  
   - **MaxPacketSize**: 8–16 bytes  
   - **Polling Interval**: 10–255 ms  

2. **Bulk IN**  
   - **Purpose**: Send Ethernet frames (network packets) from device → host  
   - **MaxPacketSize**: Typically 64 bytes (USB 2.0 full‑speed) or 512 bytes (high‑speed)  

3. **Bulk OUT**  
   - **Purpose**: Receive Ethernet frames from host → device  
   - **MaxPacketSize**: Same as Bulk IN  

## RNDIS Protocol (Microsoft)

RNDIS is a Microsoft extension over CDC, still supported by QEMU’s `usb-net` device.

1. **Initialization Sequence**  
   - Host sends **RNDIS_INITIALIZE_MSG** (OID requests list, medium type, version).  
   - Device responds with **RNDIS_INITIALIZE_CMPLT** (status, max packet transfer size).  

2. **Control Requests**  
   - **SET_MSG**: Host → Device (e.g. OID_GEN_CURRENT_PACKET_FILTER)  
   - **GET_MSG**: Host → Device (query OIDs like OID_GEN_MAC_ADDRESS)  
   - **RESET_MSG**: Host → Device (reset statistics, filters)  

3. **Data Messages**  
   - **RNDIS_PACKET_MSG** header (44 bytes):  
     ```c
     struct rndis_packet_msg {
         uint32_t MessageType;      // RNDIS_PACKET_MSG (0x00000001)
         uint32_t MessageLength;    // header + data length + padding
         uint32_t DataOffset;       // offset from start of this struct
         uint32_t DataLength;       // length of Ethernet frame
         // optional PerPacketInfoOffset / Length
     };
     ```
   - Payload follows immediately; padded to 4 bytes.

## AX88179 USB → Ethernet Controller

The **ASIX AX88179** is a popular USB 3.0 → Gigabit Ethernet chip.  
- Uses a proprietary bulk‑transfer protocol:  
  - **8 byte header** before each packet:  
    - 4 bytes: packet length (little‑endian)  
    - 4 bytes: padding or flags (usually zero)  
- **Interrupt IN** endpoint delivers link‑status changes and PHY information
- **Bulk IN** endpoint for receiving packets
- **Bulk OUT** endpoint for sending packets

Since its protocol is proprietary, and PincerOS does not have the datasheet, the driver is not fully implemented. We run into an issue of after receiving for a while, the chip starts reporting gibberish data or stops receiving altogether. This may be an issue in how the chip is configured but without the datasheet, it is hard to really know. 

---

{: .fs-6 .fw-300 }
