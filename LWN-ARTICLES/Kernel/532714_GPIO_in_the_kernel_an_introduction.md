---
title: "GPIO in the kernel: an introduction"
url: https://lwn.net/Articles/532714/
date: "January 16, 2013"
category: "GPIO; General-purpose IO"
author: "By Jonathan Corbet January 16, 2013"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
January 16, 2013

A GPIO (general-purpose I/O) device looks like the most boring sort of peripheral that a computer might offer. It is a single electrical signal that the CPU can either set to one of two values — zero or one, naturally — or read one of those values from (or both). Either way, a GPIO does not seem like a particularly expressive device. But, at their simplest, GPIOs can be used to control LEDs, reset lines, or pod-bay door locks. With additional "bit-banging" logic, GPIOs can be combined to implement higher-level protocols like [i2c](<https://en.wikipedia.org/wiki/I%C2%B2C>) or [DDC](<https://en.wikipedia.org/wiki/Display_Data_Channel>) — a frequent occurrence on contemporary systems. GPIOs are thus useful in a lot of contexts. 

GPIO lines seem to be especially prevalent in embedded systems; even so, there never seems to be enough of them. As one might expect, a system with dozens (or even hundreds) of GPIOs needs some sort of rational abstraction for managing them. The kernel has had such a mechanism since 2.6.21 (it was initially added by David Brownell). The API has changed surprisingly little since then, but that period of relative stasis may be about to come about to an end. The intended changes are best understood in the context of the existing API, though, so that is what this article will cover. Subsequent installments will look at how the GPIO API may evolve in the near future. 

Naturally, there is an include file for working with GPIOs: 

```
#include <linux/gpio.h>
```

In current kernels, every GPIO in the system is represented by a simple unsigned integer. There is no provision for somehow mapping a desired function ("the sensor power line for the first camera device," say) onto a GPIO number; the code must come by that knowledge by other means. Often that is done through a long series of macro definitions; it is also possible to pass GPIO numbers through platform data or a device tree. 

GPIOs must be allocated before use, though the current implementation does not enforce this requirement. The basic allocation function is: 

```
int gpio_request(unsigned int gpio, const char *label);
```

The `gpio` parameter indicates which GPIO is required, while `label` associates a string with it that can later appear in sysfs. The usual convention applies: a zero return code indicates success; otherwise the return value will be a negative error number. A GPIO can be returned to the system with: 

```
void gpio_free(unsigned int gpio);
```

There are some variants of these functions; `gpio_request_one()` can be used to set the initial configuration of the GPIO, and `gpio_request_array()` can request and configure a whole set of GPIOs with a single call. There are also "managed" versions (`devm_gpio_request()`, for example) that automatically handle cleanup if the developer forgets. 

Some GPIOs are used for output, others for input. A suitably-wired GPIO can be used in either mode, though only one direction is active at any given time. Kernel code must inform the GPIO core of how a line is to be used; that is done with these functions: 

```
int gpio_direction_input(unsigned int gpio);
        int gpio_direction_output(unsigned int gpio, int value);
```

In either case, `gpio` is the GPIO number. In the output case, the value of the GPIO (zero or one) must also be specified; the GPIO will be set accordingly as part of the call. For both functions, the return value is again zero or a negative error number. The direction of (suitably capable) GPIOs can be changed at any time. 

For input GPIOs, the current value can be read with: 

```
int gpio_get_value(unsigned int gpio);
```

This function returns the value of the provided `gpio`; it has no provision for returning an error code. It is assumed (correctly in almost all cases) that any errors will be found when `gpio_direction_input()` is called, so checking the return value from that function is important. 

Setting the value of output GPIOs can always be done using `gpio_direction_output()`, but, if the GPIO is known to be in output mode already, `gpio_set_value()` may be a bit more efficient: 

```
void gpio_set_value(unsigned int gpio, int value);
```

Some GPIO controllers can generate interrupts when an input GPIO changes value. In such cases, code wishing to handle such interrupts should start by determining which IRQ number is associated with a given GPIO line: 

```
int gpio_to_irq(unsigned int gpio);
```

The given `gpio` must have been obtained with `gpio_request()` and put into the input mode first. If there is an associated interrupt number, it will be passed back as the return value from `gpio_to_irq()`; otherwise a negative error number will be returned. Once obtained in this manner, the interrupt number can be passed to `request_irq()` to set up the handling of the interrupt. 

Finally, the GPIO subsystem is able to represent GPIO lines via a sysfs hierarchy, allowing user space to query (and possibly modify) them. Kernel code can cause a specific GPIO to appear in sysfs with: 

```
int gpio_export(unsigned int gpio, bool direction_may_change);
```

The `direction_may_change` parameter controls whether user space is allowed to change the direction of the GPIO; in many cases, allowing that control would be asking for bad things to happen to the system as a whole. A GPIO can be removed from sysfs with `gpio_unexport()` or given another name with `gpio_export_link()`. 

And that is an overview of the kernel's low-level GPIO interface. A number of details have naturally been left out; see [Documentation/gpio.txt](<https://lwn.net/Articles/532717/>) for a more thorough description. Also omitted is the low-level driver's side of the API, by which GPIO lines can be made available to the GPIO subsystem; covering that API may be the subject of a future article. The next installment, though, will look at a couple of perceived deficiencies in the above-described API and how they might be remedied.
