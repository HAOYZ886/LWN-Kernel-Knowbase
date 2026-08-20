---
title: "Supporting multi-platform ARM kernels"
url: https://lwn.net/Articles/496400/
date: "May 9, 2012"
category: "Architectures-Arm; Device tree"
author: "By Jonathan Corbet May 9, 2012"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
May 9, 2012

The diversity of the ARM architecture is one of its great strengths: manufacturers have been able to create a wide range of interesting system-on-chip devices around the common ARM processor core. But this diversity, combined with a general lack of hardware discoverability, makes ARM systems hard to support in the kernel. As things stand now, a special kernel must be built for any specific ARM system. With most other architectures, it is possible to support most or all systems with a single binary kernel (or maybe two for 32-bit and 64-bit configurations). In the ARM realm, there is no single binary kernel that can run everywhere. Work is being done to improve that situation, but some interesting decisions will have to be made on the way. 

On an x86 system, the kernel is, for the most part, able to boot and ask the hardware to describe itself; kernels can thus configure themselves for the specific system on which they are run. In the ARM world, the hardware usually has no such capability, so the kernel must be told which devices are present and where they can be found. Traditionally, this configuration has been done in "board files," which have a number of tasks: 

  * Define any system-specific functions and setup code. 

  * Create a description of the available peripherals, usually through the definition of a number of [platform devices](<https://lwn.net/Articles/448499/>). 

  * Create a special machine description structure that includes a magic number defined for that particular system. That number must be passed to the kernel by the bootloader; the kernel uses it to find the machine description for the specific system being booted. 

There are currently hundreds of board files in the ARM architecture subtree, and some unknown number of them shipped in devices but never contributed upstream. Within a given platform type (a specific system-on-chip line from a vendor), it is often possible to build multiple board files into a single kernel, with the actual machine type being specified at boot time. But combining board files across platform types is not generally possible. 

One of the main goals of the current flurry of work in the ARM subtree is to make multi-platform kernels possible. An important step in that direction is the elimination of board files as much as possible; they are being replaced with [device trees](<https://lwn.net/Articles/414016/>). In the end, a board file is largely a static data structure describing the topology of the system; that data structure can just as easily be put into a text file passed into the kernel by the boot loader. By moving the hardware configuration information out of the kernel itself, the ARM developers make the kernel more easily applicable to a wider variety of hardware. There are a lot of other things to be done before we have true multi-platform support—work toward properly abstracting interrupts and clocks continues, for example—but device tree support is an important piece of the puzzle. 

Arnd Bergmann recently [asked](<https://lwn.net/Articles/496419/>) a question to the kernel development community: does it make sense to support legacy board files in multi-platform kernels? Or would it be better to limit support to systems that use device trees for hardware enumeration? Arnd was pretty clear on what his own position was: 

My feeling is that we should just mandate DT booting for multiplatform kernels, because it significantly reduces the combinatorial space at compile time, avoids a lot of legacy board files that we cannot test anyway, reduces the total kernel size and gives an incentive for people to move forward to DT with their existing boards. 

There was a surprising amount of opposition to this idea. Some developers seemed to interpret Arnd's message as a call to drop support for systems that lack device tree support, but that is not the point at all. Current single-platform builds will continue to work as they always have; nobody is trying to take that away. The point, instead, is to make life easier for developers trying to make multi-platform builds work; multi-platform ARM kernels have never worked in the past, so excluding some systems will not deprive their users of anything they already had. 

Some others saw it as an arbitrary restriction without any real technical basis. There is nothing standing in the way of including non-device-tree systems in a multi-platform kernel except the extra complexity and bloat that they bring. But complexity and bloat are technical problems, especially when the problem being solved is difficult enough as it is. It was also pointed out that there are some older platforms that have not seen any real maintenance in recent times, but which are still useful for users. 

In the end, it will come down to what the users of multi-platform ARM kernels want. It was not immediately clear to everybody that there are users for such kernels: ARM kernels are usually targeted to specific devices, so adding support for other systems gives no benefit at all. Thus, embedded systems manufacturers are likely to be uninterested in multi-platform support. Distributors are another story, though; they would like to support a wide range of systems without having to build large numbers of kernels. As Debian developer Wookey [put it](<https://lwn.net/Articles/496429/>): 

We are keen on multiplatform kernels because building a great pile of different ones is a massive pain (and not just for arm because it holds up security updates), and if we could still cover all that lot with one kernel, or indeed any number less than 7 that would be great. 

In response, Arnd [amended his proposal](<https://lwn.net/Articles/496425/>) to allow board files for subarchitectures that don't look likely to support device trees anytime soon. At that point, the discussion wound down without any sort of formal conclusion. The topic will likely be discussed at the upcoming Linaro Connect event and, probably, afterward as well. There are a number of other issues to be dealt with before multi-platform ARM kernels are a reality; that gives some time for this particular decision to be considered with all the relevant needs in mind.
