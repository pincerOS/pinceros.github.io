---
title: RNG
nav_order: 3
parent: drivers
---

# RNG Driver

Author: Hunter Ross [@hunteross](https://github.com/hunteross/)

---

### Source: /crates/kernel/src/device/rng.rs

*Note: This driver is for the Raspberry Pi 3 (BCM2385) and will not work for the raspberry pi 4. This driver has also not been thoroughly tested*

---

## API

**rng_read(&mut self, buf: &mut [u32], wait: bool) -> usize**

The rng device will try to fill the provided buffer with as many random words as it can. 

If `wait == false` and no words are ready then it returns 0. If `wait == true`, the device will wait until a word is ready to be read and will return the number of words put into the buffer.

*Note: The device may not fill the entire length of the buffer*