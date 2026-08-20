---
title: The USB composite framework
url: https://lwn.net/Articles/395712/
date: "July 14, 2010"
category: Device drivers; USB
author: "July 14, 2010 This article was contributed by Michal \"mina86\" Nazarewicz"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

July 14, 2010

This article was contributed by Michal "mina86" Nazarewicz

Linux is widely used in mobile devices, which should not come as a surprise. It is a powerful and versatile system, and one of its strengths is its support for USB devices of all kinds. That includes "gadgets" -- devices that act as USB slaves -- like USB flash drives (i.e. pendrives). The USB composite framework makes writing drivers for these kinds of devices relatively easy. 

As users keep more data on their mobile devices, the demand for interoperability with desktop computers increases. No one wants to buy a special cable or a "docking station" just to copy a few photos. What users want is to connect the device via a USB cable and get it working out of the box. Linux can give that to them.

Have you ever wondered how this actually works? What happens behind the scenes when a USB connection is established? Better yet, have you wondered how to write a USB gadget for your new and shiny embedded evaluation board?

In this article, I will try to shed some light on that topic. 

#### USB overview

The Universal Serial Bus (or USB) standard defines a master-slave communication protocol. This means that there is one control entity (a master or a host), which decides who can transmit data through the wire. The other entities (slaves, devices, or gadgets) must obey and respond to the host's requests. Slaves do not communicate with each other. A host is usually a desktop computer, while the gadgets are devices such as mice, keyboards, phones, printers, etc.

People are used to seeing Linux systems in the master or host role on a USB bus. But the Linux USB stack also provides support for the slave or gadget role -- the device at the other end of the wire. For example, when one connects a pendrive to a Linux host, it handles it with a [usb-storage](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=blob;f=drivers/usb/storage/usb.c;h=bbeeb92a213101d7eeab4314c91532da707247f7;hb=e40152ee1e1c7a63f4777791863215e3faa37a86>) driver. However, if we had a Linux machine with a USB Device Controller (or UDC), we could run Alan Stern's [File Storage Gadget](<http://git.kernel.org/?p=linux/kernel/git/hpa/linux-2.6-allstable.git;a=blob;f=drivers/usb/gadget/file_storage.c;h=b49d86e3e45b639d2d3f4760d027e38a0fd52d44;hb=e40152ee1e1c7a63f4777791863215e3faa37a86>) (or FSG). FSG is, as its name implies, a gadget driver which implements the USB mass storage device class (or UMC). That would allow the machine in question to act as a USB drive (aka pendrive). 

When a device is connected, an enumeration process begins. During this process, the device is assigned a unique 7-bit identifier used for addressing. As a consequence, up to 127 slaves (including hubs) can be connected to a single host.

Communication is based on logical pipes which join the master with one of a slave's endpoints (or EPs). There can be up to 16 endpoints (numbered from 0 to 15) on a device. Endpoint zero (or EP0) is reserved for setup requests (eg. a query for descriptors, request to set a particular configuration, etc.).

