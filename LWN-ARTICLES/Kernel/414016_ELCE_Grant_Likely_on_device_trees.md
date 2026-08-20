---
title: "ELCE: Grant Likely on device trees"
url: https://lwn.net/Articles/414016/
date: "November 10, 2010"
category: Device tree
author: "By Jake Edge November 10, 2010"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
November 10, 2010

Device trees are a fairly hot topic in the embedded Linux world as a means to more easily support multiple system-on-chip (SoC) devices with a single kernel image. Much of the work implementing device trees for the PowerPC architecture, as well as making that code more generic so that others could use it, has been done by Grant Likely. He spoke at the recent [Embedded Linux Conference Europe](<http://www.embeddedlinuxconference.com/elc_europe10/index.html>) (ELCE) to explain what device trees are, what they can do, and to update the attendees on efforts to allow the ARM architecture use them. 

[ ![\[Grant Likely\]](https://static.lwn.net/images/2010/elce-likely-sm.jpg) ](<https://lwn.net/Articles/414439/>)

All of the work that is going into adding device tree support for various architectures is not being done for an immediate benefit to users, Likely said. It is, instead, being done to make it easier to manage embedded Linux distributions, while simplifying the boot process. It will also make it easier to port devices (i.e. components and "IP blocks") to different SoCs. But it is ""not going to make your Android phone faster"". 

A device tree is just a data structure that came from [OpenFirmware](<http://en.wikipedia.org/wiki/Open_Firmware>). It represents the devices that are part of particular system, such that it can be passed to the kernel at boot time, and the kernel can initialize and use those devices. For architectures that don't use device trees, C code must be written to add all of the different devices that are present in the hardware. Unlike desktop and server systems, many embedded SoCs do not provide a way to enumerate their devices at boot time. That means developers have to hardcode the devices, their addresses, interrupts, and so on, into the kernel. 

The requirement to put all of the device definitions into C code is hard to manage, Likely said. Each different SoC variant has to have its own, slightly tweaked kernel version. In addition, the full configuration of the device is scattered over multiple C files, rather than kept in a single place. Device trees can change all of that. 

A device tree consists of a set of nodes with properties, which are simple key-value pairs. The nodes are organized into a tree structure, unsurprisingly, and the property values can store arbitrary data types. In addition, there are some standard usage conventions for properties so that they can be reused in various ways. The most important of these is the `compatible` property that uniquely defines devices, but there are also conventions for specifying address ranges, IRQs, GPIOs, and so forth. 

Likely used a [simplified example from devicetree.org](<http://devicetree.org/Device_Tree_Usage#Basic_Concepts>) to show what these trees look like. They are defined with an essentially C-like syntax: 

```
/ {
    	compatible = "acme,coyotes-revenge";
    
    	cpus {
    	    cpu@0 {
    		compatible = "arm,cortex-a9";
    	    };
    	    cpu@1 {
    		compatible = "arm,cortex-a9";
    	    };
    	};
    
    	serial@101F0000 {
    	    compatible = "arm,pl011";
    	};
            ...
    	external-bus {
    	    ethernet@0,0 {
    		compatible = "smc,smc91c111";
    	    };
    
    	    i2c@1,0 {
    		compatible = "acme,a1234-i2c-bus";
    		rtc@58 {
    		    compatible = "maxim,ds1338";
    		};
    	    };
                ...
```

The `compatible` tags allow companies to define their own namespace ("acme", "arm", "smc", and "maxim" in the example) that they can manage however they like. The kernel already knows how to attach an ethernet device to a local bus or a temperature sensor to an i2c bus, so why redo it in C for every different SoC, he asked. By parsing the device tree (or the binary "flattened" device tree), the kernel can set up the device bindings that it finds in the tree. 

One of the questions that he often gets asked is: ""why bother changing what we already have?"" That is a ""hard question to answer"" in some ways, because for a lot of situations, what we have in the kernel currently does work. But in order to support large numbers of SoCs with a single kernel (or perhaps a small set of kernels), something like device tree is required. Both Google (for Android) and Canonical (for Linaro) are very interested in seeing device tree support for ARM. 

Beyond that, ""going data-driven to describe our platforms is the right thing to do"". There is proof that it works in the x86 world as ""that's how it's been done for a long time"". PowerPC converted to device trees five years ago or so and it works well. There may be architectures that won't need to support multiple devices with a single kernel, and device trees may not be the right choice for those, but for most of the architectures that Linux supports, Likely clearly thinks that device trees are the right solution. 

He next looked at what device trees aren't. They don't replace board-specific code, and developers will ""still have to write drivers for weird stuff"". Instead, device trees simplify the common case. Device tree is also not a boot architecture, it's ""just a data structure"". Ideally, the firmware will pass a device tree to the kernel at boot time, but it doesn't have to be done that way. The device tree could be included into the kernel image. There are plenty of devices with firmware that doesn't know about device trees, Likely said, and they won't have to. 

There is currently a push to get ARM devices into servers, as they can provide lots of cores at low power usage. In order to facilitate that, there needs to be one CD that can boot any of those servers, like it is in the x86 world. Device trees are what will be used to make that happen, Likely said. 

Firmware that does support device trees will obtain a `.dtb` (i.e. flattened device tree binary) file from somewhere in memory, and either pass it verbatim to the kernel or modify it before passing. Another option would be for the firmware to create the `.dtb` on-the-fly, which is what OpenFirmware does, but that is a ""dangerous"" option. It is much easier to change the kernel than the firmware, so any bugs in the firmware's `.dtb` creation code will inevitably be worked around in the kernel. In any case, the kernel doesn't care how the `.dtb` is created. 

For ARM, the plan is to pass a device tree, rather than the existing, rather inflexible ARM device configuration known as ATAGs. The kernel will set up the memory for the processor and unflatten the `.dtb` into memory. It will unpack it into a ""live tree"" that can then be directly dereferenced and used by the kernel to register devices. 

The Linux device model is also tree-based, and there is some congruence between device tree and the device model, but there is not a direct 1-to-1 mapping between them. That was done ""quite deliberately"" as the design goal was ""not to describe what Linux wants"", instead it was meant to describe the hardware. Over time, the Linux device model will change, so hardcoding Linux-specific values into the device tree has been avoided. The device tree is meant to be used as support data, and the devices it describes get registered using the Linux device model. 

Device drivers will match `compatible` property values with device nodes in a device tree. It is the driver that will determine how to configure the device based on its description in a device tree. None of that configuration code lives in the device tree handling, it is part of the drivers which can then be built as loadable kernel modules. 

Over the last year, Likely has spent a lot of time making the device tree support be generic. Previously, there were three separate copies of much of the support code (for Microblaze, SPARC, and PowerPC). He has removed any endian dependencies so that any architecture can use device trees. Most of that work is now done and in the mainline. There is some minimal board support that has not yet been mainlined. The MIPS architecture has added device tree support as of 2.6.37-rc1 and x86 was close to getting it for 2.6.37, but some last minute changes caused the x86 device tree support to be held back until 2.6.38. 

The ARM architecture still doesn't have device tree support and ARM maintainer Russell King is ""nervous about merging an unmaintainable mess"". King is taking a wait-and-see approach until a real ARM board has device tree support. Likely agreed with that approach and ELCE provided an opportunity for him and King to sit down and discuss the issue. In the next six months or so (2.6.39 or 2.6.40), Likely expects that the board support will be completed and he seems confident that ARM device tree support in the mainline won't be far behind. 

There are other tasks to complete in addition to the board support, of course, with documentation being high on that list. There is a need for documentation on how to use device trees, and on the property conventions that are being used. The [devicetree.org wiki](<http://devicetree.org/Main_Page>) is a gathering point for much of that work. 

There were several audience questions that Likely addressed, including the suitability of device tree for Video4Linux (very suitable and the `compatible` property gives each device manufacturer its own namespace), the performance impact (no complaints, though he hasn't profiled it -- device trees are typically 4-8K in size, which should minimize their impact), and licensing or patent issues (none known so far, the code is under a BSD license so it can be used by proprietary vendors -- IBM's lawyers don't seem concerned). Overall, both Likely and the audience seemed very optimistic about the future for device trees in general and specifically for their future application in the ARM architecture.
