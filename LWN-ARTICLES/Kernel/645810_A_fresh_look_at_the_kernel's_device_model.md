---
title: "A fresh look at the kernel's device model"
url: https://lwn.net/Articles/645810/
date: "May 27, 2015"
category: Device model
author: "May 27, 2015 This article was contributed by Neil Brown"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

May 27, 2015

This article was contributed by Neil Brown

Understanding the Linux device model is (or should be, at least) central to working with device drivers in Linux — and drivers constitute over half of the kernel code. I've been working with a variety of device drivers at differing levels of involvement for some years but until recently I didn't feel that I really understood the model. This is potentially dangerous as, without a good understanding, it is easy to make poor choices.

The problem, or at least _my_ problem, is firmly rooted in the terminology. The device model involves things called "device" and "driver", "bus" and "class". To be able to understand the model, I need accurate definitions of these terms, and useful definitions are hard to find.

An [LWN article](<https://lwn.net/Articles/31185/>) from 2003 is an excellent example, as it clearly presents some definitions of the sort that can be found in other documentation and in the source code. It declares a device to be: ""A physical or virtual object which attaches to a (possibly virtual) bus"". This sounds good and highly general, but it doesn't actually match reality, even the reality of twelve years ago when the article was written.

For example, in the device model, a partition on a hard drive is a "device" much like the hard drive as a whole is. The hard drive as a whole may attach to a "bus", but the partition certainly doesn't: at best it attaches to the whole drive. Also, there are devices that don't attach to anything, let alone a "bus". The devices listed in directories under `/sys/devices/virtual` are not "attached" to anything. That "virtual" directory is not a special bus called "virtual", it is simply a place to put things that don't belong anywhere else.

Similar oversimplifications are found when trying to find definitions of the other objects. This is very likely because the driver model was still under development and the meanings that would end up being useful had not yet fully crystallized. Now, over a decade later, the available documentation still refers to the same terms and generally uses the same imprecise definitions.

#### The eye of the beholder

The epiphany that allowed me to form a coherent understanding of the device model was that none of these terms really have an external meaning at all. They are defined purely by the code that implements them. A device is simply any data structure that contains an embedded `struct device`, no more and no less.

Meaning comes only from the mind of the developer working with the code. Having multiple independent developers will likely result in multiple different meanings. The meanings that seem to be associated with the terms and that are found in documentation are the meanings that the early developers were thinking about. Those ideas have been revised over time, and other developers have had other thoughts.

The definition "anything with a `struct device`" may be accurate, but is not useful for someone considering the implementation of a new driver or modification to an old one. Similarly "what other developers are thinking" is too nebulous to be useful. With a bit of effort, and some carefully chosen examples, each of these can be fleshed out a bit and together form a picture that is, hopefully, a good start. So, to present my understanding of the device model, and particularly of those four terms, I will present some examples to show what other developers have thought, and what value a `struct device` provides.

The first example revolves around the [TCA6507](<http://www.ti.com/product/tca6507>) chip from Texas Instruments. This is a simple piece of hardware that accepts requests over an "i2c" bus, and responds by draining current through seven separate pins with various on/off patterns. This is particularly intended to pull electrical current through an LED to make it glow, but can equally pull current through a resistor to create logic 0 or 1 levels. This example is chosen because it is the most "device-like" of devices that I am familiar with — it perfectly fits the earlier definition.

The second example is the [workqueue](<https://www.kernel.org/doc/Documentation/workqueue.txt>) mechanism in Linux. It allows arbitrary tasks to be handed off for asynchronous completion, either promptly or after a delay, and will attempt to make optimal use of resources in doing so. It is also the least "device-like" thing I came across.

With these examples in mind, together with the previously mentioned block devices, we can proceed to those definitions.

#### Devices

A device is an instance. It is corresponds to a thing, or maybe an "object" in the most general sense of the word. A device gains its thing-hood primarily by a person thinking that it is something worth identifying. A device may sometimes correspond to a specific piece of hardware like an integrated circuit, but it could equally correspond to a collection of such circuits or just one component of the functionality of a circuit. Hardware need not exist at all — a device could be virtualized or could represent something that has no real physical equivalent at all. It is just a "thing".

The TCA6507 chip is represented in Linux by a device. Each of the seven controllable pins may be connected to something and this may lead to more devices. If a pin is attached to an LED, for example, then there will be a separate device that represents that LED, though arguably it could be seen as representing the signalling capability of the "LED plus pin" combination. Different people will probably look at this in different ways.

If a pin is connected to a "pull-up" resistor and used to signal a logic level, then it will be represented in Linux as a "GPIO" — General Purpose I/O pin. In terms of the device model, all of the pins that are configured as GPIOs are presented as a single "gpiochip" device. So while there is one device for _each_ LED, there is one device for _all_ GPIOs.

Each individual GPIO can be configured and used internally, or may be exported to user space through `sysfs`. When a GPIO is exported, a new device is created to represent just that one GPIO. This is visible as a directory under `/sys/class/gpio`; files are available there that can be used to set the output level to 1 or 0.

There are two important lessons in this example. One is that choices are context-dependent and probably very developer-dependent as well. Grouping GPIOs into a "chip" seems to make sense, while doing the same with LEDs doesn't seem to be a priority, though there has been a [suggestion](<http://comments.gmane.org/gmane.linux.leds/1929>) that there might be value in that. The second lesson is that one reason to make a "thing" into a "device" in the device model is so that it can appear in `sysfs` and be directly examined or manipulated.

Moving on to our second example we find something that is not at all "device-like". There are many "things" or "instances" in Linux that are not device-like and are not represented as devices: filesystems and processes are obvious examples. One that _is_ represented as a device is the workqueue.

The workqueue subsystem in Linux creates a "device" to represent each distinct queue. The [apparent reason](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=226223ab3c4118ddd10688cc2c131135848371ab>) for this is much like the reason for (sometimes) making devices for GPIOs — it allows the thing (i.e. the workqueue) to be examined and managed via `sysfs`. A thing doesn't _have_ to be a device to appear in `sysfs`, modules and filesystems are clear counter-examples to that idea. But making something a "device" is a relatively easy and well-worn path to `sysfs` access.

The compelling reason to use a "device" to represent some "thing" seems to be the interfaces. A "device" not only has standard interfaces in sysfs, it also has standard interfaces for power management, and may make use of internal services (like the [devm resource management API](<https://lwn.net/Articles/222860/>)) that are only provided to devices. There is also useful functionality for grouping "like" devices together, for varying definitions of "like".

#### Classes

A "class" is both the implementation of a set of devices, and the set of devices themselves. A class can be thought of as a driver in the more general sense of the word. The device model has specific objects called "drivers" but a "class" is not one of those.

All the devices in a particular class tend to expose much the same interface, either to other devices or to user space (via sysfs or otherwise). Exactly how uniform the included devices are is really up to the class though. It is not unusual for there to be optional aspects of an interface that not all devices in a class present. It is not unheard-of for some devices in the same class to be completely different from others.

So far we have met three classes in our examples. A device that represents an LED attached to a TCA6507 is a member of the "leds" class. This class supports the blinking, flashing, and brightness control features of physical LEDs. The class requires an underlying device to be available, such as a TCA6507 or a GPIO or any of various other options. This underlying device must be able to turn the LED on or off, may be able to set the brightness, and might even provide timer functionality to autonomously blink the LED with a given period and duty cycle. The "leds" class hides as much of this detail as it can to provide a simple abstract device.

Similar to the "leds" class is the "gpio" class; it provides a uniform interface to a variety of devices that can generate (output) or can sense (input) an electrical logic level. If the underlying device can generate an interrupt on a level change, "gpio" can translate that to a notification via `poll()` or can route it to the interrupt handler for some other device. The gpio class provides both the "gpiochip" devices and the individual "gpio" devices.

The third class we have met is the "disk" class which provides both whole hard drives and partitions within drives. As with the "leds" class, there are a few different interfaces to storage functionality that can be provided and the "disk" class presents a unified interface to that functionality.

There is some obvious similarity between the "gpio" class implementing both "gpios" and "gpiochips", and the "disk" class implementing both "disks" and "partitions". There are also differences. One of those is that "gpio" and "gpiochip" provide completely different interfaces, while "disk" and "partition" have a lot of commonality in their interfaces — both have block sizes and support I/O, but only a "disk" can be "removable".

Another, less obvious difference involves another aspect of the device model. Each "device" can have a "type". This type is often presented in the "uevent" file in the relevant `sysfs` directory. For example, the command:

```
grep DEVTYPE /sys/class/block/*/uevent
```

will show the type of every block device on your system. The different types in the "gpio" class are not known to the device model, though, so they are not reported by the `uevent` file. A human or a script would need to deduce the type from the device names if it was important.

Each of the classes listed here can be seen as providing a generic interface over a range of different hardware. This seems to be part of the original intention of the "class" facility. However "generic" isn't a very precise term. What one developer sees as "generic" another developer might see as "specific". These perspectives can change over time too, particularly if a simple or successful interface gets used more broadly than its initial context. 

To enforce this point it is worth briefly considering the "backlight" class of devices. A backlight for a graphics display can use a number of different underlying technologies, including a device of the "leds" class. So depending on your perspective, an LED might be a generic interface for signaling, or a specific underlying technology for backlighting. It depends on whose eye is beholding.

#### Buses

A "bus" is similar to a "class" in several ways, but it has an important difference. While a class is a complete implementation of the devices that are members of that class, the bus is only a partial implementation. For complete functionality, a bus usually works with a set of "drivers". A bus may implement some devices completely by itself, like a class does. Other devices will require a driver to be attached. The choice of driver can be made by the bus, by the driver (which can be asked if it "matches" a given device), or by a request through `sysfs`.

Our examples so far provide two examples of buses. The workqueue subsystem defines a "workqueue" bus to hold the devices that it creates for each workqueue. The set of drivers for this bus is empty. There are no separate implementations and no indication that there ever might be. This is probably the most minimal structure that a bus can have.

The other example, which has not yet been made explicit, is the [i2c](<http://en.wikipedia.org/wiki/I%C2%B2C>) bus, which is a standard two-wire bus for communicating between integrated circuits, typically all on a single board. This is the bus that is used to control the TCA6507, so the [leds-tca6507](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/drivers/leds/leds-tca6507.c?id=030bbdbf4c833bc69f502eae58498bc5572db736>) driver is written to work with the "i2c" bus in Linux.

The "i2c" bus in the device model is a collection of code that provides interface support between an individual driver like leds-tca6507 and some i2c bus master such as the [OMAP I2C](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/drivers/i2c/busses/i2c-omap.c?id=030bbdbf4c833bc69f502eae58498bc5572db736>) controller. It manages bus arbitration, retry handling, and various other protocol details. The "i2c" bus thus supports two different types of device, though the distinction is not directly visible in `sysfs` as the types are not given textual names.

The [`i2c_client_type`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/drivers/i2c/i2c-core.c?id=030bbdbf4c833bc69f502eae58498bc5572db736#n744>) includes all devices that are supported by separate drivers and represent hardware that can be communicated with via the i2c protocol. The [`i2c_adapter_type`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/drivers/i2c/i2c-core.c?id=030bbdbf4c833bc69f502eae58498bc5572db736#n1210>), instead, is implemented in the i2c bus code without using a separate driver. It represents the whole bus and exposes a character-special device in `/dev` (e.g. `/dev/i2c-0`) that can be used to interact directly with i2c clients, bypassing any driver.

By providing code and device support to both the client (or slave) side and the adapter (or master) side of an electrical bus, the Linux "i2c" bus very clearly represents the whole i2c bus. When there is an electrical bus to represent, a device model bus will often fill that role. When there is no electrical bus, as with workqueues, a bus might represent something else entirely.

Devices in a bus tend to reflect specific hardware rather than generic functionality, but once again it is hard to draw a clear line. One of the many drivers for the "usb" bus is the "usbhid" driver that supports any mouse, keyboard or similar "human interface device". This isn't really very specific.

#### When is a bus not a bus?

I had a particular reason for choosing an "leds" device as one of the examples, and that is because the "leds" class has a particularly interesting structure. Each "leds" device exposes a "trigger" attribute in sysfs. This file contains a list of all possible triggers, the currently active one surrounded by brackets. For example:

> 

```
>     none usb-gadget usb-host cpu0 cpu1 cpu2 cpu3 cpu4 cpu5 cpu6 cpu7
>     AC-online BAT0-charging-or-full BAT0-charging BAT0-full
>     BAT0-charging-blink-full-solid [mmc0] rfkill1 phy0rx phy0tx phy0assoc
>     phy0radio phy0tpt rfkill2 rfkill3 rfkill33
>
```

Writing the name of some trigger to this file will locate the driver for that trigger, loading a module if necessary, and will configure the device to use that trigger. This may involve presenting different attributes via `sysfs`. This mechanism for binding a trigger to an "leds" device is extremely similar to the mechanism that a bus provides to bind a driver to a device. Lots of the details are different, but the core functionality and purpose are the same.

We saw earlier that the workqueue subsystem, despite being a bus, had no separate drivers and so could have been a class. Here we see that "leds", despite being a class, has a number of separate drivers (called "triggers") and so could have been a bus. This seems to emphasize that fact that the choice of bus or class, like the particular role of a device, is truly in the eye of the beholder. There is no firm external meaning.

We can get a hint of what meaning one developer saw by examining the [declaration](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/kernel/workqueue.c?id=030bbdbf4c833bc69f502eae58498bc5572db736#n4911>) and registration of the "workqueue" bus. It is declared:

```
static struct bus_type wq_subsys = { /* ... */ };
```

and registered:

```
return subsys_virtual_register(&wq_subsys, NULL);
```

So, while it is a `bus_type`, it is named as a "subsys" or "subsystem" and registered as a "virtual subsystem". It seems the eye of at least one developer looked at a "bus" and saw a "subsystem".

There appears, both here and elsewhere, to be a desire to discard the separate concepts of "class" and "bus" and instead just have "subsystems". A subsystem would be exactly what a bus is, but without all the baggage that comes with the name. As a class provides nothing that is not provided by a bus, it could simply be dropped. Whether this transition will ever be complete remains to be seen.

#### A lesson learned

This, then, is the lesson of the driver model: the implementation provides functionality, not meaning. The meaning comes from the thoughts of developers and is coherent or disjoint in the same measure that those developers are of one mind, or not.

A "device", is just a thing that provides and consumes interfaces. It represents an idea more than it represents any particular hardware. A "bus" is better known as a "subsystem" and is some code to implement devices together with a mechanism to attach separate "drivers" to those devices. A bus (or subsystem) is also the set of device associated with that code. A "class" is just a bus without the mechanism for separate drivers. And a "driver" is code that works in concert with a particular bus to implement certain devices. 

These definitions helped me to lose the baggage that I tried to associate with "device" and "bus" and provided clearer understanding. But it is not yet enough for a complete understanding. Devices do not exist in isolation — they have those interfaces for a reason. A full understanding of the device model requires some understanding of how they all fit together and a good place to look at that is in `/sys/devices` which contains all the devices on a particular system.

So next week we will dive in to `/sys/devices` and find out how that directory tree is structured, what it contains, and what more we can learn about devices.
