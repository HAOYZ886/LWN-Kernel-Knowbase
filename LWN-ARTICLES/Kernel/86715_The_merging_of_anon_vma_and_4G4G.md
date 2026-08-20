---
title: The merging of anon_vma and 4G/4G
url: https://lwn.net/Articles/86715/
date: "May 26, 2004"
category: "Large-memory systems; Memory management-Object-based reverse mapping; Object-based reverse mapping; anon vma"
author: 
---

Immediately prior to releasing 2.6.7-rc1, Linus merged the full remaining set of virtual memory patches from Andrea Arcangeli and Hugh Dickins, including the anon_vma code. This action has raised eyebrows in some quarters; some developers had been under the impression that 2.6 was a stable kernel series. Nobody seems to doubt that the object-based reverse mapping code is a good idea in the long run, but merging it now strikes some developers as unlikely to increase the stability of the 2.6 kernel in the near future. 

Linus [defends](<https://lwn.net/Articles/86718/>) the change in this way: 

It's not "fundamental", in that the reverse mapping is still done. It's just done in a slightly different way. Going to rmap was a _fundamental_ change to how we did VM. In contrast, this was just an "implementation detail". 

Most "implementation details" fit into rather less than 40 individual patches, do not involve difficult special cases (such as making all uses of `mremap()` work correctly), and avoid making significant changes to core parts of the virtual memory subsystem. That said, one should note that the core _decision-making_ VM code has not been changed; the algorithm for choosing pages to move into and out of memory is the same as before. It is also notable that there have been almost no VM-related problem reports since 2.6.7-rc1 was released. This particular change may just work out in the short term after all. 

A related topic is the 4G/4G patch, which separates kernel and user space entirely so that each can make full use of the 4G virtual address space on 32-bit systems. This patch has been considered for merging for some time, but has never quite found its way in. Most developers see it as an ugly hack (though, perhaps, a necessary one), and there is fear of the (possibly overstated) performance overhead that the 4G/4G mode imposes. Even so, some people wonder when this patch might be merged. 

The answer seems to be "never, if at all possible." The motivations behind this patch are (1) to make more kernel-space low memory available on large-memory systems, and (2) to provide a larger virtual address space for applications. The first reason may well have just become moot; the anon_vma patch was merged because, among other things, it significantly reduces the amount of low memory used by the VM subsystem. The initial [reports](<https://lwn.net/Articles/86723/>) suggest that the current VM code handles 32GB of memory nicely on 32-bit systems. Since 32-bit systems rarely come more heavily loaded than that (so far), it is thought that the VM has gotten as good as it needs to be on those systems. 

The real hope, however, is that a serious transition to 64-bit systems will happen before too long. The x86 architecture has been stretched much further than anybody would have expected it to go, and x86_64 makes the transition so easy that there is very little reason not to do it. The 4G/4G patch is likely to hang around (and be included by some distributors) for some time; if nothing else, all of the currently-deployed monster x86 systems are likely to go on running for a while yet. But the mainline kernel may just get away with saying "switch to 64-bit" and leaving that particular patch out.
