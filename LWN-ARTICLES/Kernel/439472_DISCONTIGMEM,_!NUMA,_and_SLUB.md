---
title: "DISCONTIGMEM, !NUMA, and SLUB"
url: https://lwn.net/Articles/439472/
date: "April 20, 2011"
category: "Memory management-Slab allocators"
author: "By Jonathan Corbet April 20, 2011"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
April 20, 2011

The kernel has two different ways of dealing with systems where there are large gaps in the physical memory address space - DISCONTIGMEM and SPARSEMEM. Of those two, DISCONTIGMEM is the older; it has been semi-deprecated for some time and appears to be on its (slow) way out. But some architectures still use it. Recent changes (and the resulting crashes) have shown that there are some interesting misunderstandings about how DISCONTIGMEM is handled in the kernel. 

The problem comes down to this: DISCONTIGMEM tracks separate ranges of memory by putting each into its own virtual NUMA node. The result is that a system running in this mode can appear to have multiple NUMA nodes, even if NUMA support is not configured in. That apparently works well much of the time, but it recently has been shown to cause crashes in the SLUB allocator, which is not prepared for the appearance of multiple NUMA nodes on a non-NUMA system. 

There was a surprisingly acrimonious discussion on just whose fault this misunderstanding is and how to fix it. Options including changing DISCONTIGMEM to not "abuse" (in some peoples' view) the NUMA concept in this way; that might be a long-term solution, but the bug exists now and, as James Bottomley [put it](<https://lwn.net/Articles/439475/>): ""That has to be fixed in -stable. I don't really think a DISCONTIGMEM re-engineering effort would be the best thing for the -stable series."" Another option is to force NUMA support to be configured in when DISCONTIGMEM is used; that could bloat the kernel on embedded systems and requires acceptance of the strange concept that uniprocessor systems can be NUMA. The kernel could be fixed to handle non-zero NUMA nodes at all times; that could involve a significant code audit as the problems might not be limited to the SLUB allocator. The SLUB allocator could be disallowed on non-NUMA DISCONTIGMEM systems, but, once again, there may be issues elsewhere. Or the process of escorting DISCONTIGMEM out of the kernel could be expedited \- though that would not be suitable for the stable series. 

As of this writing the discussion continues; it's not clear what form the real solution will take. The problem is subtle and there do not appear to be any easy fixes at hand.
