---
title: AArch64 Virtual Memory Structure
nav_order: 1
parent: Virtual Memory
---

## Overview

Virtual memory functions through traversing levels of page tables, which are essentially arrays of translation descriptors. This section will discuss the following

- Navigating page/translation tables using a virtual address
- Variable width virtual address spaces
- Managing the kernel and user translation tables


### Virtual Address Structure And Translation Control Registers

To begin, lets discuss how a 48 bit virtual address for a 4KB page is used to access the relevant data as visualized in the [ARM documentation](https://developer.arm.com/documentation/101811/0104/Translation-granule/The-starting-level-of-address-translation)

1. The 9 bits in the range [47:39] (inclusive) are used to index into the top level page table, which in this case is the level 0 table. This page table is stored in either TTBR0_EL1 or TTBR1_EL1, with the distinction being discussed later on this page. Indexing into the page table with these bits will lead to a translation descriptor, which stores the address of the corresponding level 1 table.
2. The 9 bits in the range [38:30] (inclusive) are used to index into the level 1 table. Once again, the translation descriptor has an address, however this address can point to either a level 2 table or to a 1 GB block/page. The method for differentiating the two is discussed on the translation descriptor page.
3. The 9 bits in the range [29:21] (inclusive) are used to index into the level 2 table. Here, the translation descriptor will have either the address of a 2MB block, or the address of a level 3 table.
4. The 9 bits in the range [20:12] (inclusive) are used to index into the level 3 table. The translation descriptor will then have the address of the page with the data that you are trying to access.
5. The last 12 bits (bits [11:0] inclusive) are used to find the requested byte address in the page, giving you your data.

This walkthrough is for a 4KB page, but the general principle applies to all page sizes. For example, if you have a 2MB block/page in the same 48 bit virtual address space, bits [47:21] will still be used to navigate through the page tables at levels 0, 1, and 2, and then that will leave the last 21 bits to be used to find the offset of the byte that you are looking for in the block. The reason that this works is because 2^21 is equal to the number of bytes in a 2MB page the same way that 2^12 is equal to 4096 bytes, which is the number of bytes in a 4KB page.

### Variable Width Virtual Address Spaces

AArch64 allows you to configure the size of the virtual address space. 48 bit virtual address spaces are very common and is what PincerOS uses in most of its execution, however the PincerOS kernel actually initializes in a 25 bit virtual address space and then switches into a 48 bit virtual address space.

This is accomplished through setting specific bits in the Translation Control Register, which for EL1 (which is traditionally used when running in the kernel) and EL0 (traditionally used when running in userspace) is [TCR_EL1](https://developer.arm.com/documentation/ddi0601/2025-03/AArch64-Registers/TCR-EL1--Translation-Control-Register--EL1-). The number of bits in the EL1 virtual address space is regulated by bits [5:0] and the number of bits in the EL0 virtual address space is regulated by bits [21:16]. The value stored in these spots in TCR_EL1 is 64 - DESIRED_VIRTUAL_ADDRESS_SPACE_SIZE. For example, for a 25 bit virtual address space the value in the field would be set to 39 because 64 - 39 = 25. It is quite important to correctly configure the Translation Control register, so it is suggested that you review the ARM documentation and reference manual.

It shoud be observed that for a 25 bit virtual address space, the full 9 bits for the level 2 table index are not available, however this does not significantly alter the translation process. Below is an example of the translation process for a 25 bit virtual address space

1. The 4 bits in the range [24:21] are used to index into the top level page table, which in this case would be the level 2 table. Once again, the translation descriptor has an address which can point to either a 2MB block or the address of a level 3 page table.
4. The 9 bits in the range [20:12] (inclusive) are used to index into the level 3 table. The translation descriptor will then have the address of the page with the data that you are trying to access.
5. The last 12 bits (bits [11:0] inclusive) are used to find the requested byte address in the page, giving you your data.

Setting a virtual address width is a tradeoff between speed and size. A virtual address space with more bits can cover a larger range of memory addresses, however each TLB miss requires traversing more levels of the page table tree. A virtual address space with less bits is the opposite, with it being faster to traverse the page table tree due to translation potentially starting at a lower level of the tree, at the cost of being able to cover a smaller range of memory addresses.

### Kernel And User Translation Tables

The earlier sections mention top level page tables, which are the page tables at which the translation process starts. This value is stored in the Translation Table Base Register (TTBR) for your exception level, with the kernel top level page table usually being stored in TTBR1 and the user top level page table usually being stored in TTBR0. This leads to the kernel virtual address space being in the "higher half" of the virtual address space and the user virtual address space being in the "lower half." An important distinction between the two is that addresses corresponding with the higher half set all bits not used for virtual address translation set to 1, while for the lower half they are all set to zero. For an example of this in action, please look into the PincerOS example file at `kernel/crates/kernel//examples/vaToPaKernel.rs` and observe how the requested virtual address is modified in order to correctly acess an address in the higher half of the virtual address space. 

