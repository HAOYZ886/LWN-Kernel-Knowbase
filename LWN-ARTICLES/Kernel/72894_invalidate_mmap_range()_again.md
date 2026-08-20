---
title: invalidate_mmap_range() again
url: https://lwn.net/Articles/72894/
date: "February 25, 2004"
category: "Modules-Licensing"
author: 
---

The question of whether `invalidate_mmap_range()` should be exported to non-GPL modules was discussed here [last week](<http://lwn.net/Articles/70926/>). There still has been no (public) resolution of the question as of this writing, but the discussion has progressed somewhat. This issue may give some hints as to how other export requests may be viewed in the future. 

Andrew Morton [posted](<https://lwn.net/Articles/72899/>) two criteria which should be used in considering the request. The first is: does the export make sense from a technical point of view? In other words, is the ability to clear page table entries which point at the page cache a legitimate feature for filesystems to want? The consensus answer here appears to be "yes"; distributed filesystems, in particular, will need this capability. 

Andrew also noted that the technical question really should be the only one that matters. If there is a valid technical reason for filesystems to use that function, it should be exported to them. In the real world, however, a second question must also be considered: is IBM's proprietary GPFS filesystem, being the module driving the proposed export change, a derived product of the kernel or not? Here there is less of a consensus. 

IBM's claim is that GPFS was developed under AIX and simply ported to Linux; it is thus an independent development and clearly not derived from the Linux kernel. Critics point to the large, BSD-licensed layer of glue code which is required to make GPFS actually work with Linux; this layer, they say, shows that GPFS does so much messing around with kernel internals (rather than using the existing, exported interface) that it must be a derived product. Interestingly, IBM supporters also point to the large glue layer. If GPFS were truly derived from the kernel, they say, there would be no need for a large impedance-matching layer. 

Without access to the GPFS source, it is going to be hard for any independent party to make a real determination on the status of GPFS. In the end, however, somebody is going to have to make a decision anyway. The odds would appear to favor IBM getting what it wants in this case. But a clear message is being sent: the kernel developers are increasingly suspicious of (and hostile to) changes which make life easier for vendors of closed-source modules.
