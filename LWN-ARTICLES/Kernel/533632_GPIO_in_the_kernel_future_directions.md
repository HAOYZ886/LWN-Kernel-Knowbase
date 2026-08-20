---
title: "GPIO in the kernel: future directions"
url: https://lwn.net/Articles/533632/
date: "January 23, 2013"
category: "GPIO; General-purpose IO"
author: "By Jonathan Corbet January 23, 2013"
---

By **Jonathan Corbet**  
January 23, 2013

[Last week's article](<https://lwn.net/Articles/532714/>) covered the kernel's current internal API for general-purpose I/O (GPIO) lines. The GPIO API has seen relatively little change in recent years, but that situation may be about to change as the result of a couple of significant patch sets that seek to rework how the GPIO API works in the interest of greater robustness and better performance. 

#### No more numbers

The current GPIO API relies on simple integers to identify specific GPIO lines. It works, but there are some shortcomings to this approach. Kernel code is rarely interested in "GPIO #37"; instead, it wants "the GPIO connected to the monitor's DDC line" or something to that effect. For well-defined systems where the use of GPIO lines never changes, preprocessor definitions can be used to identify lines, but that approach falls apart when the same GPIO can be put to different uses in different systems. As hardware gets more dynamic, with GPIOs possibly showing up at any time, there is no easy way to know which GPIO goes where. It can be easy to get the wrong one by mistake. 

As a result, platform and driver developers have come up with various ways to locate GPIOs of interest. Even your editor once submitted [a patch adding a `gpio_lookup()` function](<https://lkml.org/lkml/2009/10/10/162>) to the GPIO API, but that patch didn't pass muster and was eventually dropped in favor of a driver-specific solution. So the number-based API has remained — until now. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Alexandre Courbot's [descriptor-based GPIO interface](<https://lwn.net/Articles/531848/>) seeks to change the situation by introducing a new `struct gpio_desc *` pointer type. GPIO lines would be represented by one of these pointers; what lives behind the pointer would be hidden from GPIO users, though. Internally, gpiolib (the implementation of the GPIO API used by most architectures) is refactored to use descriptors rather than numbers, and a new set of functions is presented to users. These functions will look familiar to users of the current GPIO API: 

```
#include <linux/gpio/consumer.h>
    
        int gpiod_direction_input(struct gpio_desc *desc);
        int gpiod_direction_output(struct gpio_desc *desc, int value);
        int gpiod_get_value(struct gpio_desc *desc);
        void gpiod_set_value(struct gpio_desc *desc, int value);
        int gpiod_to_irq(struct gpio_desc *desc);
        int gpiod_export(struct gpio_desc *desc, bool direction_may_change);
        int gpiod_export_link(struct device *dev, const char *name,
    			  struct gpio_desc *desc);
        void gpiod_unexport(struct gpio_desc *desc);
```

In short: the `gpio_` prefix on the existing GPIO functions has been changed to `gpiod_` and the integer GPIO number argument is now a `struct gpio_desc *`. There is also a new include file for the new functions; otherwise the interfaces are identical. The existing, integer-based API still exists, but it has been reimplemented as a layer on top of the descriptor-based API shown here. 

What is missing from the above list, though, is any way of obtaining a descriptor for a GPIO line in the first place. One way to do that is to get the descriptor from the traditional GPIO number: 

```
struct gpio_desc *gpio_to_desc(unsigned gpio);
```

There is also a `desc_to_gpio()` for going in the opposite direction. Using this function makes it easy to transition existing code over to the new API. Obtaining a descriptor in this manner will ensure that no code accesses a GPIO without having first properly obtained a descriptor for it, but it would be better to do away with the numbers altogether in favor of a more robust way of looking up GPIOs. The patch set adds this functionality in this form: 

```
struct gpio_desc *gpiod_get(struct device *dev, const char *name);
```

Here, `dev` should be the device providing the GPIO line, and "name" describes that line. The `dev` pointer is needed to disambiguate the name, and because code accessing a GPIO line should know which device it is working through in any case. So, for example, a video acquisition bridge device may need access to GPIO lines with names like "sensor-power", "sensor-reset", "sensor-i2c-clock" and "sensor-i2c-data". The driver could then request those lines by name with `gpiod_get()` without ever having to be concerned with numbers. 

Needless to say, there is a `gpiod_put()` for releasing access to a GPIO line. 

The actual association of names with GPIO lines can be done by the driver that implements those lines, if the names are static and known. In many cases, though, the routing of GPIO lines will have been done by whoever designed a specific system-on-chip or board; there is no way for the driver author to know ahead of time how a specific system may be wired. In this case, the names of the GPIO lines will most likely be specified in the device tree, or, if all else fails, in a platform data structure. 

The response to this interface is generally positive; it seems almost certain that it will be merged in the near future. The biggest remaining concern, perhaps, is that the descriptor interface is implemented entirely within the gpiolib layer. Most architectures use gpiolib to implement the GPIO interface, but it is not mandatory; in some cases, the `gpio_*` functions are implemented as macros that access the device registers directly. Such an implementation is probably more efficient, but GPIO is not usually a performance-critical part of the system. So there may be pressure for all architectures to move to gpiolib; that, in turn, would facilitate the eventual removal of the number-based API entirely. 

#### Block GPIO

The GPIO interface as described so far is focused on the management of individual GPIO lines. But GPIOs are often used together as a group. As a simple example, consider a pair of GPIOs used as an I2C bus; one line handles data, the other the clock. A bit-banging driver can manage those two lines together to communicate with connected I2C devices; the kernel contains a driver in `drivers/i2c/busses/i2-gpio.c` for just this purpose. 

Most of the time, managing GPIOs individually, even when they are used as a group, works fine. Computers are quite fast relative to the timing requirements of most of the serial communications protocols that are subject to implementation with GPIO. But there are exceptions, especially when the hardware implementing the GPIO lines themselves is slow; that can make it hard to change multiple lines in a simultaneous manner. But, sometimes, the hardware _can_ change lines simultaneously if properly asked; often the lines are represented by bits in the same device register and can all be changed together with a single I/O memory write operation. 

Roland Stigge's [block GPIO patch set](<https://lwn.net/Articles/533557/>) is an attempt to make that functionality available in the kernel. Code that needs to manipulate multiple GPIOs as a group would start by associating them in a single block with: 

```
struct gpio_block *gpio_block_create(unsigned int *gpios, size_t size,
    				     	 const char *name);
```

`gpios` points to an array of `size` GPIO numbers which are to be grouped into a block; the given `name` can be used to work with the block from user space. The GPIOs should have already been requested with `gpio_request()`; they also need to have their direction set individually. It's worth noting that the GPIOs need not be located on the same hardware; if they are spread out, or if the underlying driver does not implement the internal block API, the block GPIO interface will just access those lines individually as is done now. 

Manipulation of GPIO blocks is done with: 

```
unsigned long gpio_block_get(struct gpio_block *block, unsigned long mask);
        void gpio_block_set(struct gpio_block *block, unsigned long mask,
    		    	unsigned long values);
```

For both functions, `block` is a GPIO block created as described above, and `mask` is a bitmask specifying which GPIOs in the block are to be acted upon; each bit in `mask` enables the corresponding GPIO in the array passed to `gpio_block_create()`. This API implies that the number of bits in a `long` forces an upper bound on number of lines grouped into a GPIO block; that seems unlikely to be a problem in real-world use. `gpio_block_get()` will read the specified lines, simultaneously if possible, and return a bitmask with the result. The lines in a GPIO block can be set as a unit with `gpio_block_set()`. 

A GPIO block is released with: 

```
void gpio_block_free(struct gpio_block *block);
```

There is also a pair of registration functions: 

```
int gpio_block_register(struct gpio_block *block);
        void gpio_block_unregister(struct gpio_block *block);
```

Registering a GPIO block makes it available to user space. There is a sysfs interface that can be used to query and set the GPIOs in a block. Interestingly, registration also creates a device node (using the name provided to `gpio_block_create()`); reading from that device returns the current state of the GPIOs in the block, while writing it will set the GPIOs accordingly. There is an `ioctl()` operation (which, strangely, uses zero as the command number) to set the mask to be used with read and write operations. 

This patch set has not generated as much discussion as the descriptor-based API patches (it is also obviously not yet integrated with the descriptor API). Most likely, relatively few developers have felt the need for a block-based API. That said, there are cases when it is likely to be useful, and there appears to be no opposition, so this API can eventually be expected to be merged as well.
