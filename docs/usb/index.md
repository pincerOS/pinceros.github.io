---
title: USB
nav_order: 3
---

# USB
{: .no_toc }

---

Author: Aaron Lo [@22aronl](https://github.com/22aronl/)

A USB2.0 Complient USB Stack that supports Control, Interrupt, and Bulk endpoints. The stack was designed using [CSUD](https://github.com/Chadderz121/csud/) as its base and then extended to supporting more features as well as working on hardware. [FreeBSD](https://www.freebsd.org/) was used for reference on some initalization code for the driver.

In QEMU, it supports basic HID Mouse, HID Keyboard, USB-Hub, and a RNDIS Usb-Net.
On Hardware, it supports HID Mouse, HID Keyboard, and an under-ideal-circumstances AX88179 USB-Net, all interfaceable through USB2.0 Hubs. However, currently, a hub can only support a single device at a time.

For more information, take a look at the subpages

{: .fs-6 .fw-300 }
