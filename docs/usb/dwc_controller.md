---
title: DWC USB Controller  
nav_order: 1
parent: USB
---

# DWC‑OTG USB 2.0 Controller  
{: .no_toc }

**Author:** Aaron Lo [@22aronl](https://github.com/22aronl/)  

---

## Overview

The Raspberry Pi 3 B and 4 B integrate a DesignWare Cores (DWC‑OTG) USB 2.0 controller that can operate in both **host** and **device** modes. On the Pi 4 B, this controller is wired through the USB‑C power port; to enable USB‑C data functionality, power delivery must be managed via the GPIO header.

At a high level, the DWC controller exposes a set of memory‑mapped registers. The OS uses these registers to:

1. Drive the USB bus  
2. Discover and configure connected devices  
3. Exchange data through endpoints  

---

## Initialization Sequence

When PincerOS (or any OS) boots the DWC‑OTG controller in **host** mode, it must:

1. **Select Host Mode**  
2. **Choose Internal PHY**  
3. **Reset** the controller  
4. **(PincerOS‑specific)** Switch to non‑scatter/gather DMA  
5. **Enable interrupts** and install the IRQ handler  

Additionally, the OS must allocate DMA‑capable memory regions and turn on USB power via the GPU’s mailbox interface.

---

## Device Enumeration

Once initialized, the controller enumerates each port:

1. **Bus Reset**  
2. **Get Device Descriptor**  
3. **Assign Address**  
4. **Read Configuration Descriptor**  
5. **Load and bind** the appropriate driver  

To simplify driver binding, PincerOS presents a **fake root hub**. This virtual hub lets the OS use a single, uniform interface for all downstream devices, regardless of type.

---

## USB Endpoint Types

PincerOS supports three endpoint transfer types. Each endpoint appears as a “channel” on the DWC controller (except channel 0, which is reserved for control transfers).

### 1. Control Transfers  
- **Channel:** 0  
- **Usage:**  
  1. Send SETUP packet  
  2. Data stage (IN or OUT)  
  3. Status stage  
- **Purpose:** Configure devices and retrieve descriptors  

### 2. Interrupt Transfers  
- **Channels:** 1–N  
- **Usage:** Periodic, low‑latency transfers for HID devices (keyboards, mice)  
- **Characteristics:**  
  - Guaranteed polling interval  
  - Small maximum packet size  
- **Implementation:** PincerOS uses a timer to schedule each interrupt transaction.

### 3. Bulk Transfers  
- **Channels:** 1–N  
- **Usage:** Large, non‑time‑critical data payloads (e.g., mass storage, printers)  
- **Characteristics:**  
  - No guaranteed latency  
  - High throughput  


---

## DWC Register Map and Key Registers

The DWC‑OTG controller provides a set of memory‑mapped registers for configuration, status, and control. Key registers include:

- **Global Registers**  
  - `GOTGCTL` (0x000): OTG control and status  
  - `GUSBCFG` (0x010): USB configuration (PHY selection, turn‑around time)  
  - `GRSTCTL` (0x014): Software resets (core, host, device, AHB)  
  - `GINTSTS` (0x018): Raw interrupt status  
  - `GINTMSK` (0x01C): Interrupt mask  

- **Host Port and Channel Registers**  
  - `HPRT0` (0x440): Port status and control for port 0  
  - `HCCHARx` (0x500 + x×0x20): Host channel characteristics  
  - `HCTSIZx` (0x508 + x×0x20): Transfer size and packet count  
  - `HCINTx` (0x50C + x×0x20): Host channel interrupt  
  - `HCINTMSKx` (0x510 + x×0x20): Host channel interrupt mask  

Refer to the official DesignWare TRM for a complete map and detailed bit definitions.

---

## Non‑Polling, Interrupt‑Driven Operation

In CSUD, as well as many other simple USB stacks, the DWC controller is polled until the USB transfer is complete. However, this approach is inefficient and can lead to high CPU usage, especially in a multitasking environment. PincerOS adopts an interrupt‑driven model to improve efficiency and responsiveness.

1. **Enable Interrupts**  
   - Set bits in `GINTMSK` for events of interest (e.g., Channel interrupt, Port Changed).
   - Set bits in `HCINTMSK` for each host channel to enable interrupts for transfer completion, errors, and stalls.
2. **Wait**
    - The CPU can perform other tasks after sending a request, waiting for the USB device to respond.
2. **Interrupt Handler**  
   - Read `GINTSTS` to identify the source.  
   - **Host Channel (`HCHINT`)**:  
     - Check `HCINTx` for transfer-complete, stall, or error flags.  
     - Clear by writing to `HCINTx`.  
   - Handle the event (e.g., process data, notify the driver).
3. **Bottom‑Half Processing**  
   - Defer heavy work (descriptor parsing, packet processing) to a workqueue or tasklet.  

This model reduces CPU overhead and improves responsiveness by reacting only when hardware events occur.
