---
title: When writeback goes wrong
url: https://lwn.net/Articles/384093/
date: "April 20, 2010"
category: "Lumpy reclaim; Memory management-Writeback"
author: "By Jonathan Corbet April 20, 2010"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
April 20, 2010

Like any other performance-conscious kernel, Linux does not immediately flush data written to files back to the underlying storage. Caching that data in memory can help optimize filesystem layout and seek times; it also eliminates duplicate writes should the same blocks be written multiple times in succession. Sooner or later (preferably sooner), that data must find its way to persistent storage; the process of getting it there is called "writeback." Unfortunately, as some recent discussions demonstrate, all is not well in the Linux writeback code at the moment. 

There are two distinct ways in which writeback is done in contemporary kernels. A series of kernel threads handles writeback to specific block devices, attempting to keep each device busy as much of the time as possible. But writeback also happens in the form of "direct reclaim," and that, it seems, is where much of the trouble is. Direct reclaim happens when the core memory allocator is short of memory; rather than cause memory allocations to fail, the memory management subsystem will go casting around for pages to free. Once a sufficient amount of memory is freed, the allocator will look again, hoping that nobody else has swiped the pages it worked so hard to free in the meantime. 

Dave Chinner recently [encountered a problem](<https://lwn.net/Articles/384110/>) involving direct reclaim which manifested itself as a kernel stack overflow. Direct reclaim can happen as a result of almost any memory allocation call, meaning that it can be tacked onto the end of a call chain of nearly arbitrary length. So, by the time that direct reclaim is entered, a large amount of kernel stack space may have already been used. Kernel stacks are small - usually no larger than 8KB and often only 4KB - so there is not a lot of space to spare in the best of conditions. Direct reclaim, being invoked from random places in the kernel, cannot count on finding the best of conditions. 

The problem is that direct reclaim, itself, can invoke code paths of great complexity. At best, reclaim of dirty pages involves a call into filesystem code, which is complex enough in its own right. But if that filesystem is part of a union mount which sits on top of a RAID device which, in turn, is made up of iSCSI drives distributed over the network, the resulting call chain may be deep indeed. This is not a task that one wants to undertake with stack space already depleted. 

Dave ran into stack overflows - with an 8K stack - while working with XFS. The XFS filesystem is not known for its minimalist approach to stack use, but that hardly matters; in the case he describes, over 3K of stack space was already used before XFS got a chance to take its share. This is clearly a situation where things can go easily wrong. Dave's answer was [a patch](<https://lwn.net/Articles/384112/>) which disables the use of writeback in direct reclaim. Instead, the direct reclaim path must content itself with kicking off the flusher threads and grabbing any clean pages which it may find. 

There is another advantage to avoiding writeback in direct reclaim. The per-device flusher threads can accumulate adjacent disk blocks and attempt to write data in a way which minimizes seeks, thus maximizing I/O throughput. Direct reclaim, instead, takes pages from the least-recently-used (LRU) list with an eye toward freeing pages in a specific zone. As a result, pages flushed by direct reclaim tend to be scattered more widely across the storage devices, causing higher seek rates and worse performance. So disabling writeback in direct reclaim looks like a winning strategy. 

Except, of course, we're talking about virtual memory management code, and nothing is quite that simple. As Mel Gorman [pointed out](<https://lwn.net/Articles/384113/>), no longer waiting for writeback in direct reclaim may well increase the frequency with which direct reclaim fails. That, in turn, can throw the system into the out-of-memory state, which is rarely a fun experience for anybody involved. This is not just a theoretical concern; it [has been observed](<https://lwn.net/Articles/384116/>) at Google and elsewhere. 

[PULL QUOTE:  Lumpy reclaim, by its nature, is likely to create seeky I/O patterns, but skipping lumpy reclaim increases the likelihood of higher-order allocation failures.  END QUOTE] Direct reclaim is also where [lumpy reclaim](<http://lwn.net/Articles/211505/>) is done. The lumpy reclaim algorithm attempts to free pages in physically-contiguous (in RAM) chunks, minimizing memory fragmentation and increasing the reliability of larger allocations. There is, unfortunately, a tradeoff to be made here: the nature of virtual memory is such that pages which are physically contiguous in RAM are likely to be widely dispersed on the backing storage device. So lumpy reclaim, by its nature, is likely to create seeky I/O patterns, but skipping lumpy reclaim increases the likelihood of higher-order allocation failures. 

So various other solutions have been contemplated. One of those is simply putting the kernel on a new stack-usage diet in the hope of avoiding stack overflows in the future. Dave's stack trace, for example, shows that the `select()` system call grabs 1600 bytes of stack before actually doing any work. Once again, though, there is a tradeoff here: `select()` behaves that way in order to reduce allocations (and improve performance) for the common case where the number of file descriptors is relatively small. Constraining its stack use would make an often performance-critical system call slower. 

Beyond that, reducing stack usage - while being a worthy activity in its own right - is seen as a temporary fix at best. Stack fixes can make a specific call chain work, but, as long as arbitrarily-complex writeback paths can be invoked with an arbitrary amount of stack space already used, problems will pop up in places. So a more definitive kind of fix is required; stack diets may buy time but will not really solve the problem. 

One common suggestion is to move direct reclaim into a separate kernel thread. That would put reclaim (and writeback) onto its own stack where there will be no contention with system calls or other kernel code. The memory allocation paths could poke this thread when its services are needed and, if necessary, block until the reclaim thread has made some pages available. Eventually, the lumpy reclaim code could perhaps be made smarter so that it produces less seeky I/O patterns. 

Another possibility is simply to increase the size of the kernel stack. But, given that overflows are being seen with 8K stacks, an expansion to 16K would be required. The increase in memory use would not be welcome, and the increase in larger allocations required to provide those stacks would put more pressure on the lumpy reclaim code. Still, such an expansion may well be in the cards at some point. 

[According to Andrew Morton](<https://lwn.net/Articles/384119/>), though, the real problem is to be found elsewhere: 

The poor IO patterns thing is a regression. Some time several years ago (around 2.6.16, perhaps), page reclaim started to do a LOT more dirty-page writeback than it used to. AFAIK nobody attempted to work out why, nor attempted to try to fix it. 

In other words, the problem is not how direct reclaim is behaving. It is, instead, the fact that direct reclaim is happening as often as it is in the first place. If there were less need to invoke direct reclaim in the first place, the problems it causes would be less pressing. 

So, if Andrew gets his way, the focus of this work will shift to figuring out why the memory management code's behavior changed and fixing it. To that end, Dave has posted [a set of tracepoints](<https://lwn.net/Articles/384120/>) which should give some visibility into how the writeback code is making its decisions. Those tracepoints have already revealed some bugs, which have been duly fixed. The main issue remains unresolved, though. It has already been named as a discussion topic for the upcoming filesystems, storage, and memory management workshop (happening with LinuxCon in August), but many of the people involved are hoping that this particular issue will be long-solved by then.
