---
title: "KS2012: memcg/mm: Moving zcache towards the mainline"
url: https://lwn.net/Articles/516538/
date: "September 17, 2012"
category: "Memory management-Swapping"
author: "By Michael Kerrisk September 17, 2012 2012 Kernel Summit"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Michael Kerrisk**  
September 17, 2012

* * *

[2012 Kernel Summit](<https://lwn.net/Articles/KernelSummit2012/>)

During the 2012 Kernel Summit [memcg/mm minisummit](<https://lwn.net/Articles/516439/>), Konrad Wilk briefly introduced [zcache](<https://lwn.net/Articles/397574/>) as a prelude to considering how to bring zcache into the mainline kernel. Zcache reserves a portion of memory for compressed page storage. As memory gets tight and the system is about to swap, the kernel will instead start compressing pages. This defers having to transfer pages to much slower secondary storage or, optimally, completely avoids doing so, depending on the workload. 

Zcache currently lives in the staging tree and Greg Kroah-Hartman was recently [asked](<https://lwn.net/Articles/516435/>) what was necessary to move it out. Konrad asked if anyone in the room had a chance to review the code, but it was clear that no one had within the last year or so. He said that there are a few users interested in developing features on top of zcache, but that developing against something in staging is high-risk. 

Andrew Morton felt that staging is basically a graveyard as far as zcache is concerned and maybe the time has come to try and move it to the core VM and force the memory-management developers to review it. It may also be necessary to revisit whether the feature is even necessary and whether distributions will support it. However, apparently there are already users of zcache and KVM is interested if the sizing of the cache can be properly managed; thus, it should be possible to justify moving it to core VM. 

There were concerns that compressed RAM features always look good on paper, but that there are too many workloads whose pages do not compress well; there were also questions around how to determine the optimal size of the compressed cache. Andrea Arcangeli was also concerned that the first pages to be evicted from memory are, by definition of the LRU page order, the ones that are least likely to be used in the future. These are the pages that should be going to secondary storage and more frequently used pages should be going to zcache. As it stands, zcache may fill up with no-longer-used pages and then the system continues to move used pages from and to the disk. 

Konrad says he can address these concerns, but also tried to manage expectations, saying that there were some known problems with the current zcache implementation and people should be aware that there are circumstances where it could trigger the OOM killer. 

[Next: Dirty/writeback LRU](<https://lwn.net/Articles/516539/>)
