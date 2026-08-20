---
title: Filesystem metadata memory management
url: https://lwn.net/Articles/752552/
date: "April 24, 2018"
category: Filesystems
author: "By Jonathan Corbet April 24, 2018 LSFMM"
---

By **Jonathan Corbet**  
April 24, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

It is a good thing that strong coffee was served at the 2018 Linux Storage, Filesystem, and Memory-Management Summit; full awareness was required from the first session, in which Josef Bacik discussed some issues that have arisen in the interaction between filesystems and the memory-management subsystem. Filesystems cache a lot of data from files, but also a lot of metadata about those files. It turns out, though, that management of the cached metadata does not work as well as one might like. 

In particular, Bacik said, he has been working on better tracking of dirty metadata in memory. This is a problem that every filesystem faces, and each one has come up with its own solution. Left unchecked, the amount of dirty metadata in the system is limited only by the amount of memory available — which can be problematic, given that the owner of the system usually wants to use at least some of that memory for other purposes. The Btrfs filesystem uses a special inode to track and limit dirty metadata, but that leads to "horrifying things" around the process of evicting that metadata when the time comes. Among other things, it doesn't support sub-page-sized blocks. 

The Btrfs developers would like to take an approach closer to what is done in XFS, he said. That approach works well, but still suffers from the problem that the metadata cache can grow to fill all of memory. To address this problem, he's working on adding separate counters to track metadata [![\[Josef Bacik\]](https://static.lwn.net/images/conf/2018/lsfmm/JosefBacik-sm.jpg)](<https://lwn.net/Articles/752574/>) pages, with the idea that filesystems will get a writeback callback from the memory-management subsystem indicating the number of these pages that should be freed. This callback will come from [`balance_dirty_pages()`](<https://elixir.bootlin.com/linux/latest/source/mm/page-writeback.c#L1553>), which knows how much memory it needs to free and can pass that information down to filesystem code. 

Dave Chinner noted that memory accounting at this level is all based on pages, which doesn't necessarily match the problem of tracking filesystem metadata, which exists in many sizes from a fraction of a page to many pages. So tracking dirty metadata in terms of pages doesn't really solve the problem. Bacik responded that he has changed all of the counters to units of bytes to address just this problem. He asked the memory-management developers in the room whether they objected to this change; nobody seemed to. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Moving on, he said that `balance_dirty_pages()` has a global limit of dirty memory that encompasses both data and metadata pages. But, he asserted, it is generally better to write out data pages than metadata pages when memory must be freed, so Btrfs focuses on the data pages first. Is that behavior good enough, he asked, or should there be separate limits for data and metadata? 

Johannes Weiner said that one limit should apply to both types, since the system is committed to writing back all of the dirty data in the end. Jan Kara said, though, that there is a fundamental difference between writing back data, which just has to be pushed out of the page cache, and metadata, which must be written back via calls to shrinkers. Bacik said that the shrinker interface is a part of the problem, since it is not well designed. The Btrfs changes are going to put more pressure on the shrinker interface, since there will be unknown amounts of data attached to each object that might be written back. There is, he said, a lot of memory hidden in places where the memory-management subsystem has no idea how to find it. 

Rik van Riel said that, when a filesystem allocates memory, it expects the memory-management code to be able to free up enough memory to satisfy the request. But, while allocations require pages, shrinkers work on smaller objects. A shrinker may free the amount of memory requested by the memory-management code, but those objects can be scattered across memory and the end result may be to free no pages at all. As more memory is taken up by filesystem caches, he admonished, the filesystem has to free those caches or it will find its allocation requests failing. Kara responded, though, that fragmentation tends not to be a huge problem in this context, since the objects involved are relatively large. 

Chinner said that XFS does the majority of its metadata writeback by way of shrinkers, but also all of its page-reclaim work. The XFS cache structure is too complicated to express in the page cache, so the page cache is not used for this purpose. So, for XFS, there are no problems resulting from the shrinkers and the page cache not working together. Bacik said, though, that this results in XFS tending to write data randomly during reclaim, which can lead to excessive latencies, so Facebook has had to patch that feature out. There needs to be a way to do XFS metadata writeback in places where latency is expected. 

Mike Snitzer suggested that "bufio", a shrinker-based mechanism used by the device mapper might be able to help here; it has recently been enhanced to handle objects with sizes that are not a power of two. Bacik said he'd looked at it, but he thinks that maybe the shrinker interface is not the best one for this problem. Chinner said the solution could be a new type of shrinker; the same mechanism could be used, but the accounting would be adapted to handle the metadata issues. 

Chris Mason said that Btrfs has a set of throttling hooks that slow down allocations in places that are known to be safe; it is working well. The only caveat is that things fall apart if the `kswapd` thread starts performing I/O. In general, Weiner said, the entity that is issuing writes is not the one that is performing allocations, so throttling writes is the wrong way to address the problem. There should be two separate throttling points, one for writes and one for allocations; conflating the two has caused a lot of problems. 

Bacik closed the session by saying that he will do the work to add a new shrinker interface with byte-based accounting for metadata.
