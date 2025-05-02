---
title: GPIO
nav_order: 2
parent: Drivers
---

# GPIO Driver

Author: Hunter Ross [@hunteross](https://github.com/hunteross/)

---

### Source: /crates/kernel/src/device/gpio.rs

This driver is used to configure the GPIO pins on the Raspberry Pi 4. The Raspberry Pi 4 has a total of 58 GPIO pins. GPIO configuration could be necessary in order to have other subsystems function correctly.

---

## API

### set_function(&mut self, pin: u8, function: GpioFunction) ### 

Sets the function for a specfic GPIO pin

*For more information on different GPIO functions refer to section 5.3 of the bcm2711-peripherals datasheet*

### set_pull(&mut self, pin: u8, pull: GpioPull) ### 

Configures the pull up / pull down resistor for a specific GPIO pin

### set_pull_mask(&mut self, mask: u16, index: usize, pull: GpioPull) ### 

Configures the pull up / pull down resistors for multiple pins.
Pins are designated by providing an `index` referring to which set of pins we want configured (ie. 0-15, 15-31, etc.) and a bitmask: `mask` that determines which pins in that set we want to set the pull to.

### add_event_detect(&mut self, pin: u8, event: GpioEvent) ### 

Adds an event detect for a specific GPIO pin. When this event occurs an interrupt should be triggered.

### check_event(&self, pin: u8) -> bool ### 

Checks if an event was triggered for a specific GPIO pin

### clear_event(&mut self, pin: u8) ### 

Clears the event detection for a specific GPIO pin

### check_any_event(&self) -> bool ### 

Check if any event was triggered

### clear_all_events(&mut self) ### 

Clear event detection for all GPIO pins

### set_high(&mut self, pin: u8) ### 

Sets a GPIO pin high

### set_low(&mut self, pin: u8) ### 

Sets a GPIO pin low

### toggle(&mut self, pin: u8) ### 

Toggles a GPIO pin high -> low or low -> high

### read(&self, pin: u8) -> bool ### 

Read a GPIO pin value (1 or 0), returns a bool

### read_mask(&self, index: usize) -> u32 ### 

Reads a mask of 32 pins, designate which set of pins you want with `index`

---

### For more information please refer to the bcm2711-peripherals datasheet

