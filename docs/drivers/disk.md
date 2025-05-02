---
title: Disk
nav_order: 1
parent: Drivers
---

# Disk

Author: Hunter Ross [@hunteross](https://github.com/hunteross/)

---

## SD Card Driver

### Source: [/crates/kernel/src/device/sdcard.rs](https://github.com/pincerOS/kernel/blob/main/crates/kernel/src/device/sdcard.rs)

This driver was initially developed with hardware in mind and thus was ported from another [bare metal Raspberry Pi project](https://github.com/rsta2/circle/blob/master/addon/SDCard/emmc.cpp)

The SD Card initializes correctly on hardware w/out issue and only minor changes are needed to initialize on QEMU. For R/W on QEMU some other minor changes are needed. These can be seen on the branch [`ext2-sd-test`](https://github.com/pincerOS/kernel/compare/main...ext2-sd-test).

---

## API

### read(&mut self, buf: &mut [u8]) -> Result<u32, SdCardError>

Reads bytes from its current offset into `buf`. Mostly used interally and `read_sector` is preferred. Returns bytes read or `SdCardError`.

### write(&mut self, buf: &[u8]) -> Result<u32, SdCardError>

Writes bytes from `buf` to its current offset. Mostly used internally and `write_sector` is preferred. Returns bytes written or `SdCardError`.

### seek(&mut self, offset: u64) -> u64

Changes the current block offset. Returns the offset value.

### get_capacity(&self) -> u64

Returns the capacity of the SD Card.

### get_block_size(&self) -> u32

Returns the block size of the SD Card. *Should be 512?*

### get_id(&self) -> &[u32; 4]

Returns the id of the card.

## The following functions are used to implement the `BlockDevice` trait for the SD Card Driver and are preferred over `read()` and `write()`.

### read_sector(&mut self, index: u64, buffer: &mut [u8; filesystem::SECTOR_SIZE],) -> Result<(), filesystem::BlockDeviceError>

Reads a sector's worth of bytes from offset: `index` into `buf`.

### write_sector(&mut self, index: u64, buffer: &[u8; filesystem::SECTOR_SIZE],) -> Result<(), filesystem::BlockDeviceError>

Writes a sector's worth of bytes from `buf` to offset: `index`.

### read_sectors(&mut self, index: u64, buffer: &mut [u8],) -> Result<(), filesystem::BlockDeviceError>

Reads multiple sectors worth of bytes from offset: `index` into `buf`.

*Note: `buf` must be a multiple of the block size*

### write_sectors(&mut self, index: u64, buffer: &[u8],) -> Result<(), filesystem::BlockDeviceError>

Writes multiple sectors worth of bytes from `buf` to offset: `index`.

*Note: `buf` must be a multiple of the block size*

---

## Filesystem Integration

SD Card integration with the filesystem is not yet complete. Currently the disk is able to read from directories and create files within them. Some of this work can be seen on the branch `ext2-sd-test`

To help complete this the `FileDescriptor` trait needs further implementation for `Ext2File`