Pipes are unidirectional (one-way) and data can go to (via an IN endpoint) or from (via an OUT endpoint) the host. (It is important to remember that from a slave's point of view, an IN endpoint is the one it writes to and an OUT endpoint is the one it reads from.) There are also four transfer modes: bulk, isochronous, interrupt, and control.

![\[USB Gadget's descriptors structure\]](https://static.lwn.net/images/2010/usb-gadget.png)

Endpoints are grouped into interfaces which are then grouped into configurations. Different configurations may contain different interfaces, as well as have different power demands. All that information is saved in various descriptors requested by the host during enumeration. One can see them using the `lsusb` tool. Here is the stripped-down (and annotated) output for a Kingston pendrive:

```
Bus 001 Device 004: ID 0951:1614 Kingston Technology
        Device Descriptor:
          idVendor           0x0951 Kingston Technology
          idProduct          0x1614
          bNumConfigurations      1     _[only one configuration]_
          Configuration Descriptor:     _[the first and only config]_
            bNumInterfaces          1   _[only one interface]_
            MaxPower              200mA
            Interface Descriptor:       _[the first and only intf.]_
              bNumEndpoints           2 _[two endpoints]_
              bInterfaceClass         8 Mass Storage
              bInterfaceSubClass      6 SCSI
              bInterfaceProtocol     80 Bulk (Zip)
              Endpoint Descriptor:      _[the first endpoint]_
                bEndpointAddress     0x81  EP 1 IN
                bmAttributes            2
                  Transfer Type            Bulk
                  Usage Type               Data
              Endpoint Descriptor:      _[the second endpoint]_
                bEndpointAddress     0x02  EP 2 OUT
                bmAttributes            2
                  Transfer Type            Bulk
                  Usage Type               Data
```

After the host receives the descriptors and learns what kind of a gadget has been connected, it can choose a configuration to be used and start communicating. At most one configuration can be active at a time.

#### Linux USB composite framework

There is, however, another module that implements UMC: my [Mass Storage Gadget](<http://git.kernel.org/?p=linux/kernel/git/hpa/linux-2.6-allstable.git;a=blob;f=drivers/usb/gadget/mass_storage.c;h=705cc1f76327c6cc953f4b2efcc6788a77ef760d;hb=e40152ee1e1c7a63f4777791863215e3faa37a86>) (or MSG). The obvious question is, why there are two drivers that seem to do the very same thing. This has something to do with the Linux USB composite framework.

The "old way" of creating gadgets is to get the specification and implement everything as a single, monolithic module. [Gadget Zero](<http://git.kernel.org/?p=linux/kernel/git/hpa/linux-2.6-allstable.git;a=blob;f=drivers/usb/gadget/zero.c;h=807280d069f96e815ea11c1eddddb29cbe6692fd;hb=e40152ee1e1c7a63f4777791863215e3faa37a86>), File Storage Gadget, and [GadgetFS](<http://git.kernel.org/?p=linux/kernel/git/hpa/linux-2.6-allstable.git;a=blob;f=drivers/usb/gadget/inode.c;h=de8a83803505032542db52582f5258bdcb6473f1;hb=e40152ee1e1c7a63f4777791863215e3faa37a86>) are examples of such gadgets.

This approach has two rather big disadvantages:

  * many of the common USB functionalities (core device setup requests on EP0) have to be implemented in each and every module; and
  * it can be tricky to combine the code from several gadgets into a new gadget with combined functionality.

For those reasons, David Brownell came up with the [composite framework](<http://git.kernel.org/?p=linux/kernel/git/hpa/linux-2.6-allstable.git;a=blob;f=include/linux/usb/composite.h;h=738ea1a691cb8a5dab64d631792871273fd45053;hb=e40152ee1e1c7a63f4777791863215e3faa37a86>) which has two advantages over the old approach:

  * all of the core USB requests are implemented by the framework; and
  * a single functionality or a USB composite function is developed separately from other functions as well as from the USB bus logic that is not directly related to this function. Later, such functions are combined using the composite function to form a composite gadget.

![\[USB Composite Gadget's descriptors
    structure\]](https://static.lwn.net/images/2010/usb-composite-gadget.png)

From a composite gadget's perspective, a device has some functions grouped into configurations. One function may be present in any number of configurations. Each function may have several interfaces and other descriptors but that is transparent to the kernel module.

Put on top of the "raw" USB descriptors structure, a USB composite function can be regarded as an abstraction for a group of interfaces.

That is another excellent property of the framework -- most implementation details are hidden "under the hood" and one does not need to think about them when developing a gadget. Instead of thinking about endpoints and interfaces, one thinks about functions. Therefore, FSG is a gadget developed in the "old way", whereas MSG is a composite gadget which uses only one composite function -- the [Mass Storage Function](<http://git.kernel.org/?p=linux/kernel/git/hpa/linux-2.6-allstable.git;a=blob;f=drivers/usb/gadget/f_mass_storage.c;h=f4911c09022e92c2c16cc1b294045a56a0aca231;hb=e40152ee1e1c7a63f4777791863215e3faa37a86>) (or MSF). As a matter of fact, MSF has been created from FSG to allow for the creation of more complicated drivers that would have UMC as part of their functionality.

#### Overall driver structure

In this article, I will try to explain how to create a mass storage composite gadget. It is in the kernel already, but let's forget that FSG and MSG exist for a moment.

What is great about Linux, is that a lot has already been done and one can get results with relatively little effort. As such, I will show how to create a working driver using MSF and some "composite glue".

I will start with the structure of the [module](<http://lwn.net/Articles/395889/>), while skipping the details of the Mass Storage Function. The first step is to define a device descriptor. It stores some basic information about the gadget:

```
static struct usb_device_descriptor msg_dev_desc = {
        	.bLength =		sizeof msg_dev_desc,
        	.bDescriptorType =	USB_DT_DEVICE,
        	.bcdUSB =		cpu_to_le16(0x0200),
        	.idVendor =		cpu_to_le16(FSG_VENDOR_ID),
        	.idProduct =		cpu_to_le16(FSG_PRODUCT_ID),
        };
```

The [`usb_device_descriptor` structure](<http://git.kernel.org/?p=linux/kernel/git/hpa/linux-2.6-allstable.git;a=blob;f=include/linux/usb/ch9.h;h=e58369ff8168b7a81058efa3b060a807284ff73f;hb=e40152ee1e1c7a63f4777791863215e3faa37a86#l217>) has some more fields but they are not required or not important for our module. What has been set is:

`bLength` and `bDescriptorType`
    A standard fields each descriptor has.
`bsdUSB`
    The version of USB specification the device supports encoded in BCD (so `0x200` means `2.00`).
`idVendor` and `idProduct`
    Each device must have a unique vendor and product identifier pair. To avoid collisions, companies (vendors) can buy a vendor ID which gives them a namespace of 65536 product IDs to use. NetChip has donated some product IDs to the Linux community. Later, the Linux Foundation got the whole vendor ID for use with Linux. [`FSG_VENDOR_ID`](<http://git.kernel.org/?p=linux/kernel/git/hpa/linux-2.6-allstable.git;a=blob;f=drivers/usb/gadget/storage_common.c;h=868d8ee86756671192a2fd1e93c866ae06121229;hb=e40152ee1e1c7a63f4777791863215e3faa37a86#l60>) is actually NetChip's vendor ID and, along with `FSG_PRODUCT_ID`, that is what FSG uses.

The next step is to define an USB configuration which will be provided by the driver. It is described by a [`usb_configuration` structure](<http://git.kernel.org/?p=linux/kernel/git/hpa/linux-2.6-allstable.git;a=blob;f=include/linux/usb/composite.h;h=738ea1a691cb8a5dab64d631792871273fd45053;hb=e40152ee1e1c7a63f4777791863215e3faa37a86#l157>) which, among other things, points to a `bind` callback function. Its purpose is to bind all USB composite functions to the configuration. Usually, it is a simple function, as most of the job is done prior to its invocation.

Put together it looks as follows:

```
static struct usb_configuration msg_config = {
        	.label			= "Linux Mass Storage",
        	.bind			= msg_do_config,
        	.bConfigurationValue	= 1,
        	.bmAttributes		= USB_CONFIG_ATT_SELFPOWER,
        };
    
        static int __ref msg_do_config(struct usb_configuration *c)
        {
        	return [fsg_add](<http://git.kernel.org/?p=linux/kernel/git/hpa/linux-2.6-allstable.git;a=blob;f=drivers/usb/gadget/f_mass_storage.c;h=f4911c09022e92c2c16cc1b294045a56a0aca231;hb=e40152ee1e1c7a63f4777791863215e3faa37a86#l2968>)(c->cdev, c, &msg_fsg_common);
        }
```

The `msg_config` object specifies a label (used for debug messages), the `bind` callback, configuration's number (each configuration must have a unique, non-zero number), and indicates that the device is self powered. All that the `msg_bind` does is bind the MSF to the configuration.

That definition is then used by the `msg_bind()` function, which is a callback to set up composite functions, prepare descriptors, add all configurations supported by the device, etc.:

```
static int __ref msg_bind([struct usb_composite_dev](<http://git.kernel.org/?p=linux/kernel/git/hpa/linux-2.6-allstable.git;a=blob;f=include/linux/usb/composite.h;h=738ea1a691cb8a5dab64d631792871273fd45053;hb=e40152ee1e1c7a63f4777791863215e3faa37a86#l237>) *cdev)
        {
        	int ret;
    
        	ret = msg_fsg_init(cdev);
        	if (ret < 0)
        		return ret;
    
        	ret = [usb_add_config](<http://git.kernel.org/?p=linux/kernel/git/hpa/linux-2.6-allstable.git;a=blob;f=drivers/usb/gadget/composite.c;h=09289bb1e20fcf984b388b32f55a6c404cc1dcb3;hb=e40152ee1e1c7a63f4777791863215e3faa37a86#l469>)(cdev, &msg_config);
        	if (ret >= 0)
        		set_bit(0, &msg_registered);
        	[fsg_common_put](<http://git.kernel.org/?p=linux/kernel/git/hpa/linux-2.6-allstable.git;a=blob;f=drivers/usb/gadget/f_mass_storage.c;h=f4911c09022e92c2c16cc1b294045a56a0aca231;hb=e40152ee1e1c7a63f4777791863215e3faa37a86#l2660>)(&msg_fsg_common);
        	return ret;
        }
```

The `msg_bind()` function does the following: initializes the Mass Storage Function, adds the previously defined configuration to the USB device, and (at the end) _puts_ the `msg_fsg_common` object. . If everything succeeds, it sets the `msg_registered` flag so it is recorded that the gadget has been registered and initialized. 

With all of the above, a composite device can be defined. For this purpose, the [`usb_composite_driver` structure](<http://git.kernel.org/?p=linux/kernel/git/hpa/linux-2.6-allstable.git;a=blob;f=include/linux/usb/composite.h;h=738ea1a691cb8a5dab64d631792871273fd45053;hb=e40152ee1e1c7a63f4777791863215e3faa37a86#l237>) is used. Besides specifying the name, it points to the device descriptors and the bind callback:

```
static [struct usb_composite_driver](<http://git.kernel.org/?p=linux/kernel/git/hpa/linux-2.6-allstable.git;a=blob;f=include/linux/usb/composite.h;h=738ea1a691cb8a5dab64d631792871273fd45053;hb=e40152ee1e1c7a63f4777791863215e3faa37a86#l237>) msg_device = {
        	.name		= "g_my_mass_storage",
        	.dev		= &msg_dev_desc,
        	.bind		= msg_bind,
        };
```

At this point, all that is left are the init and exit module functions:

```
static int __init msg_init(void)
        {
        	return usb_composite_register(&msg_device);
        }
    
        static void msg_exit(void)
        {
        	if (test_and_clear_bit(0, &msg_registered))
        		usb_composite_unregister(&msg_device);
        }
```

They use the [`usb_composite_register()`](<http://git.kernel.org/?p=linux/kernel/git/hpa/linux-2.6-allstable.git;a=blob;f=drivers/usb/gadget/composite.c;h=09289bb1e20fcf984b388b32f55a6c404cc1dcb3;hb=e40152ee1e1c7a63f4777791863215e3faa37a86#l1109>) and [`usb_composite_unregister()`](<http://git.kernel.org/?p=linux/kernel/git/hpa/linux-2.6-allstable.git;a=blob;f=drivers/usb/gadget/composite.c;h=09289bb1e20fcf984b388b32f55a6c404cc1dcb3;hb=e40152ee1e1c7a63f4777791863215e3faa37a86#l1138>) functions to register and unregister the device. The `msg_registered` variable is used to ensure the device is unregistered only once.

To sum things up:

  * A composite device (`msg_device`) is registered when in `msg_init()` when the module loads.
  * It has a device `bind` callback (`msg_bind()`) that initializes MSF and adds configuration to the gadget.
  * The configuration (`msg_config`) has its own `bind` callback (`msg_do_config()`), which binds MSF to the configuration.
  * The really hard work is done inside the MSF.

#### Mass Storage Function

With the big picture in mind, lets get into the finer details: the inner workings of the Mass Storage Function. There are a couple of things to watch out for when dealing with it.

First of all, because MSF can be bound to several configurations, it needs to share some data between the instances and at the same time store information specific for each configuration. The [`fsg_common` structure](<http://git.kernel.org/?p=linux/kernel/git/hpa/linux-2.6-allstable.git;a=blob;f=drivers/usb/gadget/f_mass_storage.c;h=f4911c09022e92c2c16cc1b294045a56a0aca231;hb=e40152ee1e1c7a63f4777791863215e3faa37a86#l318>) is used for shared data. An instance of this structure needs to be initialized prior to binding MSF.

Because the common object is used by several MSF instances, it has no single owner thus a reference counter is needed to decide when it can be destroyed. That's the reason for the [`fsg_common_put()`](<http://git.kernel.org/?p=linux/kernel/git/hpa/linux-2.6-allstable.git;a=blob;f=drivers/usb/gadget/f_mass_storage.c;h=f4911c09022e92c2c16cc1b294045a56a0aca231;hb=e40152ee1e1c7a63f4777791863215e3faa37a86#l2660>) call at the end of `msg_bind()` function.

Closely connected with the `fsg_common` structure is a [worker thread](<http://git.kernel.org/?p=linux/kernel/git/hpa/linux-2.6-allstable.git;a=blob;f=drivers/usb/gadget/f_mass_storage.c;h=f4911c09022e92c2c16cc1b294045a56a0aca231;hb=e40152ee1e1c7a63f4777791863215e3faa37a86#l2559>) which MSF uses to handle all the host's requests. When a `fsg_common` object is created, a thread is started as well. It terminates either when the `fsg_common` object is destroyed or when it is killed with an INT, TERM, or KILL signal. In the latter case, the `fsg_common` object may still exist even after worker's death. Whatever reason, when thread exits a `thread_exits` callback is invoked.

It is important to note that a signal may terminate the worker thread, but why would one want to do that? The reason is simple. As long as MSF is holding any open files, the filesystems which those files belong to cannot be unmounted. That is bad news for a shutdown script.

What Alan Stern came up with in FSG, is to close all backing files when the worker thread receives an INT, TERM, or KILL signal. Because MSF is to be used with various composite gadgets, rather than hardcoding that behavior a callback has been introduced.

The last thing to note is that MSF is customizable. The UMC specification allows for a single device to have several logical units (sometimes called LUNs, which is strictly speaking incorrect since LUN stands for Logical Unit Number). Each logical unit may be read-only or read-write, may emulate a CD-ROM or disk drive, and may be removable or not.

All of this configuration must be specified when the `fsg_common` structure is initialized. The [`fsg_config` structure](<http://git.kernel.org/?p=linux/kernel/git/hpa/linux-2.6-allstable.git;a=blob;f=drivers/usb/gadget/f_mass_storage.c;h=f4911c09022e92c2c16cc1b294045a56a0aca231;hb=e40152ee1e1c7a63f4777791863215e3faa37a86#l383>) is used for exactly that purpose. In most cases, a module author does not want to fill it themselves, but rather let a user of the module decide the settings.

To make it as easy as possible, an [`fsg_module_parameters` structure](<http://git.kernel.org/?p=linux/kernel/git/hpa/linux-2.6-allstable.git;a=blob;f=drivers/usb/gadget/f_mass_storage.c;h=f4911c09022e92c2c16cc1b294045a56a0aca231;hb=e40152ee1e1c7a63f4777791863215e3faa37a86#l3010>) and an [`FSG_MODULE_PARAMETERS()` macro](<http://git.kernel.org/?p=linux/kernel/git/hpa/linux-2.6-allstable.git;a=blob;f=drivers/usb/gadget/f_mass_storage.c;h=f4911c09022e92c2c16cc1b294045a56a0aca231;hb=e40152ee1e1c7a63f4777791863215e3faa37a86#l3033>) are provided by the MSF. The former stores user-supplied arguments, whereas the latter defines several module parameters.

Having an `fsg_module_parameters` object, one may use [fsg_config_from_params()](<http://git.kernel.org/?p=linux/kernel/git/hpa/linux-2.6-allstable.git;a=blob;f=drivers/usb/gadget/f_mass_storage.c;h=f4911c09022e92c2c16cc1b294045a56a0aca231;hb=e40152ee1e1c7a63f4777791863215e3faa37a86#l3048>) followed by [fsg_common_init()](<http://git.kernel.org/?p=linux/kernel/git/hpa/linux-2.6-allstable.git;a=blob;f=drivers/usb/gadget/f_mass_storage.c;h=f4911c09022e92c2c16cc1b294045a56a0aca231;hb=e40152ee1e1c7a63f4777791863215e3faa37a86#l2666>) to create an `fsg_common` object. Alternatively, [fsg_common_from_params()](<http://git.kernel.org/?p=linux/kernel/git/hpa/linux-2.6-allstable.git;a=blob;f=drivers/usb/gadget/f_mass_storage.c;h=f4911c09022e92c2c16cc1b294045a56a0aca231;hb=e40152ee1e1c7a63f4777791863215e3faa37a86#l3084>) can be used which merges the call to the other two functions.

Here is how it all works when put together:

```
static struct fsg_module_parameters msg_mod_data = { .stall = 1 };
        FSG_MODULE_PARAMETERS(/* no prefix */, msg_mod_data);
    
        static struct fsg_common msg_fsg_common;
    
        static int msg_thread_exits(struct fsg_common *common)
        {
        	msg_exit();
        	return 0;
        }
    
        static int msg_fsg_init(struct usb_composite_dev *cdev)
        {
        	struct fsg_config config;
        	struct fsg_common *retp;
    
        	fsg_config_from_params(&config, &msg_mod_data);
    
        	config.thread_exits = msg_thread_exits;
    
        	retp = fsg_common_init(&msg_fsg_common, cdev, &config);
        	return IS_ERR(retp) ? PTR_ERR(retp) : 0;
        }
```

The `msg_exit()` function has been chosen as MSF's `thread_exits` callback. Since MSF is nonoperational after the thread has exited, there is no need to keep the composite device registered, instead the gadget is unregistered.

At this point, it should become obvious why the `msg_registered` flag is being used. Since `usb_composite_unregister()` can be called from two different places, a mechanism to guarantee that it will be called only once is needed -- atomic bit operations are perfect for such tasks.

And that would be it. We are done. One can grab [the full source code](<http://lwn.net/Articles/395889/>) and start playing with it.

The beauty of the composite framework is that all the really hard stuff has been already written. One can write devices and experiment with different configurations without deep knowledge of the USB specification or the Linux gadget API. At the same time, it is a perfect introduction to some more serious USB programming.

#### Running

To use the gadget, one needs to provide a disk image that will act as a real USB device to the USB host. Using `dd` on the device is perfect for creating one:

```
# **dd if=/dev/zero of=disk.img bs=1M count=64**
```

With disk image in place, the module can be loaded:

```
# **insmod g_my_mass.ko file=$PWD/disk.img**
```

Connecting the device to the host should produce several messages in the host system log, among others:

```
usb 1-4.4: new high speed USB device using ehci_hcd and address 8
        usb 1-4.4: New USB device found, idVendor=0525, idProduct=a4a5
        usb-storage: device scan complete
        sd 6:0:0:0: [sdb] Attached SCSI removable disk
        sd 6:0:0:0: [sdb] 131072 512-byte logical blocks: (67.1 MB/64.0 MiB)
         sdb: unknown partition table
```

All that is left is creating a partition with a filesystem and starting using the pendrive:

```
# **fdisk /dev/sdb**
        ...
        # **dmesg -c**
        sd 6:0:0:0: [sdb] Assuming drive cache: write through
         sdb: sdb1
    
        # **mkfs.vfat /dev/sdb1**
        mkfs.vfat 3.0.9 (31 Jan 2010)
        # **mount /dev/sdb1 /mnt/**
        # **touch /mnt/foo**
        # **umount /mnt**
```

As has been shown, the gadget works like a charm.

#### Conclusion

The Linux USB composite framework provides a way to add USB devices in a fairly straightforward way. Before the composite framework came along, developers needed to implement all USB requests for each gadget they wanted to add to the system. The framework handles basic USB requests and separates each USB composite function, which allows gadget authors to think in terms of functions rather than low-level interfaces and communication handling. 

As one might guess, this article just scratches the surface of what the composite framework can do. The driver that was shown is a single-configuration, single-function gadget, so the advantages over non-composite gadgets is not readily apparent. A future article may look at drivers for more powerful gadgets using the composite framework.
