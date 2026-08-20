---
title: Software and hardware obsolescence in the kernel
url: https://lwn.net/Articles/829733/
date: "August 28, 2020"
category: Architectures
author: "By Jonathan Corbet August 28, 2020 LPC"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
August 28, 2020

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2020-Linux_Plumbers_Conference>)

Adding code to the kernel to support new hardware is relatively easy. Removing code that is no longer useful can be harder, mostly because it can be difficult to know when something is truly no longer needed. Arnd Bergmann, who [removed support for eight architectures](<https://lwn.net/Articles/748074/>) from the kernel in 2018, knows well just how hard this can be. At the [2020 Linux Plumbers Conference](<https://linuxplumbersconf.org/>), he led two sessions dedicated to the topic of obsolete software and hardware. With a bit of effort, he said, it should be possible to have a better idea of when something can be removed. 

#### The software side

Obsolete hardware, he said, can be defined as devices that are no longer being made, usually because they have been superseded by newer, cheaper, and better products. Obsolete hardware can still be useful, and often remains in use for a long time, but it's hard to know whether any specific device is still used. Obsolete code is a bit different; the hardware it enables might still be in production, but all of its users are running older kernels and are never going to upgrade. In such cases, the code can be removed, since nobody benefits from its ongoing maintenance. 

Bergmann's proposal is to create a way of documenting which code in the kernel is there solely for the support of obsolete hardware; in particular, it would note the kernel configuration symbols associated with that hardware. For each symbol, the document would describe why it is still in use and for how long that situation is expected to continue. The consequences of removing this support (effects on other drivers that depend on it, for example) would be noted, as would the benefits that would come from removing it. 

There are various groups that would be impacted by this change. The kernel retains support for a number of hobbyist platforms, for example; these include processor architectures with no commercial value but an ongoing [![\[Arnd Bergmann\]](https://static.lwn.net/images/conf/2020/lpc/ArndBergmann-sm.png)](<https://lwn.net/Articles/829738/>) hobbyist following. The kernel still supports a number of Sun 3 workstation models; he has no idea whether anybody is actually running such systems or not. Kernel developers generally like to keep hobbyist platforms alive as long as somebody is willing to maintain them. 

Then there are platforms with few users, but those users may really need them. These include various types of embedded systems, industrial controllers, military systems, and more. There are also systems that are clearly headed toward obsolescence in the future. These include 32-bit architectures which, while still heavily used now, will eventually go away. Systems with big-endian byte order have declined 90% in the last ten years, and may eventually vanish entirely. 

So where should this sort of information be documented? He proposed a few options, including a new file in the documentation directory, in the `Kconfig` files that define the relevant configuration symbols, somewhere on [wiki.kernel.org](<https://wiki.kernel.org/>), or somewhere else entirely. Responding to a poll in the conference system, just over half of the attendees indicated a preference for a file in the kernel tree. 

At this point your editor had to jump in and ask how this idea compares to the kernel's `feature-removal-schedule.txt` file. This file was [added in 2005](<https://lwn.net/Articles/118844/>) as a way of warning users about features that would go away soon; this file itself was [removed in 2012](<https://git.kernel.org/linus/9c0ece069b32>) after Linus Torvalds got fed up with it. Why should the fate of this new file be different? Bergmann responded that this file would not be a schedule for removal of support; instead, it would be a way of documenting that support needs to be kept for at least a certain period of time. Users of the affected hardware could easily update the file at any time to assure the community that they still exist. As documentation for the reasons to keep support in the kernel, it would be more useful. 

Florian Weimer asked what the effect would be on user space if this proposal were adopted; the answer was "none". User-space interfaces are in a different category, Bergmann said, with a much higher bar to be overcome before they can be removed. This file would cover hardware-specific code. Mike Rapoport added that it would be a way to know when users would be unaffected, once it becomes clear that nobody is running upstream kernels on the hardware in question. 

Catalin Marinas suggested creating a `CONFIG_OBSOLETE` marker for code that supports obsolete hardware, but Will Deacon was nervous about that idea. He recently did some work on the page-table code for 32-bit SPARC machines; he got no comments on those changes, but he did get reports when various continuous-integration systems tested them. A `CONFIG_OBSOLETE` marker might be taken as a sign by the maintainers of such systems that the code no longer needs to be tested, reducing the test coverage significantly. 

Bergmann added that 32-bit SPARC is an interesting case. People have been finding serious bugs in that code, he said, and System V interprocess communication isn't working at all. There is a lot of testing of 32-bit SPARC user space, but the tests all run on 64-bit kernels, where these problems do not exist. He is confident that this code has few — if any — remaining users. 

Len Brown returned to the question of the old `feature-removal-schedule.txt` file, asking what had been learned from that experience. Bergmann replied that his proposed documentation is intended to help plan removal. It is, he said, a lot of work to try to figure out if a particular feature is being used by anybody; documenting users in this way would reduce that work considerably. Laurent Pinchart added that this information could also be useful for developers who would like to find users of a given piece of hardware to test a proposed change. 

As the session came to a close, James Bottomley noted that this kind of problem arises often in the SCSI subsystem, which tends to "keep drivers forever". Eventually, though, internal API changes force discussions on the removal of specific drivers, but that is always a hard thing to do. It is easy to say that a removed driver can always be resurrected from the Git history if it turns out to be needed, but that doesn't work out well in practice. 

Bergmann ended things by noting that the maintainer of a given driver is usually the person who knows that nobody is using a given device. But once that happens, the maintainer often goes away as well, taking that knowledge with them. At that point, it's nobody's job to remove the code in question, and it can persist for years. 

#### System-on-chip obsolescence

Bergmann returned to this topic in another session dedicated to the life cycle of system-on-chip (SoC) products. Having spent a lot of time working on various aspects of architecture support in the kernel, he has learned a few things about how support for SoCs evolves and what that might mean for the architectures currently supported by the kernel. 

There are, he said, five levels of support for any given SoC in the kernel: 

  1. Full upstream support, with all code in mainline, all features working, and new kernel features fully supported. 
  2. Minimal upstream support, but fixes and updates still make it into the stable kernel releases. 
  3. Updates in mainline are sporadic at best; perhaps fixes go into the long-term-support kernels. 
  4. No more upstream support; users are getting any updates directly from the vendor. 
  5. The system runs, but there are no updates or ongoing support in the mainline kernel. There might still be code in the kernel, but it is not used by anybody. 

The most important phase for SoC support is the bringup stage, when things are first made to work; if at all possible, that support should be brought all the way to the "full support" level. The level of support normally only goes down from there. People stop applying updates and, eventually, those updates stop appearing at all. 

Problems at bringup tend to happen in fairly predictable areas, with GPU drivers being at the top of the list. That said, the situation has gotten a lot better in recent times, with increasing numbers of GPUs having upstream support. Android patches can be another sticking point; that, too, is improving over time. Short time to market and short product lifespan can both be impediments to full support as well. 

Bergmann put up a diagram displaying the "CPU architecture world map" as of 2010; it can be seen on page 6 of [his slides [PDF]](<https://linuxplumbersconf.org/event/7/contributions/655/attachments/638/1161/LPC20_SoC_support_in_the_kernel_1.pdf>): 

> ![\[CPU architecture
world map\]](https://static.lwn.net/images/conf/2020/lpc/arnd-cpumap.png)

This map plots architectures used in settings from microcontrollers through to data-center applications on one dimension, and their affinity to big-endian or little-endian operation on the other. These architectures were spread across the map, with IBM Z occupying the big-endian, data-center corner, and numerous architectures like Blackfin and unicore32 in the little-endian, microcontroller corner. 

There were a lot of architectures available at that time, he said, and the future looked great for many of them. The Arm architecture was "a tiny thing" only used on phones, not particularly significant at the time. But phones turned out to be the key to success for Arm; as it increased its performance it was able to eliminate most of the others. 

The SoC part of the market, in particular, is represented by the middle part of the map: systems larger than microcontrollers, but smaller than data-center processors. There are three generations of these that are important to the kernel. The first, typified by the Armv5 architecture, came out around 2000 and is still going strong; these are uniprocessor systems with memory sizes measured in megabytes. The Armv7-A generation launched in 2007 with up to four cores on an SoC and memory sizes up to 2GB; this generation is completely dominated by Arm processors. Finally, the Armv8-A (and x86-64) generation, beginning in 2014, supports memory sizes above 2GB and 64-bit processors. 

He discussed memory technologies for a while, noting that DDR3 memory tends to be the most cost-effective option for sizes up to 2-4GB, but it is not competitive above that. That's significant because middle-generation processors cannot handle DDR4 memory. 

The only reason to go with first-generation processors, he said, is if extremely low cost is the driving factor. For most other applications, 64-bit systems are taking over; they are replacing 32-bit SoCs from a number of vendors. The middle, Armv7-A generation is slowly being squeezed out. 

#### Kernel support implications

So what are the implications for kernel support? He started with a plot showing how many machines are currently supported by the kernel; most of those, at this point, are described by devicetree files. There are a few hundred remaining that require board files (compiled machine descriptions written as C code). He suggested that the time may be coming when all board-file machines could be removed; if those machines were still in use, he said, somebody would have converted them to devicetree. 

By 2017, it became clear that many architectures were approaching the end of their lives; that led to the removal of support for eight of them in 2018\. Some remaining architectures are starting to look shaky; there will probably be no new products for the Itanium, SPARC M8, or Fujitsu SPARC64 processors, for example. The industry is coalescing mostly on the x86 and Arm architectures at this point. 

Those architectures clearly have new products coming out in 2020 and beyond, so they will be around for a while. There are some others as well. The RISC-V architecture is growing quickly. The IBM Power10 and Z15 architectures are still being developed. [Kalray Coolidge](<https://www.kalrayinc.com/kalray-unveils-coolidge-at-ces-2020-2/>) and [Tachyum Prodigy](<https://www.tachyum.com/products>) are under development without in-kernel support at this point. There is a 64-bit version of the ARC architecture under development with no kernel support yet. There are still MIPS chips coming out from vendors like Loongson and Ingenic and, perhaps surprisingly, still SoCs based on the 20-year-old Armv5 core being introduced. 

Big-endian systems are clearly on their way out, he said. There were a number of architectures that supported both; most are moving to little-endian only. SPARC32 and OpenRISC are holdouts, but their users are expected to migrate to RISC-V in the relatively near future. About the only architecture still going forward with big-endian is IBM Z. 

There are some new SoC architectures in the works. The most significant one is RISC-V, with numerous SoCs from various vendors. Expectations for RISC-V are high, but there are still no products supported in the kernel. The ARC architecture has been around for 25 years and remains interesting; it sees a lot of use in microcontrollers. There is not much support for 32-bit ARC SoCs in the kernel, and no support yet for the upcoming 64-bit version. That support is evidently under development, though. 

Where does all this lead? Bergmann concluded with a set of predictions for what the situation will be in 2030. The market will be split among the x86-64, Armv8+, and RISC-V architectures, he said; it will be difficult for any others to find a way to squeeze in. The upstreaming of support for these architectures in the kernel will continue to improve. IBM Z mainframes will still be profitable. 

The last Armv7 chips, instead, have been released now, but they will still be shipping in 2030 (and in use for long after that). So 32-bit systems will still need to be supported well beyond 2030. For those reasons and more, he is not expecting to see further removals of architecture support from the kernel for at least the next year. 

At the other end, 128-bit architectures, such as [CHERI](<https://www.cl.cam.ac.uk/research/security/ctsrd/cheri/cheri-faq.html>), will be coming into their own. That is likely to be a huge challenge to support in the kernel. The original kernel only supported 32-bit systems until the port to the Alpha architecture happened; that port was only feasible because the kernel was still quite small at the time. The (now much larger) kernel has the assumption that an `unsigned long` is the same size as a pointer wired deeply into it; breaking that assumption is going to be a painful and traumatic experience. Fixing that may be a job for a new generation of kernel hackers.
