---
title: "LSFMM: Writeback latency issues"
url: https://lwn.net/Articles/548112/
date: "April 23, 2013"
category: "Memory management-Swapping"
author: "By Jonathan Corbet April 23, 2013 LSFMM Summit 2013"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
April 23, 2013

* * *

[LSFMM Summit 2013](<https://lwn.net/Articles/LSFMM2013/>)

As Rik van Riel pointed out at the beginning of this session, it would not be a proper summit without a discussion of writeback. Writeback — the process of writing dirty pages of memory back to persistent storage — has been a problematic performance issue for some years. Significant progress has been made in this area, though, as indicated by both the relatively small amount of time devoted to writeback this year and the nature of the session. At this point, the writeback discussion is dedicated to the solution of problems that tend to show up with specific workloads, rather than worrying about the state of writeback in general. 

One problem, brought up by Mel Gorman, has to do with page allocation being slower than people would like at times. The delays happen in direct reclaim (when an allocating process is put to work finding pages to reclaim because nothing is immediately available for allocation). The problem is somewhat mysterious at this point, it happens even when there are plenty of clean pages in the page cache. Somebody just needs to put in the time to observe the problem and figure out where it is coming from. 

Another issue is the speed of allocation of transparent huge pages. This problem is better understood: allocating a huge page may require performing [compaction](<https://lwn.net/Articles/368869/>) — moving other pages out of the way so that a suitably large, physically-contiguous page can be made. The compaction process takes a while, to the point that application startup can take twice as long when transparent huge pages are enabled. In some settings, that represents a crippling performance regression. 

One possible solution here is to give up immediately if a huge page is not available when a page fault happens. Instead, an ordinary page can be allocated and the `khugepaged` daemon can, hopefully, replace that page with a huge page at some point in the future. Andi Kleen argued against this idea, though, saying that it was far too slow. It is better, he said, to just pay the cost up front to get into the desired state from the beginning. Mel would like to find ways to make compaction go faster, but it's not clear what those would be at this point. 

On another front, it seems that the memory management subsystem has a tendency to reclaim the block bitmaps used by filesystems. Those bitmaps are needed to satisfy future I/O requests, so they will be read back in quickly; in the meantime, though, performance suffers. There was some discussion of the mechanism by which these particular pages are not being moved to the system's active list (where they would be relatively unlikely to be reclaimed) despite being marked accessed by the filesystem code. It was suggested that the internal `mark_page_accessed()` should move a page directly to the active list. The developers in the room deemed this problem to be the most important one to fix. 

There was some discussion of how the reclaim process flushes each page individually, causing a lot of costly inter-processor interrupts. Clearly, there needs to be some sort of batching applied to this particular operation. 

The topic of the shrinker interface returned during this session. Some developers feel that the direct reclaim process should not call shrinkers at all. Glauber Costa claimed that a lot of those calls are completely useless, all they can really do is to add latency to direct reclaim and worsen memory performance elsewhere. Calling shrinkers tends to free objects that are not useful as a solution to the specific shortage that put a process into direct reclaim in the first place. That just hurts the performance of the system as a whole. 

What would be nicer, Glauber said, would be to have a way to invoke a shrinker for the specific resource that is needed at the time. If a process is trying to allocate a directory entry (dentry), then the shrinker for the dentry cache should be called to free more dentries. Perhaps the slabs from which objects are allocated should have a pointer to the relevant shrinker added to them. Dave Chinner worried about potential locking problems that could result from calling shrinkers from the slab allocator, though. Mel worried that balancing problems could result if shrinkers are invoked only for specific data structures. 

Another problem with shrinkers is that many of them cannot be called in direct reclaim if the allocation in question has the `GFP_NOFS` flag set. That causes a lot of shrinker work to be deferred until some luckless process that can do full reclaim comes along; that process gets a lot of unrelated work dumped onto it. This, it was argued, should be the #2 priority among the problems to fix. 

Shrinkers also have an interesting property: they can be called concurrently, leading to locking issues. For these reasons and others, there was some agreement that it might be best to remove most shrinker calls from direct reclaim. Instead, a flag would be set that would cause the `kswapd` daemon to invoke the shrinkers from its own thread. A change along these lines can be expected sometime in the relatively near future. 

The final topic of discussion was a return to compaction. The compaction process works by identifying the pages that prevent the construction of a huge page and migrating them elsewhere. The problem, it seems, is that this procedure is a bit racy: other processes can slip in and steal the newly-freed pages before the targeted huge page can be created and allocated. That can result in the need to perform multiple passes over the page to actually free the whole thing — a less than desirable outcome. So, somehow, the pages that are freed for huge page construction need to be set aside somewhere where other processes cannot grab them. 

The end result of this session was a flip chart full of things to do and problems to fix. It will keep the memory management developers busy for a while — and the rest of us should benefit nicely.
