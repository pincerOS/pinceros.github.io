---
title: Usb Enumeration
nav_order: 2
---

# Usb Enumeration
{: .no_toc }

---

Author: Aaron Lo [@22aronl](https://github.com/22aronl/)

# USB Enumeration Process

The USB enumeration process identifies and configures a newly connected USB device so that the host can communicate with it. When a device is plugged in, the host controller and USB stack perform a series of control transfers to determine the device’s capabilities and assign it an address.

---

## 1. Device Connection

When a device is physically attached, the host’s port logic detects the connection (via a rise in D+ or D– line voltage) and signals the USB stack to begin enumeration.

---

## 2. Bus Reset

The host issues a **Reset** on the bus (holding both D+ and D– low for 10 ms) to force the device into a known state. This ensures:
- The device’s internal state machines are reset.
- The device responds at default address 0.

---

## 3. Get Device Descriptor (Partial)

1. **Setup Packet**  
   - **bmRequestType**: `0x80` (Device‑to‑Host, Standard, Device)  
   - **bRequest**: `0x06` (GET_DESCRIPTOR)  
   - **wValue**: `0x0100` (Descriptor Type: Device, Index 0)  
   - **wIndex**: `0x0000`  
   - **wLength**: typically 8 bytes (to read the first part)
2. **Data Stage**: Device returns the first 8 bytes of its Device Descriptor, which include:
   - USB version
   - Maximum packet size for endpoint 0
3. **Status Stage**: Host sends an ACK.

---

## 4. Set Address

1. **Setup Packet**  
   - **bmRequestType**: `0x00` (Host‑to‑Device, Standard, Device)  
   - **bRequest**: `0x05` (SET_ADDRESS)  
   - **wValue**: new device address (1–127)  
   - **wIndex**, **wLength**: `0x0000`
2. **Status Stage**: Host acknowledges; device switches from address 0 to the assigned address on the next **IN**.

---

## 5. Get Full Device Descriptor

With the new address in effect, the host re-issues a GET_DESCRIPTOR (as in step 3), this time requesting the full length (typically 18 bytes) so it can read:
- Vendor ID (VID) and Product ID (PID)
- Device class, subclass, protocol
- Number of possible configurations

---

## 6. Get Configuration Descriptor (Header)

1. **Setup Packet**  
   - **bmRequestType**: `0x80`  
   - **bRequest**: `0x06`  
   - **wValue**: `0x0200` (Descriptor Type: Configuration, Index 0)  
   - **wIndex**, **wLength**: typically 9 bytes
2. **Data Stage**: Host reads the 9-byte configuration header:
   - Total length of all descriptors
   - Number of interfaces
   - Maximum power consumption

---

## 7. Get Full Configuration Descriptor

Knowing the total length, the host requests the full configuration block (including all interface and endpoint descriptors) so it can parse each interface.

---

## 8. Set Configuration

1. **Setup Packet**  
   - **bmRequestType**: `0x00`  
   - **bRequest**: `0x09` (SET_CONFIGURATION)  
   - **wValue**: configuration value (from the header)  
   - **wIndex**, **wLength**: `0x0000`
2. **Status Stage**: Host and device acknowledge; the device moves from the **Default** to the **Configured** state, and its endpoints (other than EP0) become active.

---

## 9. Interface and Alternate Setting (if needed)

Some devices support alternate interface settings for different bandwidth/power modes.

1. **Get Interface Descriptor**  
   - Similar GET_DESCRIPTOR with **wValue** `0x0400` (Interface, index)  
2. **Set Interface**  
   - **bRequest**: `0x0B` (SET_INTERFACE)  
   - **wValue**: alternate setting number  
   - **wIndex**: interface number  

---

## 10. Endpoint Descriptors

From the full configuration descriptor (step 7), the host already knows each endpoint’s:
- Address (direction, endpoint number)
- Transfer type (control, interrupt, bulk, isochronous)
- Maximum packet size
- Polling interval (for interrupt/isochronous)

No further GET_DESCRIPTOR is required unless the host needs to re-read.

---

## 11. Driver Binding

Based on the class, subclass, protocol, VID/PID and interface descriptors:
- The OS loads or binds the correct driver (e.g., USB mass storage, CDC-NCM, HID, etc.)
- Any class-specific initialization is performed

---

## 12. Device Ready

With the configuration set and the driver loaded, the device is in the **Configured** state. The host can now issue transfers on the data endpoints to send and receive application data.


{: .fs-6 .fw-300 }
