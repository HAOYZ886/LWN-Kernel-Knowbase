---
title: ARM kernel consolidation
url: https://lwn.net/Articles/443510/
date: "May 18, 2011"
category: "Architectures-Arm"
author: "May 18, 2011 This article was contributed by Paul McKenney"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

May 18, 2011

This article was contributed by Paul McKenney

Some of you might have heard about [some discomfort with the state of the ARM architecture](<http://lwn.net/Articles/439314/>) in the kernel recently. Given that ARM Linux consolidation was one of the issues that [Linaro](<http://www.linaro.org/about-linaro/>) was specifically set up to address, it is only fair to ask "What is Linaro doing about this?" So it should not come as a surprise that this topic featured prominently at the recent [Linaro Developers Summit](<https://wiki.linaro.org/Events/2011-05-LDS>) in Budapest, Hungary. 

Duplicate code and out-of-tree patches make Linux on ARM more difficult to use and develop for. Therefore, Linaro is working to consolidate code and to push code upstream. This should make the upstream Linux kernel more capable of handling ARM boards and system-on-chips (SoCs). However, ARM Linux kernel consolidation is an issue not just for Linaro, but rather across the entire ARM Linux kernel community, as well as the ARM SoC, board, and system vendors. Therefore, although I expect that Linaro will play a key role, the ultimate solution spans the entire ARM community. It is also important to note that this effort is a proposal for an experiment rather than a set of hard-and-fast marching orders. 

#### Code organization

If we are to make any progress at all, we must start somewhere. An excellent place to start is by organizing the ARM Linux kernel code by function rather than by SoC/board implementation. Grouping together code with similar purposes will make it easier to notice common patterns and, indeed, common code. For example, currently many ARM SoCs use similar "IP blocks" (such as I2C controllers) but each SoC provides a completely different I2C driver that lives in the corresponding `arch/arm/mach-` directory. We expect that drivers for identical hardware "IP blocks" across different ARM boards and SoCs will be consolidated into a single driver that works with any system using the corresponding IP block. In some cases, differences in the way that a given IP block is connected to the SoC or board in question may introduce complications, but such complications can almost always be addressed. 

This raises the question of where similar code should be moved to. The short answer that was agreed to by all involved is "Not in the `arch/arm` directory!" Drivers should of course move to the appropriate subdirectory of the top-level `drivers` tree. That said, ARM SoCs have a wide variety of devices ranging from touchscreens to GPS receivers to accelerometers, and new types of devices can be expected to appear. So in some cases it might be necessary not merely to move the driver to a new place, but also to create a new place in the `drivers` tree. 

But what about non-driver code? Where should it live? It is helpful to look at several examples: (1) the `struct clk` code that Jeremy Kerr, Russell King, Thomas Gleixner, and many others have been working on, (2) the device-tree code that Grant Likely has been leading up, and (3) the generic interrupt chip implementation that Thomas Gleixner has been working on. 

The `struct clk` code is motivated by the fact that many SoCs and boards have elaborate clock trees. These trees are needed, among other things, to allow the tradeoff between performance and energy efficiency to be set as needed for individual devices on that SoC or board. The `struct clk` code allows these trees to be represented with a common format while providing plugins to accommodate behavior specific to a given SoC or board. The [generic interrupt chip implementation](<http://lwn.net/Articles/439176/>) has a similar role, but with respect to interrupt distribution rather than clock trees. 

[Device trees](<http://lwn.net/Articles/414016/>) are intended to allow the hardware configuration of a board to be represented via data rather than code, which should ease the task of creating a single Linux kernel binary that boots on a variety of ARM boards. The device-tree infrastructure patches have recently been [accepted by Russell King](<http://article.gmane.org/gmane.linux.linaro.devel/4170>), which should initiate the transition of specific board code to device tree descriptions. 

The `struct clk` code is already used by both the ARM and SH CPU architectures, so it is not ARM-specific, but rather core Linux kernel code. Similarly, the device-tree code is not ARM-specific; it is also used by the PowerPC, Microblaze, and SPARC architectures, and even by [x86](<http://lwn.net/Articles/424401/>). Device tree therefore is also Linux core kernel code. The virtual-interrupt code goes even further, being common across all CPU architectures. The lesson here is that ARM kernel code consolidation need not necessarily be limited to ARM. In fact, the more architectures that a given piece of code supports, the more developers can be expected to contribute both code and testing to it, and the more robust and maintainable that code will be. 

There will of course need to be at least some ARM-specific code, but the end goal is for that code to be limited to ARM core architecture code and ARM SoC core architecture code. Furthermore, the ARM SoC core architecture code should consist primarily of small plugins for core-Linux-kernel frameworks, which should in turn greatly ease the development and maintenance of new ARM boards and SoCs. 

It is all very easy to write about doing this, but quite another to actually accomplish it. After all, although there are a good number of extremely talented and energetic ARM developers and maintainers, many of the newer ARM developers are also new to the Linux kernel, and cannot be expected to to know where new code should be placed. Such people might be tempted to continue placing most of their code in their SoC and board subdirectories, which would just perpetuate the current ARM Linux kernel difficulties. 

Part of the solution will be additional documentation, especially on writing ARM drivers and board ports. Deepak Saxena, the new Linaro Kernel Working Group lead, will be making this happen. Unfortunately, documentation is only useful to the extent that anyone actually reads it. Fortunately, just as every problem in computer science seems to be solvable by adding an additional level of indirection, every maintainership problem seems to be solvable by adding an additional git tree and maintainers. These maintainers would help generate common code and of course point developers at documentation as it becomes available. 

#### Git trees

One approach would be to use Nicolas Pitre's existing Linaro kernel git tree. However, Nicolas's existing git tree is an [ integration tree](<https://wiki.linaro.org/WorkingGroups/Kernel/KernelTree>) that allows people to easily pull the latest and greatest ARM code against the most recent mainline kernel version. In contrast, a maintainership tree contains patches that are to be upstreamed, normally based on a more-recent mainline release candidate. If we tried to use a single git tree for both integration and for maintainership, we would either unnecessarily expose ARM users to unrelated core-kernel bugs, or we would fail to track mainline closely enough for maintainership, which would force a full rebase and testing cycle to happen in a very short time at the beginning of each merge window. 

Of course, in theory we could have both maintainership and integration branches within the same git tree, but separating these two very different functions into separate git trees is most likely to work well, especially in the beginning. 

This new git tree (which was [announced](<https://lwn.net/Articles/443515/>) on May 18) will have at least one branch per participating ARM subarchitecture, and these branches will not be normally subject to rebasing, thus making it easy to develop against this new tree. Following the usual practice, maintainers of participating ARM sub-architectures will send pull requests to a group of maintainers for this new tree. Also following the usual practice, a merge of all the branches will be sent to Stephen Rothwell's -next tree, but the branches will be individually pushed to Linus Torvalds, perhaps via Russell King's existing ARM tree. 

The pushing of individual branch to Linus might seem surprising, but Linus really does want to see the conflicts that arise. Such conflicts presumably help Linus identify areas in need of his attention. 

Of course, this new git tree will not be limited to Linaro, but neither is it mandatory outside of Linaro. That said, I am very happy to say that some maintainers outside of Linaro have expressed interest in participating in this effort. 

The Budapest meeting put forward a list of members of the maintainership group for this new git tree, namely Arnd Bergmann, Nicolas Pitre, and Marc Zyngier, with help from Thomas Gleixner. Russell King will of course also have write access to this tree. The tree will be set up in time to handle the 2.6.41 merge window. The plan is to start small and grow by evolution rather than by any attempts at intelligent design. 

As noted at the beginning of this article, this effort is an experiment rather than a set of hard-and-fast marching orders. Although this proposed experiment cannot be expected to solve each and every ARM Linux problem, they will hopefully provide a good start. Every little bit helps, and every cleanup frees a little time to start on the next cleanup. There is reason to hope that this effort will help to reduce the "[endless amounts of new pointless platform code](<http://lwn.net/Articles/439326/>)" that irritated Linus Torvalds last month. 

####  Acknowledgments

I owe thanks to the many people who helped take notes at the recent Linaro Developers Summit in Budapest, and to all the people involved in the discussions, both in the room and via IRC. Special thanks go to Jake Edge, David Rusling, Nicolas Pitre, Deepak Saxena, and Grant Likely for their review of an early draft of this article. However, all remaining errors and omissions are the sole property of the author.
