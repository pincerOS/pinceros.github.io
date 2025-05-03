---
title: Design Considerations
nav_order: 3
parent: Virtual Memory
---

## Overview

Virtual memory is a very powerful tool when building operating systems, so it is important to consider how you will design your virtual memory structure and expose it to the rest of the operating system. Below are some general considerations to make when designing a virtual memory system

### Acessing Page Tables

Translation descriptors which compose the page tables store the physical addresses of their next pages, which can raise some challenges if your kernel is running with virtual memory enabled, as it is not guaranteed that the physical address and the virtual address of the page in the translation descriptor are the same.

- One solution to this problem is identity mapping the physical memory of your system. This means that somewhere before virtual memory is enabled, you will want to set up the kernel page table such that its address in the virtual address space is itself.
- Another solution which is somewhat more complicated is setting up recursive page tables. This essentially means that there will be some entry in your page table that will allow it to access itself. To give a more concrete example, in a system with a page size of 4KB and a 48 bit address space, the top level page table could be set up to be stored at address 0xFFFFFFFFF000. When this address is accessed every stage of the translation going to the very last entry in each of the page table, with the level 3 page table storing the address of the top level page table, thereby allowing you to access and modify your page table.
- A third solution is to have some sort of functionality to be able to convert back and forth between physical and virtual addresses. Something like this would likely be tied to the system or allocator which hands out pages for use in the page tables. For example, if the physical and virtual addresses of that page allocator are known, then they could be used to write functions which can translate between physical and virtual addresses.

### Exposing Virtual Memory

Virtual memory is very closely tied to user processes and managing the ranges that they are allowed to access. One of the most common ways that user processes interact with virtual memory is through some sort of system similar to Linux's `mmap` system call, and the kernel will need to have internal mechanisims for manipulating virtual memory to support all of mmap's functionality. However, as seen in the previous section, there are quite a few flags to set and details to consider when manipulating the translation descriptors inside of the page tables. As such, it is reccomended to have a a clean interface for accessing and modifying those descriptors, translating between physical and virtual addresses, and allocating and deallocating pages. Internally, PincerOS exposes functions which allow for the setting of translation descriptors to specific levels in the page table structure, which are then used to power the in-kernel mmap and munmap.

### Efficiently Managing Allocated Ranges

If user processes are able to request ranges of memory for their own use through a `mmap` like interface, the need will arise for a efficient way to query and insert into the set of reserved mappings for the process. The traditional approach to solving this problem is to use some sort of self balancing tree, with the Linux kernel using a red-black tree and the PincerOS kernel using the Rust's BTreeMap.

