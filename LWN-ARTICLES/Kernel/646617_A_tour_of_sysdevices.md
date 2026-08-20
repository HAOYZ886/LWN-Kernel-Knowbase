---
title: A tour of /sys/devices
url: https://lwn.net/Articles/646617/
date: "June 3, 2015"
category: Device model; Sysfs
author: "June 3, 2015 This article was contributed by Neil Brown"
---

June 3, 2015

This article was contributed by Neil Brown

Modern Linux systems have a directory tree at `/sys/devices` that contains information about all of the "devices" represented in the device model. Having [clarified](<https://lwn.net/Articles/645810/>) exactly what each of "devices", "buses", and "classes" are (or maybe what they aren't), I am now in a position to address a shortcoming I had to admit in a [recent article](<https://lwn.net/Articles/604413/>) where I stated that `/sys/devices` ""has a structure that, in all honesty, is rather hard to describe"".

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

`/sys/devices` is part of the `sysfs` virtual filesystem; it presents devices as directories arranged in a hierarchy. It includes various files to allow details of each device to be examined and sometimes to be changed. Seeing how they all plug together can give insights into the device model that are not evident by just examining individual devices. To describe this directory structure in detail there are three particular concepts we need to examine beyond the device, bus, and class that we met last time. We need to explore parentage, attributes, and namespace management.

#### Parenthood

Each device (or "thing") in the Linux device model can have a "parent" device. In the examples from last week, the "tca6507" device was the parent of some "`leds`" devices, and itself had a parent that was an i2c adapter device, such as "i2c-1". The parent of the device representing a partition on a hard drive is the device representing the hard drive as a whole. "Workqueue" devices do not have parents. This variety leads to the question of what, exactly, is a "parent". There are several different, though related, concepts that are tied up in this idea of "parenthood".

#### Parent as connection point or service provider

Firstly there is the idea of "addressability" and the related idea of "connectedness". When a device represents a piece of hardware, the parent often represents a connection to that device. This parent knows how to interpret the "address" of the device, and how to send instructions to that address, or how to transfer data to or from that address. The parent doesn't know much about what those instructions or data might do; that knowledge is in the device itself (and its driver).

Closely related to connection is the idea of service provision. When a device represents a piece of hardware, the primary service it needs from a parent device is communication with the CPU. When a device represents something else, the primary service requirement might be different.

Many devices that are implemented by classes do not exactly represent hardware but instead represent an abstraction. Devices in the "disk" class represent anything that can provide read or write access to addressable blocks of storage. Any device that can facilitate that access might reasonably find itself as the parent of a "disk" device. Sometimes the thing that provides service is not a device — it might be a file or a network connections or a set of devices. In those cases the disk device won't have a parent.

I like to picture these two complementary ideas (connection and service provision) by imagining the path a request takes from user space all the way to ultimate physical reality. A request such as "turn the LED on" or "read that block of data", enters the device hierarchy at one of the leaf devices, possibly by a `write()` or `ioctl()` on a device node in `/dev` or by a write to a sysfs attribute file. The request then propagates up the tree, being translated at each stage as is appropriate to each level. Some levels might add extra detail to the address, some levels might wrap "protocol headers" around the message, or divide the message up into components. Another level might add a retry mechanism or might impose a media-access protocol.

Once the message gets to the top of the hierarchy it is transmitted on a core system bus (probably via writing to one or more memory addresses), at which point the request travels through the hardware from controller to controller in a rough analog of traveling back down the tree of devices and out to the physical leaves that are the hardware that will actually pass current through an LED, or read the data off some sort of media.

To make this more concrete, consider an `leds` device with the "`default-on`" trigger selected, and then consider writing "`100`" to the "`brightness`" file, which implies a request to set the brightness to `100/255` of the maximum brightness (assuming that `max_brightness` is `255`). The full path to that file might be:

```
/sys/devices/platform/omap_i2c.2/i2c-2/2-0045/leds/gta04:green:power/brightness
```

The trigger driver leaves that request for the `leds` core to handle; the core determines whether one of two possible interfaces (`brightness_set()` or `brightness_set_sync()`) is available and calls it if so; otherwise it returns an error. When an `leds` device is registered by a call to `led_classdev_register()`, a `struct led_classdev` must be passed; it contains various function pointers and other settings that provide access to the underlying mechanism. So the `leds` driver turns the "set brightness" request into one of two possible function calls into the parent device.

If that parent device is the TCA6507 we discussed previously (named "2-0045" above) then `tca6507_brightness_set()` is called. The TCA6507 does not provide completely independent control of the seven output pins. Apart from fully on and fully off, there are at most three distinct brightness levels from a set of 16 possible levels. If the request level matches a level already configured, that can be assigned to the appropriate pin. If not, and there are fewer than three in use, then the new level can be programmed to a free slot. If all brightness slots are in use, the closest to the requested value is used.

In any case, the decisions need to be sent to the TCA6507 which, like most simple I2C devices, appears to contain a set of registers. So the `leds-tca6507` driver converts the `brightness_set()` request into one or more "write to register" requests. Those requests must, in turn, be passed to the device via an I2C bus. 

When an I2C driver is attached to a device, its `probe()` function is called with a `struct i2c_client` parameter. When the driver wants to write to a register it calls `i2c_smbus_write_byte_data()` or a similar function, giving it the same `i2c_client` handle, the register address, and the data. "`smbus`" refers to "System Management Bus" which is a subset of the I2C protocol — drivers that only need that subset are encouraged to use the `smbus` interfaces.

The i2c/smbus code (implementing device "i2c-2") takes the "register write" request and converts that into a message to transmitted on the i2c bus; it contains a command, an address, and data. The underlying I2C adapter driver can provide a couple of different functions, such as `smbus_xfer()` or `master_xfer()`, the latter being a lower-level function that needs more support in the I2C bus code. These functions may be called repeatedly to effect retry on certain failure modes, and are only called after `i2c_lock_adapter()` has successfully gained exclusive access to the bus.

So now those "write to register" requests have become "message transfer" calls with locking and retry.

If that underlying I2C adapter is the OMAP I2C adapter ("omap_i2c.2"), it will ensure the clocks for the adapter are enabled, make sure it isn't being held in reset, and wait to make sure the bus isn't being used by some other bus master. The driver will then load the size and address of each message into the DMA engine of the adapter so that it can copy the messages out onto the bus. This is all done by writing directly to registers in the I2C adapter using the system memory bus.

At this point the message hasn't quite reached the top of the `/sys/devices` hierarchy; that step brings in the "platform" device, which represents the whole platform and the main system memory bus, used by the I2C driver to talk to the I2C adapter. The memory bus driver doesn't get involved in copying bytes out, but it was involved when the devices were registered in that it effectively told the I2C driver what address to use to talk to its hardware.

The memory bus carries these writes to the I2C adapter, which fetches the messages from memory and serializes them out onto the I2C bus. The TCA6507 collects the bits off the bus, places them in the addressed register, and then loads internal counters as appropriate so that it can turn the particular pin on and off with the right duty cycle to achieve the requested brightness (or something close to it). This pulls current through the LED so that it generates light some of the time, and doesn't for the rest of the time. Our eyes, which naturally integrate those high frequency changes, see something close to the requested brightness level. It seems like a long journey, but it gets from: 

```
echo 100 > brightness
```

all the way to my eyes faster than I can blink.

While this is a helpful picture, it is a somewhat idealized picture. It is important to be aware that the "primary" service provider is not necessarily the only service provider. A specific LED can be configured to blink whenever some specific battery other than the primary is charging, for example. So both the LED and the charge monitoring hardware provide services to the "power supply" device.

In most cases where there are multiple services being provided, there will be one that is fairly obviously primary. Other times there is not. A RAID array receives service of equal value from a number of different block devices. And it could stop receiving that service at any moment if one fails. So there is no clear primary service provider for a RAID array, and indeed md/raid devices do not have a parent.

#### Parent as discoverer

The third idea is again related to the others but provides a helpful alternate perspective. It is "discovery". Every device must be deliberately added to the device model; this process is referred to a "device discovery". When driver code, acting on behalf of a particular device, discovers another device, it needs to assign a parent to the new device; it will typically use the current device as that parent.

Sometimes this discovery is performed by probing a (physical) bus to see what devices are connected to it. A similar situation happens when a bus supports hotplug and the bus controller reports that a new device has been attached. In these cases, the device that detects the new device is almost certainly the device that can address and send commands to the new device, so it is logical for it to be the parent.

Other times, this discovery happens by examining a configured description of available devices. This might be based on information extracted via a BIOS service such as ACPI, it might be encoded using a device tree, or it could be encoded directly in the kernel using arrays of "`struct platform_device`" and calls to `platform_add_devices()`, though that approach is deprecated. When ACPI is used, it will provide a top-level device similar to "platform", which is parent to the devices that are discovered and managed using ACPI, even though ACPI is not exactly a physical connection.

#### Parent as power source

Finally there is the idea of power management. As power management was apparently one of the driving forces that led to the creation of the device model and `/sys/devices`, you might expect that power management would be fairly central. The reality is ... more complex.

There are two sides to power management. One is system suspend or system hibernate, where the focus is putting as many devices as possible to the lowest power state. The other is runtime power management, where the focus is placing individual devices into a low power state whenever they aren't actively in use. It might seem reasonable to follow parent-child links in a depth-first order, turning off children and then their parents. But that isn't quite what is done.

In the case of system suspend, there is actually a completely separate structure that is used to determine the order for shutting down devices. As I [described in 2012](<https://lwn.net/Articles/505683/>), the power-management code maintains a linear list of devices called "`dpm_list`" that is used to sequence suspend and resume. This list is created roughly in order of device discovery, which is roughly how the tree of devices grows out to the leaves. So processing `dpm_list` in order will normally visit parents before children. However power often isn't managed in exactly the same way as device addressing, so the power management system needs a little more control, which it gets by having its own list.

Runtime power management does make some use of parent/child relationships, but not always. Possibly not even often.

For connections that are designed to carry both power and control, such as USB, it makes lots of sense for the bus controller to remain powered on (and providing power) whenever an attached device is powered on. However it may well make sense for the bus controller to enter a low-power state while the device continues to do useful work, particularly if the device can trigger some sort of wakeup event.

For buses that do not carry power, such as i2c, it makes no sense to directly link the power state of the bus controller with the power state of the attached device. Typically an i2c controller will power down (stopping its clock) whenever it doesn't have any commands to send. When a device driver needs to talk to its device it will wake up the controller (its parent), send the message, and let the controller go back to sleep, even while the device itself is fully active (blinking those LEDs).

In the runtime power management system the default behavior is that, when a child device is powered on, its parent must be powered on as well. This default can be overridden by a call to `pm_suspend_ignore_children()`. Given that many buses do not carry power, I was at first surprised at how few calls there are to this function: in 4.1-rc1 there are only 14 calls that enable the ignoring of children.

Part of the reason for this small number is that the default as stated misses an important detail. It only applies to devices that actually support different runtime power-management states. If both a parent and child support runtime power management, then the default will keep the parent powered when the child is active. If either does not support runtime power management, then no connection is implied. This means that when a device wakes up, that wakeup will only propagate up the tree until it reaches a device that sets "ignore_children" or a device that doesn't support runtime power management at all. Then the propagation will stop.

There are actually quite a lot of devices that do not support runtime power management. Most devices implemented by classes do not.

```
$ cat /sys/class/*/*/power/runtime_status | sort | uniq -c
        318 unsupported
```

Many devices implemented by buses do not either:

```
$ cat /sys/bus/*/devices/*/power/runtime_status | sort | uniq -c
         57 active
         11 suspended
        274 unsupported
```

So while runtime power management can make use of the parent link, it doesn't to a great extent. The parent very often is _not_ the source of power, only of addressing and control.

Despite the apparent richness of meaning we find in parenthood, the device model doesn't really use it much. As a message passes "up" the hierarchy, it doesn't follow the "`->parent`" links of the device model, but uses references that were provided to each device when it was initialized. The runtime power management code does make direct use of these `->parent` links, but not very much. It could just as easily make use of explicit dependencies, just as it does when there are dependencies that are not reflected in the hierarchy.

Possibly the biggest user of these parent links is the "udev" utility, which responds to device addition and removal events in a highly configurable way. This configuration can depend on information about arbitrarily distant ancestors. On the other hand, it cannot depend at all on related devices that are not direct ancestors.

#### What role for attributes?

Attributes are arbitrary details about a given device; they are presented via small files in `/sys/devices` known as "attribute files". The particular content and use of these files was [discussed here some years ago](<https://lwn.net/Articles/378884/>) and has not significantly changed since so will not be addressed further.

Each device is presented by a directory in `/sys/devices`; most attribute files appear as files directly in those directories. There are exceptions, though, and while they are not particularly interesting, they need to be understood. These exceptions collect a group of attributes together into a subdirectory. The most prevalent such subdirectory is `power`, which contains attributes relating to the power status of the device. Every device has this subdirectory.

The second most prevalent on the machine on which I am typing are the `capabilities` and `id` subdirectories that various devices in the `input` class contain. Additionally, `cpu` devices can have a two-level attribute tree with `state` directories in a `cpuidle` directory. The important point here is that, when you find a directory in the `devices` tree, it might just be a collection of attributes. The reason why they have been collected will vary from case to case.

Note that "classes" and "buses" can also have attributes. These affect the whole subsystem and so are somewhat similar to module parameters. They do not appear in "`/sys/device`", only in "`/sys/bus`" or "`/sys/class`", so they are out of scope for this article.

#### Managing namespaces

While Linux does have the [Linux Assigned Names And Numbers Authority](<http://www.lanana.org/>), that authority does not oversee the assignment of names to classes, buses, devices, or attributes. In generally we just have to "play nice", try to catch inappropriate duplication during review, and fall back on the kernel complaining if a name gets used twice in one namespace.

Within `/sys/devices` there can often be names from distinct namespaces appearing in the one directory, so it is worth knowing what namespaces there are and what support there is for conflict avoidance. Firstly, there is a namespace for buses (`/sys/bus`) and another for classes (`/sys/class`). These are currently kept separate and are largely disjoint anyway. On my laptop I find:

```
$ comm -12 <(ls /sys/bus) <(ls /sys/class)
        mei
```

so only one name in common — which is more than I expected.

Each bus and each class defines a namespace for devices that are implemented by that bus or class. Typically a simple syntactic rule will distinguish between different types of devices in the bus. So in the i2c bus, devices that represent an adapter are "i2c-%d" while devices that represent attached hardware are "%d-%04x".

Each bus or class, and to some extent each driver, also defines a namespace of attributes. A driver must clearly not define attributes that conflict with names used by its bus, but different drivers on the same bus could safely use the same name (possibly for different purposes, though that wouldn't be wise). There is also a namespace of system-imposed attributes, of which we have already seen one example: the "power" set of attributes that is imposed on every device. Clearly subsystems and drivers must avoid these.

Reserving the single name "power" for all power management attributes makes it practical to add new power management attributes, such as `pm_qos_latency_tolerance_us` which was [added](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=2d984ad132a87ca2112f81f21039493176a8bca0>) a little over a year ago. This approach of reserving just one name as a directory to minimize namespace pollution is also used to some extent to reduce the risk of a device name in one subsystem conflicting with an identically named device in another subsystem. When this is done, a directory named for the subsystem is used to contain all relevant devices of that subsystem. But this is only a sometimes thing, as we will shortly see.

#### Bringing it all together

It seems easiest to describe the whole tree by starting in the middle. Every directory under `/sys/devices` that contains a file called "`uevent`" represents a device. This file can be written to in order to synthesize "ADD" events, "REMOVE" events, or other events that can be processed by udev. It can be read to show context information that accompanies those events.

In each device directory there is also a "`power`" directory of attributes, as we have already seen, and often a "`subsystem`" symlink that points to the owning subsystem in either `/sys/bus` or `/sys/class`.

While any device must be implemented by some subsystem, it may not actually be linked into the list of devices for that subsystem. For example, the "usb" subsystem creates "`endpoint`" and "`port`" devices but does not identify them as belonging to the "usb" subsystem; as a result they don't get the "`subsystem`" symbolic link and do not appear in "`/sys/bus/usb/devices`".

A device may also have a "`device`" link which is of legacy interest only and usually points to a recent parent, and (in Linux 4.1) an "`of_node`" link that, if the device was described by a device tree, will be a link to the relevant location in `/sys/firmware/devicetree`. Other files and symbolic links will be device-specific or subsystem-specific attributes. Other directories can be one of three sorts of things.

Subdirectories of a device directory may also be device directories if the parent is a "class" or "bus" device. This case is easily recognized by looking for "`uevent`" in the child directory.

Subdirectories of a device directory may be grouping directories for all children of a particular "class", but only when the parent device is a "bus" device. So the bus device for the TCA6507 device has a directory called `.../i2c-2/2-0045`, and it contains a grouping directory `leds` that contains a further subdirectory for each `leds` device. This case is also easily recognized as the child has the same name as a class, and its children contain `uevent` files.

Finally, subdirectories of a device directory can be grouping directories for attributes and can contain arbitrary attribute files, symbolic links, and further sub-directories, but no device directories.

When a device has a parent in the device model, the parent of the device directory will either be the device directory of the parent, or will be an intermediate grouping directory named after the class of this device. When a device does not have a parent, there are a few different possibilities.

When a bus is created, it can indicate whether parentless devices should be placed in `/sys/devices/virtual`, or `/sys/devices/system`, or directly in `/sys/devices`. Classes don't have that flexibility: all parentless class devices are placed in `/sys/devices/virtual`. Being class devices, they have an intermediate directory named after the class, and the device has its device directory attached within that intermediate directory. So md RAID block devices appear as `/sys/devices/virtual/block/md0` for example.

#### Summing up

"Entities must not be multiplied beyond necessity" is a popular formulation of [Occam's Razor](<http://en.wikipedia.org/wiki/Occam%27s_razor>), but could equally be used as advice for software development. Unfortunately the current state of `/sys/devices` doesn't seem to meet that goal very well as it seems filled with multiplicity, though that is doubtless due to baggage that has been carried along as the design evolved.

Some of the multiplicities are quite valuable: having directories used for both the actual devices and for namespace control seems very sensible, though it is unfortunate that the namespace control is a little _ad hoc_. Similarly, having attributes that can be imposed system-wide (`power`, `subsystem`, `of_node`), subsystem-wide, or per-device is a fairly natural choice.

Other multiplicities are largely a matter of interpretation such as the various ways of looking at the meaning of parenthood. It could be that the concept of "parent" isn't really well defined enough to deserve to be enshrined in the hierarchy, but it does seem to be useful to some degree.

And finally there are the multiple subsystems ("class" and "bus") and the multiple roots (`/sys/devices`, `/sys/devices/virtual`, and `/sys/devices/system`) that don't appear to add much value. The fact that there are so many places for parentless devices lends weight to the possibility that parents aren't really as important as we might imagine. 

While it might be nice to have more simplicity, the more important thing is understanding what we do have. There is certainly a lot of value in `/sys/devices`, even if some aspects are not as valuable as others. Understanding what goes where, which parts are important, and which parts don't deserve so much attention, is really where it is most useful to focus. So while it might have been hard to describe, for myself at least it was worth describing.
