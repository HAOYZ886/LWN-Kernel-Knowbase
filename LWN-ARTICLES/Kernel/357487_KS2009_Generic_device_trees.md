---
title: "KS2009: Generic device trees"
url: https://lwn.net/Articles/357487/
date: "October 19, 2009"
category: Device tree
author: "By Jonathan Corbet October 19, 2009 LWN's 2009 Kernel Summit coverage"
---

By **Jonathan Corbet**  
October 19, 2009

* * *

LWN's [2009 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2009/>)

Device trees have been the subject of some acrimonious mailing list discussions in the past, but developers don't always have a good sense for what the term means. In an effort to clarify the situation, Grant Likely and Ben Herrenschmidt ran a session on how this abstraction works. 

In essence, a device tree is a data structure for describing the hardware on a system. It has its origins in OpenFirmware, and it retains the format which was laid out there. The tree structure is simple, containing nodes (devices) which have an arbitrary number of properties. A typical device tree entry looks something like the following (taken from `arch/powerpc/boot/dts/ep88xc.dts` in the kernel source): 

```
ethernet@e00 {
    	device_type = "network";
    	compatible = "fsl,mpc885-fec-enet",
    	             "fsl,pq1-fec-enet";
    	reg = <0xe00 0x188>;
    	local-mac-address = [ 00 00 00 00 00 00 ];
    	interrupts = <3 1>;
    	interrupt-parent = <&PIC>;
    	phy-handle = <&PHY0>;
    	linux,network-index = <0>;
        };
```

Most of the fields should be relatively self-explanatory; this node describes an Ethernet adapter, where its hardware resources are to be found, how it is connected into the system, and so on. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Traditionally, embedded Linux kernels run on special-purpose systems with hardware which cannot be probed for automatically. The configuration of the system usually comes down to some board-specific platform code which knows how the hardware has been put together. Device trees are an attempt to move that information out of the code and into a separate data structure. When done right, device trees can make it possible for a single kernel to support a wide range of boards - something which is hard to do when the system configuration is hardwired into the code. It can even be possible to support systems which do not exist when the kernel is built. 

Device tree proponents assert that the "board port mindset" is broken. It should not be necessary to modify the kernel for each board which comes along. These modifications, beyond being ugly and painful, lead to a lot of ifdefs and platform-specific code paths in the kernel, all of which is hard to maintain. Device trees also make it possible to get the hardware configuration from a running kernel, even if the vendor is otherwise not forthcoming with that information. 

The device tree abstraction is used by the PowerPC and MicroBlaze architectures now. There is a lot of interest in using it in the ARM architecture code, but the ARM maintainer is a bit skeptical of the idea. Still, it seems like it might be possible to convince him by carefully porting a subarchitecture or two to device trees first. There were some supportive words from the audience; Greg Kroah-Hartman liked how device trees made it possible to remove static `device` structures from the kernel, while Thomas Gleixner observed that his employees are much happier about doing ports to boards where device trees are used than to other systems. So the use of device trees in the kernel may expand, but, to a great extent, that depends on architecture maintainers who were not present at the summit.
