---
title: "High-order GFP_ATOMIC allocation trouble"
url: https://lwn.net/Articles/362179/
date: "November 17, 2009"
category: Memory management
author: "By Jonathan Corbet November 17, 2009"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
November 17, 2009

On its face, memory management would appear to be a straightforward task. When memory gets tight, the VM code need only evict the pages which will be unused for the longest time, making that memory available for shorter-term use. The hard part, of course, is identifying those pages. In the absence of perfect predictions of future memory use, the VM subsystem must rely upon a set of heuristics to make a set of (hopefully) reasonable choices. The design of heuristics which can handle most workloads is tricky, and even subtle code changes can lead to big changes in system behavior. 

Since the beginning of the 2.6.31 development cycle, some users have been complaining about an increase in kernel memory allocation failures, leading to log messages, failed applications, and the occasional unwelcome appearance of the out-of-memory killer. Various bugs have been filed (see [#14141](<http://bugzilla.kernel.org/show_bug.cgi?id=14141>) and [#14265](<http://bugzilla.kernel.org/show_bug.cgi?id=14265>), for example) and a fair amount of head-scratching has gone on. But few developers really know where to start when looking at this kind of problem, and, of those who do, some have been content to write off the problem as being caused by higher-order allocations. So progress has been slow. 

High-order (multi-page) allocations are a perennial problem on Linux systems; as memory fragments, it gets increasingly hard to find groups of physically-contiguous pages to satisfy higher-order allocation requests. Whenever possible, kernel code is written to avoid high-order allocations, but there are times when that is difficult. Many of the recently-reported problems seemingly have to do with certain not-top-of-the-line wireless network adapters which require contiguous memory chunks to operate. Fixing the problem is important - users of cheap network interfaces want to run Linux too - but there are also reports of single-page allocation failures. 

Fortunately, Mel Gorman is not afraid to wander into that part of the kernel; he has been putting some serious time into reproducing the problem and trying to understand what has gone wrong since 2.6.30. Mel has posted [a five-part patch series](<https://lwn.net/Articles/362184/>) which tries to make allocation failures less likely again. Looking at what Mel has done provides a good lesson on just how subtle this kind of programming can be. 

When looking at this code, it's worth bearing in mind that the kernel has two fundamental mechanisms for recovering memory when it is needed for new allocations. **Direct reclaim** is active memory cleaning done at allocation time; when an allocation falls short, the process trying to allocate the memory will go off and try to free some memory elsewhere in the system. Direct reclaim has the advantages of immediacy - reclaim work happens right away when memory pressure hits - and of dumping the work into processes which are allocating memory, but there are limits to how long any one process can spend reclaiming memory without introducing unacceptable latencies. So more extensive cleaning is pushed off to the **kswapd** kernel thread, which is dedicated to that task. 

Current mainline kernels do not wake up kswapd from the direct reclaim code if the direct reclaim operation fails to get the job done. But if memory is that tight, kswapd should be running, especially if high-order allocations are needed. So the first patch in Mel's series is a simple one-liner which causes kswapd to be waked on direct allocation failure and, perhaps, to work harder on recovering higher-order chunks as well. That change brings behavior back to something closer to what older kernels did. 

Patch #2 is a simple tweak which keeps realtime interrupt handlers from driving the memory allocation code too hard. Again, this is a reversion to behavior seen back in the 2.6.30 days. 

The third patch is a bit more subtle. Direct reclaim will, if it is successful, result in the creation of I/O operations to write dirty pages to their backing store. There are limits to the number of block I/O operations which can be outstanding, though; once that limit is hit the underlying device is said to be "congested" and the task performing reclaim is forced to wait until things clear out a bit. This "congestion wait" keeps the system from filling up with pending I/O operations and serves to throttle processes performing memory allocations. 

As it happens, there are actually two "wait for congestion" queues - one each for synchronous and asynchronous requests. "Synchronous" requests are those for which a process is actively waiting - read requests, usually - while asynchronous requests are those which do not have active waiters. In current kernels, direct reclaim waits on the asynchronous queue, while older kernels used the synchronous queue instead. Moving back to the synchronous queue makes a number of problems go away, but Mel sees that fix as being workload-specific. Instead, he has changed the direct reclaim code to make it wait for congestion to clear on _both_ queues. 

Why does this help? It seems to be a matter of letting kswapd get its job done. Kswapd, too, must wait when queues become congested; if direct reclaimers are frequently filling the I/O queues, kswapd will stall more often. It turns out that better results are had if kswapd is allowed to run for longer periods of time. Making direct reclaimers wait until both queues have cleared allows kswapd to get some real work done once it gets going. That is good for the creation of high-order chunks and the performance of the system in general. 

Patch #4 also relates to kswapd's duty cycle. Kswapd will stop working and go to sleep once it decides that it has done enough; one definition of "enough" is when the amount of free memory reaches an upper watermark value. But if kswapd is running, chances are good that there is unmet demand for memory in the system; in that situation, the amount of free memory may not stay above the high watermark for very long. Mel's patch has kswapd start with a catnap rather than a real sleep; after 0.1 sec., kswapd wakes back up and reassesses the situation. If the amount of free memory has fallen below the high watermark in that time, kswapd goes back to work; otherwise it goes to sleep for real. In this way, kswapd will continue to work to free memory if the system is consuming it quickly. 

The final patch touches on another aspect of waiting for congestion. When block devices become congested, kswapd waits for things to clear. But, Mel notes, that may not be the right thing to do in all situations: 

However, on systems with large numbers of high-order atomics due to crappy network cards, it's important that kswapd keep working in parallel to save their sorry ass. 

In the original version of the patch, kswapd would become increasingly resistant to waiting for congestion as the situation got worse. Motohiro Kosaki [suggested an alternative approach](<https://lwn.net/Articles/362218/>), though, wherein kswapd simply refuses to wait as long as the high watermark is not reached, and Mel adopted it. 

Mel's patch posting includes a fair amount of information on how he has tested it and what the results are. With the patch set applied, allocation failures are fewer, and system throughput improves as well. The sad truth about memory management patches, though, is that a change which improves one workload may worsen another. So these changes really need some widespread testing, especially since there is some interest in getting them into 2.6.32.
