---
title: AArch64 Translation Descriptors 
nav_order: 2
parent: Virtual Memory
---

## Overview

As mentioned in the previous page, page tables are composed of translation descriptors which are 8 byte entries composed of several fields. These are also called long descriptors. Below are some key fields to take note of:

- [0] - The valid bit, indicates if this entry contains meaninful data
- [1] - The descriptor bit. In a level 1 or 2 table, this bit being set indicates that the entry is a table descriptor. If it is not set, that indicates that the translation descriptor instead points to a block (1 GB or 2MB respectivley).
- [4:2] - These bits dictate the cache behavior of memory in this page, which is useful for devices. This is used for the Memory Attribute Indirection Register (MAIR)
- [6] - Unprivilleged access bit, used to mark a page as acessible in EL0. This should be set for pages that you want user processes to be able to access.
- [7] - Read only bit
- [10] - Access flag. This indicates that this page in memory is being accessed for the first time since this flag was previously set to 0. This should generally be set for pages that are newly placed into the page table.
- [39:12] - The address of the next translation table or the data page

As always, please refer to the ARM architecture reference manual for more information

