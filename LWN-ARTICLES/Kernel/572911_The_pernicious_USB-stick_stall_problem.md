---
title: "The pernicious USB-stick stall problem"
url: https://lwn.net/Articles/572911/
date: "November 6, 2013"
category: "Memory management-Writeback"
author: "By Jonathan Corbet November 6, 2013"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
November 6, 2013

Artem S. Tashkinov recently [encountered](<https://lwn.net/Articles/572919/>) a problem that will be familiar to at least some LWN readers. Plug a slow storage device (a USB stick, say, or a media player) into a Linux machine and write a lot of data to it. The entire system proceeds to just hang, possibly for minutes. Things eventually come back, but, by then, the user may well have given up in disgust and gone for a beer or two; by the time the system is usable again, that user may well have lost interest. 

This time around, though, Artem made an interesting observation: the system would stall when running with a 64-bit kernel, but no such problem was experienced when using a 32-bit kernel on the same hardware. One might normally expect the block I/O subsystem to be reasonably well isolated from details like the word length of the processor, but, in this case, one would be surprised. 

#### The problem

Linus was quick to [understand](<https://lwn.net/Articles/572921/>) what was going on here. It all comes down to the problem of matching the rate at which a process creates dirty memory to the rate at which that memory can be written to the underlying storage device. If a process is allowed to dirty a large amount of memory, the kernel will find itself committed to writing a chunk of data that might take minutes to transfer to persistent storage. All that data clogs up the I/O queues, possibly delaying other operations. And, as soon as somebody calls `sync()`, things stop until that entire queue is written. It's a storage equivalent to the bufferbloat problem. 

The developers responsible for the memory management and block I/O subsystems are not entirely unaware of this problem. To prevent it from happening, they have created a set of tweakable knobs under `/proc/sys/vm` to control what happens when processes create a lot of dirty pages. These knobs are: 

  * `**dirty_background_ratio**` specifies a percentage of memory; when at least that percentage is dirty, the kernel will start writing those dirty pages back to the backing device. So, if a system has 1000 pages of memory and `dirty_background_ratio` is set to 10% (the default), writeback will begin when 100 pages have been dirtied. 

  * `**dirty_ratio**` specifies the percentage at which processes that are dirtying pages are made to wait for writeback. If it is set to 20% (again, the default) on that 1000-page system, a process dirtying pages will be made to wait once the 200th page is dirtied. This mechanism will, thus, slow the dirtying of pages while the system catches up. 

  * `**dirty_background_bytes**` works like `dirty_background_ratio` except that the limit is specified as an absolute number of bytes. 

  * `**dirty_bytes**` is the equivalent of `dirty_ratio` except that, once again, it is specified in bytes rather than as a percentage of total memory. 

Setting these limits too low can affect performance: temporary files that will be deleted immediately will end up being written to persistent storage, and smaller I/O operations can lead to lower I/O bandwidth and worse on-disk placement. Setting the limits too high, instead, can lead to the sort of overbuffering described above. 

The attentive reader may well be wondering: what happens if the administrator sets both `dirty_ratio` and `dirty_bytes`, especially if the values don't agree? The way things work is that either the percentage-based or byte-based limit applies, but not both. The one that applies is simply the one that was set last; so, for example, setting `dirty_background_bytes` to some value will cause `dirty_background_ratio` to be set to zero and ignored. 

Two other details are key to understanding the behavior described by Artem: (1) by default, the percentage-based policy applies, and (2) on 32-bit systems, that ratio is calculated relative to the amount of low memory — the memory directly addressable by the kernel, not the full amount of memory in the system. In almost all 32-bit systems, only the first ~900MB of memory fall into the low-memory region. So on any current system with a reasonable amount of memory, a 64-bit kernel will implement `dirty_background_ratio` and `dirty_ratio` differently than a 32-bit system will. For Artem's 16GB system, the 64-bit `dirty_ratio` limit would be 3.2GB; the 32-bit system, instead, sets the limit at about 180MB. 

The (huge) difference between these two limits is immediately evident when writing a lot of data to a slow storage device. The lower limit does not allow anywhere near as much dirty data to accumulate before throttling the process doing the writing, with much better results for the user of the system (unless said user _wanted_ to give up in disgust and go for beer, of course). 

#### Workarounds and fixes

When the problem is that clearly understood, one can start to talk about solutions. Linus suggested that anybody running into this kind of problem can work around it now by setting `dirty_background_bytes` and `dirty_bytes` to reasonable values. But it is generally agreed that the default values on 64-bit systems just don't make much sense on contemporary systems. In fact, [according to Linus](<https://lwn.net/Articles/572928/>), the percentage-based limits have outlived their usefulness in general: 

The percentage notion really goes back to the days when we typically had 8-64 *megabytes* of memory So if you had a 8MB machine you wouldn't want to have more than one megabyte of dirty data, but if you were "Mr Moneybags" and could afford 64MB, you might want to have up to 8MB dirty!! 

Things have changed. 

Thus, he suggested, the defaults should be changed to use the byte-based limits; either that, or the percentage-based limits could be deemed to apply only to the first 1GB of memory. 

Of course, it would be nicer to have smarter behavior in the kernel. The limit that applies to a slow USB device may not be appropriate for a high-speed storage array. The kernel has logic now that tries to estimate the actual writeback speeds achievable with each attached device; with that information, one could try to limit dirty pages based on the amount of time required to write them all out. But, as Mel Gorman [noted](<https://lwn.net/Articles/572929/>), this approach is ""not that trivial to implement"". 

Andreas Dilger [argued](<https://lwn.net/Articles/572930/>) that the whole idea of building up large amounts of dirty data before starting I/O is no longer useful. The Lustre filesystem, he said, will start I/O with 8MB or so of dirty data; he thinks that kind of policy (applied on a per-file basis) could solve a lot of problems with minimal complexity. Dave Chinner, however, [sees a more complex world](<https://lwn.net/Articles/572931/>) where that kind of policy will not work for a wide range of workloads. 

Dave, instead, suggests that the kernel focus on implementing two fundamental policies: "writeback caching" (essentially how things work now) and "writethrough caching," where much lower limits apply and I/O starts sooner. Writeback would be used for most workloads, but writethrough makes sense for slow devices or sequential streaming I/O patterns. The key, of course, is enabling the kernel to figure out which policy should apply in each case without the need for user intervention. There are some obvious indications, including various `fadvise()` calls or high-bandwidth sequential I/O, but, doubtless, there would be details to be worked out. 

In the short term, though, we're most likely to see relatively simple fixes. Linus has posted [a patch](<https://lwn.net/Articles/572933/>) limiting the percentage-based calculations to the first 1GB of memory. This kind of change could conceivably be merged for 3.13; fancier solutions, obviously, will take longer.
