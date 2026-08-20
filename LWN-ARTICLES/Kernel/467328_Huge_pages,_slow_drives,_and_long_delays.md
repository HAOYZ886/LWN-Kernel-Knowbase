---
title: "Huge pages, slow drives, and long delays"
url: https://lwn.net/Articles/467328/
date: "November 14, 2011"
category: "Huge pages; Memory management-Huge pages"
author: "By Jonathan Corbet November 14, 2011"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
November 14, 2011

It is a rare event, but it is no fun when it strikes. Plug in a slow storage device - a USB stick or a music player, for example - and run something like `rsync` to move a lot of data to that device. The operation takes a while, which is unsurprising; more surprising is when random processes begin to stall. In the worst cases, the desktop can lock up for minutes at a time; that, needless to say, is not the kind of interactive response that most users are looking for. The problem can strike in seemingly arbitrary places; the web browser freezes, but a network audio stream continues to play without a hiccup. Everything unblocks eventually, but, by then, the user is on their third beer and contemplating the virtues of proprietary operating systems. One might be forgiven for thinking that the system should work a little better than that. 

Numerous people have reported this sort of behavior in recent times; your editor has seen it as well. But it is hard to reproduce, which means it has been hard to track down. It is also entirely possible that there is more than one bug causing this kind of behavior. In any case, there should now be one less bug of this type if [Mel Gorman's patch](<https://lwn.net/Articles/467332/>) proves to be effective. But a few developers are wondering if, in some cases, the cure is worse than the disease. 

The problem Mel found appears to go somewhat like this. A process (that web browser, say) is doing its job when it incurs a page fault. This is normal; the whole point of contemporary browsers sometimes seems to be to stress-test virtual memory management systems. The kernel will respond by grabbing a free page to slot into the process's address space. But, if the transparent huge pages feature is built into the kernel (and most distributors do enable this feature), the page fault handler will attempt to allocate a huge page instead. With luck, there will be a huge page just waiting for this occasion, but that is not always the case; in particular, if there is a process dirtying a lot of memory, there may be no huge pages available. That is when things start to go wrong. 

Once upon a time, one just had to assume that, once the system had been running for a while, large chunks of physically-contiguous memory would simply not exist. Virtual memory management tends to fragment such chunks quickly. So it is a bad idea to assume that huge pages will just be sitting there waiting for a good home; the kernel has to take explicit action to cause those pages to exist. That action is [compaction](<https://lwn.net/Articles/368869/>): moving pages around to defragment the free space and bring free huge pages into existence. Without compaction, features like transparent huge pages would simply not work in any useful way. 

A lot of the compaction work is done in the background. But current kernels will also perform "synchronous compaction" when an attempt to allocate a huge page would fail due to lack of availability. The process attempting to perform that allocation gets put to work migrating pages in an attempt to create the huge page it is asking for. This operation is not free in the best of times, but it should not be causing multi-second (or multi-minute) stalls. That is where the USB stick comes in. 

If a lot of data is being written to a slow storage device, memory will quickly be filled with dirty pages waiting to be written out. That, in itself, can be a problem, which is why the recently-merged [I/O-less dirty throttling code](<https://lwn.net/Articles/456904/>) tries hard to keep pages for any single device from taking too much memory. But writeback to a slow device plays poorly with compaction; the memory management code cannot migrate a page that is being written back until the I/O operation completes. When synchronous compaction encounters such a page, it will go to sleep waiting for the I/O on that page to complete. If the page is headed to a slow device, and it is far back on a queue of many such pages, that sleep can go on for a long time. 

One should not forget that producing a single huge page can involve migrating hundreds of ordinary pages. So once that long sleep completes, the job is far from done; the process stuck performing compaction may find itself at the back of the writeback queue quite a few times before it can finally get its page fault resolved. Only then will it be able to resume executing the code that the user actually wanted run - until the next page fault happens and the whole mess starts over again. 

Mel's fix is a simple one-liner: if a process is attempting to allocate a transparent huge page, synchronous compaction should not be performed. In such a situation, Mel figured, it is far better to just give the process an ordinary page and let it continue running. The interesting thing is that not everybody seems to agree with him. 

Andrew Morton was the first to [object](<https://lwn.net/Articles/467338/>), saying ""Presumably some people would prefer to get lots of huge pages for their 1000-hour compute job, and waiting a bit to get those pages is acceptable."" David Rientjes, presumably thinking of Google's throughput-oriented tasks, [said](<https://lwn.net/Articles/467339/>) that there are times when the latency is entirely acceptable, but that some tasks really want to get huge pages at fault time. Mel's change makes it that much less likely that processes will be allocated huge pages in response to faults; David does not appear to see that as a good thing. 

One could (and Mel did) respond that the transparent huge page mechanism does not only work at fault time. The kernel will also try to replace small pages with huge pages in the background while the process is running; that mechanism should bring more huge pages into use - for longer-running processes, at least - even if they are not available at fault time. In cases where that is not enough, there has been talk of adding a new knob to allow the system administrator to request that synchronous compaction be used. The actual semantics of such a knob are not clear; one could argue that if huge page allocations are that much more important than latency, the system should perform more aggressive page reclaim as well. 

Andrea Arcangeli [commented](<https://lwn.net/Articles/467667/>) that he does not like how Mel's change causes failures to use huge pages at fault time; he would rather find a way to keep synchronous compaction from stalling instead. Some ideas for doing that are being thrown around, but no solution has been found as of this writing. 

Such details can certainly be worked out over time. Meanwhile, if Mel's patch turns out to be the best fix, the decision on merging should be clear enough. Given a choice between (1) a system that continues to be responsive during heavy I/O to slow devices and (2) random, lengthy lockups in such situations, one might reasonably guess that most users would choose the first alternative. Barring complications, one would expect this patch to go into the mainline fairly soon, and possibly into the stable tree shortly thereafter.
