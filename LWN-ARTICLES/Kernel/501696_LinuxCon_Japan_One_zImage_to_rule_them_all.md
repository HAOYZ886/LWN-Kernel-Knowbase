---
title: "LinuxCon Japan: One zImage to rule them all"
url: https://lwn.net/Articles/501696/
date: "June 13, 2012"
category: "Architectures-Arm"
author: "By Jake Edge June 13, 2012"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
June 13, 2012

An elusive, but highly sought after goal for Linux on ARM is to have a "single" kernel image that can boot on all ARM devices. The full goal is likely not attainable, but reducing the number of different ARM kernels is, and progress is being made. Linaro kernel tech lead Deepak Saxena reported on the motivations behind the consolidation effort as well as progress toward the goal at LinuxCon Japan 2012. 

#### The holy grail

Saxena started with a problem statement. He noted that those who have worked with ARM for a while may not really see the problem, but it _is_ a problem that every ARM processor needs its own kernel. For example, a laptop vendor that makes a few small hardware changes will almost certainly need a separate kernel for each revision. The "holy grail" is a single kernel binary that will boot on any ARM device, he said. 

[ ![\[Deepak Saxena\]](https://static.lwn.net/images/2012/lcj-saxena-sm.jpg) ](<https://lwn.net/Articles/501737/>)

Requiring separate kernels "creates a lot of extra work" for developers. If they are building a driver for multiple platforms--or even three revisions of the same platform--they will probably need to build and test a kernel for each. Sometimes there is a register that can be read to determine the hardware revision, thus reducing the number of different kernels, but that often is not the case. 

For distributions, supporting ARM is difficult right now. Debian, for example, has different kernels for the BeagleBoard, PandaBoard, and others. In the consumer electronics and mobile space, the problem is similar, and those companies want to reduce the amount of testing that needs to be done. Right now, we think of cell phones more than anything else, but the Saxena pointed to the video projector and camera as devices that are also likely ARM-powered. Beyond that, there will soon be ARM servers. 

The model in the server/enterprise world is very different than mobile. Typically, a mobile device comes with a full software stack, and updates come the same way, but that is not the case for servers. You may or may not get the distribution that you want on servers and, in fact, may get Windows on those systems. That means that distributions need to have installation media that can work on a wide variety of server platforms. 

"The distros have spoken", Saxena said, and they need one kernel image that can be built, booted, and tested on all of the different platforms. There are thousands of different x86-based laptops today, and you don't need a different kernel for each. The Linux ARM community wants to get to that same model. Beyond that, cloud and hyperscale computing also need to be able to deploy on lots of different platforms without requiring separate kernels. 

"How did we get here?", he asked; you can boot a single Ubuntu or Fedora install disk on all x86 systems, but that is not true for ARM. Part of the problem is the diversity in the ARM world. There is also a lot of fragmentation in how code has been written to support Linux on ARM. There are multiple implementations for the same IP block, for example. Functionality has been duplicated at least partly because of a closed-door policy by the various ARM vendors. 

In addition, Linux ARM maintainer Russell King got overloaded, which led ARM platform and system-on-chip (SoC) developers to start pushing their code to Linus Torvalds directly. Saxena said that he may have been the first to do that, for ixp4xx, but now he sees the problems that caused and apologized for doing so. It led to an inability for anyone to track the "big picture" in Linux ARM support, he said. 

#### Fixes needed

It is now a multi-faceted problem that requires several different avenues of attack to clean up. Saxena identified four areas that are being pursued. Cleaning up and consolidating the header files within the various ARM machine and platform trees is one, while consolidating ARM drivers is another. In addition, device tree will provide a way to specify the differences between ARM SoCs at runtime. Finally, doing active maintenance of the ARM tree, keeping in mind the big picture, will also help. No one of those fixes the problem, but all of them together get us closer to the holy grail, he said. 

To start with, there are various header file collisions in the ARM tree. There are a bunch of `arm/arch/mach-*` directories, one for each of the machine types. Each of those has an `include/mach` directory that maps to the top-level `include/mach` directory at build time. In order to build for multiple machine types, those header files need to be consolidated in one place so that the remapping doesn't need to happen. 

In the 3.0 kernel tree, which was around the time the consolidation effort started, there were 64 different machine types in the tree. Some of those machine types are similar and could be consolidated. For the others, there are lots of overlapping symbols that need to be dealt with. The goal is to get rid of as many of those as possible either by making them generic for all ARMs or by moving platform-specific symbols to non-generic header files. 

There were also 577 occurrences of `#include <mach/*>` in the `drivers` directory. Unfortunately, ARM has a lot of driver-specific definitions in the architecture headers, which means that drivers depend on `arch/arm` header files. Basically, it is prevalent for ARM drivers to require definitions from both the driver directories and the architecture headers, which makes it difficult to build multi-platform kernels. 

Linaro and the ARM community started working on these problems last year. They met in August and thought they could have a solution in relatively short order, but that proved not to be the case. Some changes require coordination between multiple maintainers and others are awaiting agreement between maintainers on how to proceed. The problem "may seem trivial at first", Saxena said, but it actually fairly complicated. The hope is to have a single zImage for multiple systems by the end of 2012. 

Beyond the header file issue, there is a need to cleanup and consolidate the drivers in the tree. The problem is not directly related to creating a single kernel, but fixing it will help to get there, he said. There are lots of implementations of drivers for the same hardware in the tree, sometimes with a different API. That can cause problems with overlapping symbols and code bloat. 

The clock management code is the "epitome of code duplication and fragmentation" in the ARM kernels, Saxena said. The `clk.h` file, which declares a `struct clk`, was introduced in 2.6.16 back in 2006\. Since then, 27 different `struct clk` implementations have been added to the tree, each with its own API and semantics. Over the last two years or more, work has been done to create a common definition and API, though the job is not done yet, as there is ongoing discussion on certain parts. 

Pinmux is another example of duplication. It is a subsystem to manage the pins on SoCs and there were multiple implementations of that functionality. The problem is not as bad as `struct clk`, he said, but there was still a need to consolidate. After six months of work, a [common pinmux implementation](<https://lwn.net/Articles/468759/>) is now upstream, though there are still discussions about certain parts of the implementation and API. 

Another piece of the solution is device tree. Before device tree, very small, simple changes to the hardware would require a kernel rebuild because the information about IRQ lines, memory maps, and so forth were statically defined. Device tree makes it so that much of this information can be probed at boot time. 

Using device tree means creating a source file using a "simple markup language" that defines the hardware. It can specify where the registers are or what the interrupts are for a particular device. That means that the same kernel can be used on a new SoC once a device tree file has been created for that SoC. Since the kernel will not have to change, it makes hardware revisions and testing multiple devices that much easier. 

#### Status and plans

Currently, a single kernel can be built for the four Linaro member platforms (imx, omap2, ux500 and vexpress), though only the Versatile Express board boots at this point. The goal is to have all four booting by the end of the year. Linaro is focused on its member platforms, but would like to get other platforms supported as well. That is even more reason for SoC vendors to get their code upstream, Saxena said, as it will be more difficult to participate in the multi-platform kernel effort with code that is out of the mainline. The ARM SoC tree has been used as the base, with Arnd Bergmann maintaining a branch for the multi-platform work. 

There are, of course, things still left to do. USB drivers need to be consolidated as there are some problems building multiple host controllers into one kernel at this point. Finishing the device tree conversion is another piece of the puzzle; the infrastructure is there, but there are lots of drivers and board files to convert. At the moment there is something of a hack around the "driver `#include` madness", which needs to be cleaned up for real. 

While the holy grail has not been reached, things will be better than they are today, Saxena said. Due to micro-architecture and page table differences, four kernels are envisioned: ARM v6/7 with large physical address extensions (lpae), ARM v6/7 non-lpae, ARM v4/5, and, eventually, ARM v8. It still means multiple kernel binaries, but that could be treated in the same way that the distributions handle various CPU extensions in the x86 world. The idea of "one zImage to rule them all" turns out to not be practical, but we will end up with far fewer ARM kernel binaries in the near future. 

[ The author would like to thank the Linux Foundation for assisting with travel to Yokohama for LinuxCon Japan 2012. ]
