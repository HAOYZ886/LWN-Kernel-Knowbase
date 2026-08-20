---
title: Device tree overlays
url: https://lwn.net/Articles/616859/
date: "October 22, 2014"
category: Device tree
author: "By Jonathan Corbet October 22, 2014 LinuxCon Europe"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
October 22, 2014

* * *

[LinuxCon Europe](<https://lwn.net/Archives/ConferenceByYear/#2014-LinuxCon_Europe>)

Pantelis Antoniou started his LinuxCon Europe session on device tree overlays by noting that the device tree concept often draws complaints — frequently of the inflammatory variety. Those complaints did not prevent the room from filling up to capacity, though — it would have been standing room only except that the on-site German fire marshals took their job seriously and would not allow standing in the sessions. Device trees as currently implemented in the kernel, Pantelis said, are also not up to the task of describing current hardware. Work done by him and others should rectify that situation in the near future, though. 

He started with an overview of the device tree concept: a device tree is essentially a text file that describes the hardware to the kernel. Since many architectures do not have self-describing hardware, some sort of externally supplied description is needed for the kernel to understand the system it is running on; device trees are the solution of choice in the Linux world. But, like most technologies, device trees have their shortcomings. The device tree language is another thing that software and hardware developers have to learn; to make things worse, it is a cryptic language that presents a lot of complexity to beginners. The fact that the current device tree compiler performs no syntax checks does not help the situation; the first indication of an incorrect device tree file is typically a failure to boot. Being purely data-driven, device tree files cannot contain any imperative logic. And so on. 

But the worst problem, according to Pantelis, is that the static nature of device trees makes them incapable of describing contemporary hardware. It is not always possible to know what the hardware will look like prior to booting the system, but device trees are set in stone at boot time. For a self-contained system like a phone handset, the static nature of device trees is not a big problem. But consider hardware like the [BeagleBone](<https://lwn.net/Articles/576434/>), which can have any of a number of add-on "cape" boards that augment the hardware. Creating a device tree file for every combination of boards and capes is not a [![\[Pantelis Antoniou\]](https://static.lwn.net/images/conf/2014/lce-lpc/PantelisAntoniou-sm.jpg)](<https://lwn.net/Articles/616997/>) viable solution. Assembling a device tree in the bootloader is possible but difficult, and it falls apart when faced with multiple capes stacked onto a single system. It would be far better to be able to piece together, at boot time or afterward, separate device tree fragments representing the board and the cape(s), ending up with a description of the full system. 

This problem comes up in other settings as well. The Raspberry Pi supports "hats" for the addition of hardware. Hardware built around a field-programmable gate array (FPGA) can vary wildly in nature depending on the firmware loaded into the array; such hardware cannot possibly be supported by a static device tree. Hardware, Pantelis said, is software now. But Linux makes dealing with the new hardware unnecessarily complex, driving hardware hackers to simpler (but far less capable) systems like the Arduino. 

The first attempt to solve the problem (in the BeagleBone context) was a subsystem called "[capebus](<https://lwn.net/Articles/522087/>)." But this proposal did not last long once reviewers got a look at it. It was modeling the cape problem around a bus abstraction, but capes do not sit on a bus. So another approach was indicated; in the end, it was decided that dynamically altering the system's device tree to reflect the actual hardware was the right solution to the problem. 

A piece of the solution has been in the kernel for some time; it is controlled by the `CONFIG_OF_DYNAMIC` configuration option. It allows run-time modification of the device tree, but it is only used by the PowerPC architecture. Editing of the tree is destructive, meaning that changes cannot be reverted later; that is problematic for hardware that can be hot-removed from a running system. Changes are also not performed in an atomic manner. There is no connection to the device model code, so users must make any system topology changes independently. In short, it is a piece of the puzzle, but it is far from a complete solution. 

The first step toward that complete solution, Pantelis said, is to rework the dynamic device tree code. Some control files have been moved from `/proc` to `/sys`. Nodes in the device tree are now proper kobjects, so they have lifecycle management built into them. Some changes to better define the semantics of the reconfiguration notifiers have been made. This work was all merged into the 3.17 kernel. 

The second step is "the meat of the problem," according to Pantelis. It is often necessary for one part of a device tree to refer to another part; a camera sensor description, for example, may include a pointer to the I2C bus that carries the sensor's control channel. These references are called "phandles"; they are symbolic within the human-readable device tree, but converted to simple integer values by the device tree compiler. Pantelis had to extend the compiler to keep track of all phandles used; when requested (with the arguably strange "`-@`" command-line option), the compiler will store a sort of symbol table in the root of the compiled device tree with the list of all phandles in the tree. 

This mechanism allows the loading of a device tree fragment into the system's current device tree. The new fragment will contain references to phandles in the main tree; the new in-kernel resolver code will fix up those references to match the real phandles in that tree. The resolver will also relocate all of the phandles in the new fragment to ensure that they are unique within the device tree as a whole and adjust any internal references accordingly. 

Step three is to add the concept of device tree changesets to the kernel. A call to `of_changeset_init()` starts the addition of a changeset; then new device tree pieces can be added with `of_changeset_attach_node()`. Once the pieces are in place, it's a matter of locking the device tree and calling `of_changeset_apply()`. If the change needs to be reverted in the future (perhaps the hardware in question has been hot-unplugged from the system), `of_changeset_revert()` will put things back as they were before. 

With this infrastructure in place, device tree overlays can be supported. An overlay can add nodes to the tree, but it can also make changes to properties in the existing tree. In the simplest case, an overlay might just change a device node's status from "disabled" to "enabled." This feature is useful for hardware hackers, Pantelis said; hardware presence can be turned on or off easily with no need to reboot the system or to dig into C code. 

The resolver code was merged into the 3.18 kernel; full overlay support should come soon. In the future, there is an overlay-based FPGA manager in the works, along with a BeagleBone cape manager. There is also interest in using this feature to support multiple versions of a given board from a single device tree. The end result of all this work is that device trees have become more dynamic — and more capable — than they were when the kernel first started using them. 

[Your editor would like to thank the Linux Foundation for supporting his travel to LinuxCon Europe.]
