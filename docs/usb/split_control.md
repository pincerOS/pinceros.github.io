---
title: Split Control
nav_order: 4
---

# Split Control Transfers 2.0
{: .no_toc }

---

Author: Aaron Lo [@22aronl](https://github.com/22aronl/)

# USB Split Control Transfers

When a host and device share the same USB speed (Full‑Speed at 12 Mbps or High‑Speed at 480 Mbps), transactions proceed directly: the host sends data packets, the device responds, and the protocol remains uniform. However, when a High‑Speed host needs to communicate with a Full‑Speed device, the disparity in signaling and timing requires a special mechanism. Without it, the High‑Speed host’s 480 Mbps signaling would overwhelm a Full‑Speed device limited to 12 Mbps.

This situation commonly arises when a host must support both:

- A **USB‑Net** gadget operating at High‑Speed  
- A **USB‑Mouse** operating at Full‑Speed  

Direct transfers to the Full‑Speed device will fail. The USB specification solves this via **Split Control Transfers**, which break a single control transfer into several phases, allowing the downstream Full‑Speed device to operate at its native speed while isolating it from the High‑Speed link timing.

---

## 1. Overview of Split Transactions

A split transaction consists of three phases:

1. **Start-Split (SSPLIT)**  
   - Initiated by the host controller on the High‑Speed bus  
   - Contains the original SETUP packet (bmRequestType, bRequest, wValue, wIndex, wLength)  
   - Routed downstream toward the hub’s Transaction Translator (TT)

2. **Micro-Frame Delay**  
   - The hub buffers the SSPLIT request  
   - After one or more 125 µs micro‑frames, the hub issues an internal Full‑Speed SETUP to the attached device

3. **Complete-Split (CSPLIT)**  
   - Issued by the host to collect the response (DATA and STATUS stages)  
   - Commands the hub’s TT to deliver the Full‑Speed device’s reply back over the High‑Speed link  

---

## 2. Detailed Phases

### 2.1 SETUP Stage

- **Host → Hub (High‑Speed SSPLIT)**  
  - PID: SPLIT_START  
  - Payload: Standard SETUP packet fields  
- **Hub → Device (Full‑Speed SETUP)**  
  - After buffering, the hub’s TT generates a Full‑Speed SETUP with correct timing  

### 2.2 DATA Stage (if applicable)

- For control transfers with a DATA stage, the hub performs two split transfers:
  1. **SSPLIT (Host → Hub)**  
     - Indicates DATA–IN or DATA–OUT direction  
  2. **CSPLIT (Host → Hub)**  
     - Retrieves or sends data to the Full‑Speed device  

### 2.3 STATUS Stage

- Always a zero‑length packet  
- Host issues a final SSPLIT to signal the hub to send the Full‑Speed zero‑length status packet, followed by a CSPLIT to acknowledge completion  

---

## 3. Role of the Transaction Translator

- **Located inside USB hubs** capable of handling dual‑speed ports  
- Converts High‑Speed packets into Full‑Speed timing and vice versa  
- Buffers split transactions and ensures micro‑frame alignment  

---

## 4. When to Use Split Transfers

- Supporting mixed‑speed devices on the same root hub  
- Ensuring legacy Full‑Speed peripherals function on a High‑Speed bus  
- Required for class‑compliant devices like USB Networking (RNDIS, CDC‑ECM) coexisting with Full‑Speed HID  

---

## 5. Example Control Transfer Sequence

| Phase      | Host Packet      | Hub Action                 | Device Packet         |
|------------|------------------|-----------------------------|-----------------------|
| SETUP      | SSPLIT (SETUP)   | Buffer & translate          | Full‑Speed SETUP      |
| DATA‑IN    | SSPLIT (IN)      | Trigger Full‑Speed IN       | Full‑Speed DATA       |
| DATA‑IN    | CSPLIT (CSPLIT)  | Forward data to host        | —                     |
| STATUS     | SSPLIT (OUT ZLP) | Trigger Full‑Speed ZLP      | Full‑Speed STATUS     |
| STATUS     | CSPLIT           | Acknowledge completion      | —                     |

---

## 6. Key Benefits

- **Compatibility**: Allows High‑Speed hosts to support older Full‑Speed devices  
- **Efficiency**: Minimizes host intervention by batching downstream transactions  
- **Transparency**: USB class drivers remain unchanged—split handling is managed by the host controller and hub firmware  

---

By leveraging Split Control Transfers and the hub’s Transaction Translator, USB stacks can seamlessly communicate across speed boundaries, ensuring reliable operation of mixed‑speed USB topologies.


{: .fs-6 .fw-300 }
