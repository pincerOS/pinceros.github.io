---
title: Home
layout: home
nav_order: 1
permalink: /
---
# About The Project 🦀

PincerOS is a bare-metal microkernel-based multi-core operating system written from the ground up in Rust targeting the Raspberry Pi 4b. The project aims to be a distributed, scalable, and secure operating system for general-purpose use. We aim to support a wide range of applications such as networked video games, distributed computing, and more.

## Targeted Features ✨

- Microkernel Architecture
- Multi-core Support
- Memory Management
- Process Scheduling
- File System with Journaling Support
- Inter-process Communication (IPC)
- Device Drivers
- Networking
- Security

## Architecture 📐
PincerOS follows a microkernel architecture with the following key components:

- Kernel Core: Handles basic system operations, syscalls, scheduling, and IPC
- Memory Management: Implements virtual memory and memory protection
- Device Drivers: Manages hardware interfaces
- Network Stack: Provides networking capabilities
- Security Module: Handles access control and system security

# Installation 📦
Currently, the project can be tested on QEMU version 9.0 or higher. If your package manager doesn't have it, you will have to build QEMU from source.

## Dependencies
- Rust toolchain (https://www.rust-lang.org/tools/install)
- QEMU >= 9.0 (https://www.qemu.org/download/)
- Just (https://github.com/casey/just?tab=readme-ov-file#packages)

## Setup
<!-- 1. Install Rust target:
```rustup target add aarch64-unknown-none-softfloat``` -->

1. Clone the repository:
```git clone https://github.com/pincerOS/kernel.git```

2. Build the kernel and run:
```cd crates/kernel```
```just build-and-run``` to build and run the `main` example.

We also provide scripts for debugging and running with ui:
```bash
just --list
Available recipes:
    build example=example profile=profile target=target # Build the kernel
    build-and-run example=example profile=profile # Build and run in one command
    build-and-run-debug example=example           # Build and run with debug profile
    build-debug example=example target=target     # Build with debug profile
    debug                                         # Run with debug mode (wait for debugger)
    debug-ui                                      # Run with debug mode and UI display
    default                                       # Default: build and run the kernel
    run qemu_target=qemu_target qemu_debug=qemu_debug qemu_display=qemu_display debug_args=debug_args # Run the kernel in QEMU
    run-rpi3b                                     # Run on Raspberry Pi 3B
    run-ui                                        # Run with UI display
```

# Development Status 🚧

- [x] Basic kernel functionality
- [x] Multi-core support
- [ ] Network stack
- [ ] Application support
- [ ] File system
- [ ] Device drivers
- [ ] Security module
- [ ] Distributed computing support


# Credits 🎓
This project is a collaboration between students at the University of Texas at Austin. 🤘

- Aaron Lo (@22aronl)
- Alex Meyer (@ameyer1024)
- Anthony Wang (@honyant)
- Caleb Eden (@calebeden)
- Hunter Ross (@hunteross)
- @InsightGit
- Joyce Lai (@hexatedjuice)
- Neil Allavarpu (@NeilAllavarpu)
- @Razboy20
- Sasha Huang (@umbresp)
- Slava Andrianov (@Slava-A1)


This project also incorporates code from [CSUD](https://github.com/Chadderz121/csud/tree/master), made by Alex Chadwick and licensed under the MIT License (See CSUD_LICENSE for more details). CSUD was used as a base for the USB implementation and additional support of Interrupt & Bulk Endpoints were added on.
# License 📝

This project is licensed under the MIT License.

---

> _"Rust never sleeps." -Neil Young_
