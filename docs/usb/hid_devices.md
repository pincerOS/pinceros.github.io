---
title: HID Keyboard and Mouse
nav_order: 3
---

# HID Keyboard and Mouse
{: .no_toc }

---

Author: Aaron Lo [@22aronl](https://github.com/22aronl/)

# Human Interface Device (HID) Interface

The USB Human Interface Device (HID) interface provides a standardized way for the host to communicate with input and output peripherals—keyboards, mice, joysticks, gamepads, and other user-interface devices. HID emphasizes simplicity and efficiency, abstracting device-specific details behind two primary data structures:

- **HID Descriptor**: Describes the device’s capabilities, data formats, report sizes, and logical ranges.
- **HID Report**: Conveys actual input (or output) data between host and device.

---

## HID Descriptor

The HID Descriptor is a binary blob that the device returns in response to a GET_DESCRIPTOR request (descriptor type 0x21). It consists of:

1. **Header**  
   - `bLength` (1 byte): Size of this descriptor  
   - `bDescriptorType` (1 byte): HID (0x21)  
   - `bcdHID` (2 bytes): HID Class Specification release number  
2. **Country Code** (`bCountryCode`, 1 byte)  
3. **Number of Report Descriptors** (`bNumDescriptors`, 1 byte)  
4. **Report Descriptor Entry** (5 bytes each)  
   - `bDescriptorType` (0x22)  
   - `wDescriptorLength` (size of the report descriptor in bytes)  

The **Report Descriptor** (type 0x22) follows, composed of a series of “Items” that specify Usage Pages, Usages, Report Sizes, Report Counts, Input/Output/Feature fields, logical and physical minimums/maximums, and collection hierarchies. This descriptor fully defines the format of every report the device can send or receive.

---

## HID Report

HID Reports are exchanged over Interrupt endpoints with the following characteristics:

- **IN Endpoint**: Device → Host (input data)  
- **OUT Endpoint**: Host → Device (output commands, LEDs, force feedback)  
- **Polling Interval**: Defined in the endpoint descriptor (`bInterval`), typically 1–16 ms for low/full-speed devices.

Each report consists of:

- **Report ID** (optional, 1 byte): If the descriptor defines multiple reports, the first byte distinguishes them.
- **Payload**: Fields as defined by the Report Descriptor—bitfields for buttons, axes, modifiers, and other controls.

---

### Keyboard Reports (8 Bytes)

A standard boot‐protocol keyboard uses an 8‐byte report:

| Byte | Meaning                                          |
|------|--------------------------------------------------|
| 0    | Modifier bitmap (bit 0 = Left Ctrl, …, bit 7 = Right GUI) |
| 1    | Reserved (0x00)                                  |
| 2–7 | Up to 6 concurrent keycodes (usage IDs)          |

- **Key rollover** is typically limited to 6 keys in boot protocol (N-key rollover requires a custom report descriptor).
- **Usage IDs** correspond to USB HID Usage Tables (e.g., 0x04 = “a” key, 0x05 = “b” key).

---

### Mouse Reports (Variable Length)

Mouse reports vary by descriptor but often follow this structure:

| Byte | Meaning                              |
|------|--------------------------------------|
| 0    | Button bitmap (bit 0 = Left, bit 1 = Right, bit 2 = Middle, etc.) |
| 1    | X-axis movement (signed integer)     |
| 2    | Y-axis movement (signed integer)     |
| 3    | Wheel movement (signed integer)      |
| 4…n | Additional axes or tilt/rudder, vendor-specific fields |

- **Movement values** are typically two’s‑complement 8‑bit or 16‑bit fields.
- **Buttons** beyond three may appear as additional bits or separate fields.

---

## Transport via Interrupt Endpoint

1. **Endpoint Descriptor**  
   - `bEndpointAddress` - IN or OUT, endpoint number
   - `bmAttributes` - 0x03 (Interrupt)
   - `wMaxPacketSize` - Maximum report size (bytes)
   - `bInterval` - Polling interval (ms for full/low speed)
2. **Host Polling**  
The host polls the IN endpoint at the specified interval; the device must queue a report or NAK otherwise.
3. **Data Toggle**  
Uses DATA0/DATA1 toggling for transfer integrity.
4. **Error Recovery**  
If a transfer stalls, the host issues a CLEAR_FEATURE request to reset the endpoint.

---

## Additional Details

- **Boot Protocol vs. Report Protocol**:  
- *Boot Protocol* uses a fixed, standardized report format (for BIOS/UEFI support).  
- *Report Protocol* allows the device’s custom Report Descriptor to define arbitrary report layouts.

- **Usage Pages & Usages**:  
The HID usage tables classify controls (e.g., Generic Desktop Controls, Simulation Controls). Each control has a Usage Page (e.g., 0x01 = Generic Desktop) and a Usage ID (e.g., 0x30 = X axis).

- **Feature Reports**:  
Optional, bi‑directional reports for configuration (e.g., backlight settings, vendor-specific commands).

- **Descriptor Retrieval Sequence**:  
1. GET_DESCRIPTOR(Device) → Device Descriptor  
2. GET_DESCRIPTOR(Configuration) → Configuration + Interface + HID + Endpoint Descriptors  
3. GET_DESCRIPTOR(HID Report) → Report Descriptor  

{: .fs-6 .fw-300 }
