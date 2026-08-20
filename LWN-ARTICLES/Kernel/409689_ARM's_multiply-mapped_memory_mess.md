---
title: "ARM's multiply-mapped memory mess"
url: https://lwn.net/Articles/409689/
date: "October 12, 2010"
category: "Architectures-Arm; Development model; ioremap"
author: "By Jonathan Corbet October 12, 2010"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
October 12, 2010

As a general rule, kernel changes which break drivers at run time are not seen as a good thing. Silent data corruption is also seen as the sort of outcome that the development community would rather avoid. What happens when it becomes necessary to choose one or the other? A long-running debate in the ARM community provides at least one answer. 

First, some background. Contemporary processors do not normally address memory directly; instead, memory accesses are mediated through mappings created in the hardware's memory management unit. Depending on the processor, those mappings may be controlled through segment registers, page table entries, or some other means. The mapping will translate a virtual address into a physical address, but it also controls how the processor will access that memory and, perhaps, cache its contents. 

As [explained](<https://lwn.net/Articles/409700/>) by ARM maintainer Russell King, ARM processors have a number of attributes which affect how memory mappings work. There is the concept of a memory type; "normal memory" is subject to reordering of reads and writes, while "device memory" is not, for example. There is also a bit indicating whether memory can be shared between processors; unshared memory is faster because there is no need to worry about cross-processor cache coherency. Then, like many CPUs, ARM processors can specify different caching behavior in the mapping; RAM might be mapped with writeback caching enabled, while device memory is uncached. 

The ARM kernel maps RAM as normal memory with writeback caching; it's also marked non-shared on uniprocessor systems. The `ioremap()` system call, used to map I/O memory for CPU use, is different: that memory is mapped as device memory, uncached, and, maybe, shared. These different mappings give the expected behavior for both types of memory. Where things get tricky is when somebody calls `ioremap()` to create a new mapping for system RAM. 

The problem with these multiple mappings is that they will have differing attributes. As of version 6 of the ARM architecture, the specified behavior in that situation is "unpredictable." Users, as a rule, are not enamored of "unpredictable' behavior, especially when their data is involved. So it would make sense to avoid multiple memory mappings with differing attributes. The ARM architecture has traditionally allowed this kind of mapping, though, and a number of drivers, as a result, rely on being able to remap RAM in this way. 

Back in April, Russell raised an alarm about this issue, and [posted a patch](<http://lists.arm.linux.org.uk/lurker/message/20100408.094818.d6854bd5.en.html>) causing `ioremap()` to fail when the target is system RAM. This change avoids the potential data corruption issue, but at the cost of breaking every driver using `ioremap()` in this way. There were complaints at the time, so the patch sat out the 2.6.35 development cycle, but Russell merged it for 2.6.36. There it sat until, with the release imminent, Felipe Contreras [posted a patch](<https://lwn.net/Articles/409705/>) backing out the change, saying: 

Many drivers are broken, and there's no alternative in sight. Such a big change should stay as a warning for now, and only later should it actually fail. 

Russell was not impressed. In his view, remapping RAM in this way is a dangerous technique which will lead to data corruption sooner or later. Despite being warned six months ago, driver developers have not fixed the problem - there are as many broken drivers now as there were before. So, he says, there is no benefit to waiting any longer; the dangerous behavior should be stopped before somebody gets burned. 

On the other side, driver developers point out that everything "seems to work" as it is, so there is no urgent need for change. Furthermore, Russell's patch looks to them like an API change, and the normal rule of kernel development is that anybody making internal API changes is charged with cleaning up any resulting messes. Fixing the drivers is not a trivial task, and it's Russell's contention that they have always been broken, so he is not willing (or necessarily able) to make them all work again. 

The situation looked stalled, with a reversion of the patch looking like the only way forward. But, in fact, it looks like there is a way out. The first is to allow those mappings for one more cycle, but to put in a user-visible warning when they happen. As Andrew Morton [put it](<https://lwn.net/Articles/409709/>): 

We *do* have a plan: as of 2.6.36, the kernel will emit a WARN_ON trace when a driver does this. Offending code will be discovered, developers will get bug reports from worried users, etc. This is usually pretty effective. 

It is the "worried users" who have been missing from the equation so far; they can provide a type of pressure which, seemingly, is unavailable to worried subsystem maintainers. 

The other piece of the solution is to give driver developers a way to obtain a chunk of physically-contiguous RAM which can be remapped in this way. Such memory cannot be mapped simultaneously as system RAM. One nice idea would be to simply unmap system memory when it is put to a device's use, but that proves to be difficult to implement. The alternative is to just set aside some memory at boot time and never let the kernel use it for any other purpose; drivers can then allocate from that pool when they need to. Russell has posted [a patch](<https://lwn.net/Articles/409713/>) which makes this kind of memory set-aside possible. 

So this particular situation will probably have a happy outcome, presuming that the above outcome happens and that that no users are burned by unpredictable mappings with the 2.6.36 kernel. But it highlights some ongoing problems. It can be very hard to get developers to fix things, especially if the current code "seems to work." Those developers also became aware of the change at a very late date - if, indeed, they are even aware of it now. It seems that testing of -rc kernels by developers is not happening as much as we would like. Still, the development process seems to work, and problems like this are overcome eventually.
