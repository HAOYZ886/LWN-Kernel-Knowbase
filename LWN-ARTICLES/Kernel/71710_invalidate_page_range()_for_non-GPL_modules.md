---
title: "invalidate_page_range() for non-GPL modules"
url: https://lwn.net/Articles/71710/
date: "February 18, 2004"
category: "Modules-Licensing"
author: 
---

The kernel function `invalidate_page_range()` is not something which has a lot of callers. Its job is to invalidate all memory mappings which cover a specific part of a file, presumably because the contents of the relevant pages have changed on disk. This function is currently exported only to GPL-licensed modules. 

Paul McKenney has [requested](<https://lwn.net/Articles/71714/>) that this function be exported to all modules. It seems that IBM's GPFS filesystem needs it, and that filesystem is not free software. The claim is that the filesystem is an entirely independent development, and is thus not derived from the kernel; it should not have to be licensed under the GPL to be loadable into the kernel. 

Andrew Morton [says](<https://lwn.net/Articles/71715/>) he is not opposed to the patch. One might think it would not be _too_ controversial, especially since that function was first created and [submitted](<https://lwn.net/Articles/31981/>) by...Paul McKenney. There are developers, however, who believe that any module which is digging that deeply into the virtual memory subsystem cannot help but be derived, in some fashion, from the Linux kernel. There is also, perhaps, a certain desire to demonstrate that even IBM can't obtain arbitrary access to the kernel for proprietary modules. 

In general, the kernel hackers are more interested in seeing their work be useful and used, instead of fighting over licensing battles. So one might expect to see this patch eventually get incorporated. In more recent times, however, some developers have been adopting a firmer position with regard to proprietary modules. This patch may still get in, but it's likely to have a harder time than would have once been the case.
