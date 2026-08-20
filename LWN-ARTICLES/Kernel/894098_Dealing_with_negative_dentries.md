---
title: Dealing with negative dentries
url: https://lwn.net/Articles/894098/
date: "May 9, 2022"
category: Dentry cache
author: "By Jake Edge May 9, 2022 LSFMM"
---

By **Jake Edge**  
May 9, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

The problem of negative dentries accumulating in the dentry cache in an unbounded manner, as we [looked at](<https://lwn.net/Articles/890025/>) back in April, came up at the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM). Negative dentries reflect failed file-name lookups, which are then cached, saving an expensive operation if the file name in question is looked up again. There is no mechanism to proactively prune back those cache entries, however, so the cache keeps growing until memory pressure finally causes the system to forcibly evict some of them, which can make the system unresponsive for a long time or even cause a soft lockup. 

#### The problem

[ ![\[Stephen Brennan\]](https://static.lwn.net/images/2022/lsfmm-brennan-sm.png) ](<https://lwn.net/Articles/894380/>)

Stephen Brennan led the session; he had posted a [patch set](<https://lwn.net/ml/linux-kernel/20220331190827.48241-1-stephen.s.brennan@oracle.com/>), with a new approach to the problem, during the discussion in March. The problem that he is seeing is on big servers with lots of memory, where part of the workload is looking up unique IDs in some directory a few times per second—which goes on for months or years. Each lookup creates a negative dentry in the cache, resulting in a cache full of these entries that have never been used after they were created. 

Since there is no memory pressure, because the system has lots more memory than is needed by the workload, there is no clean up. That can lead to soft lockups when iterating through the children of a dentry because of the amount of time it takes to do so. It also leads to slab fragmentation; if the system has 500 million negative dentries and then a directory containing some of them is deleted, there will be an enormous number of partially filled slab pages. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

His goal is to have some kind of generic system for managing all of the various least-recently-used (LRU) lists in the page-cache and filesystem code. Currently, negative dentries are not moved to the head of the LRU when they are referenced; instead, they are simply marked and left in place until the shrinker runs. [Shrinkers](<https://lwn.net/Articles/550463/>) are the mechanism that the memory-management subsystem uses to request cache entries be freed. They only run when there is memory pressure, but at that point a dentry might have been marked as referenced a year ago, so that dentry is not useful anymore—if it ever was. When that happens, the shrinkers have to do a lot of work just to move entries to the end of the list before they can even be reclaimed. 

In the mailing list discussion, Dave Chinner [wanted to see](<https://lwn.net/ml/linux-mm/20220322222114.GE1609613@dread.disaster.area/>) a generic solution for various caches that all use the `list_lru` mechanism. Brennan (and others) have been thinking about that. For example, every time something gets added to these caches, the list could be rotated to move those with fewer references to the back of the list. At that point, perhaps some aging rules could be applied to negative dentries. Another thing that could be done is to track the number of entries in these caches, and their types, so that decisions could be made based on the numbers of dentries, negative dentries, or some calculation using those numbers. 

James Bottomley said that the problem is that the number of negative dentries is completely unbounded, so he wondered if the focus of the work should be on dealing with that. The number of positive dentries is bounded by the number of files in a directory, but there is a nearly infinite number of file names that are _not_ present. Brennan agreed, noting that on the systems he has looked at, 99% of the cached dentries are of the negative variety, so that is the source of the underlying problem. But he thinks the fix can be made using the existing `lru_list` and shrinker frameworks. 

While Oracle's main concern is with negative dentries, Matthew Wilcox said, there are other caches that have similar problems. For example, under certain workloads, the inode cache can similarly end up with many entries that will never be used again. That cache is bounded by the number of files in the filesystem, but it is still a "used once" problem as with negative dentries in the problem cases. 

#### Not really LRU

This is a "classic LRU problem", Kent Overstreet said. Brennan agreed with that, noting that these LRUs are not really being treated as "least recently used". There are, instead, used-once entries scattered throughout the list. Under memory pressure, those entries are the ones that should be cleaned up; if they were organized better, so that all of those that were not recently used were together, the cost of scanning the whole list for them could be reduced. But there is still the problem of workloads that create "stupid amounts" of negative dentries that will never be used again. There is no good reason to cache those at all. 

Josef Bacik said that he had solved a similar problem around five years ago by not marking entries as referenced until they are used for the second time. Prior to that, the list was being scanned to clear the referenced bits for entries that had been referenced once, but that scan took a long time. He chose to wait for the second use, instead of changing the LRU to be a real LRU, because he found that constantly shuffling things to the back of the list was "not excellent" for the workload he was looking at. 

Brennan said that it is not excellent for a lot of workloads because it leads to a lot of contention on the spinlocks. He loves the idea of waiting until the entry is used again, but it leads to another problem: "how much is too much?" Memory pressure provides a good signal to indicate that entries need to be pruned, but the soft lockups he mentioned can occur starting around 100 million negative dentries in a single directory. He would rather not use some kind of "magic threshold", but there needs to be some mechanism to start shrinking the one-use items, at least. 

#### Other possibilities

Bottomley suggested that the bounded positive dentry cache size for a directory be the limit on how large the negative-dentry cache could grow. Brennan thought that was an interesting idea. Ted Ts'o pointed out that negative dentries have no references to other data structures in the system, so they are easy to get rid of, or simply to move to another page. He wondered if simply getting rid of the negative dentries blocking the freeing of a page might be a reasonable tradeoff, even if those entries might actually be used again. There may be good reasons to give negative dentries special treatment, he said. 

Bacik said that the negative-dentry problem can be solved in a fairly straightforward way, but that if Brennan wanted to solve the more general problem, there was a lot of work that would be needed. There was some discussion of using a mechanism like the [page cache "refault" tracking](<https://lwn.net/Articles/495543/>), that was added by Johannes Weiner quite a ways back; it would allow the system to know that something it had evicted returned to the cache, so it should probably stick around longer. 

Overstreet wondered if there was a way to detect workloads that are scanning and creating lots of negative dentries, then stopping the creation of those entries. That could be tracked per process ID and limits could be placed on how many negative dentries could be created. Michal Hocko said it sounded like a similar problem to that of throttling the creation of dirty pages; when the rate of their creation gets too high, a process is throttled to slow down their creation. 

Brennan said that the problem is not necessarily that the entries are created at a high rate of hundreds or thousands per second; it could be a slow trickle of them over a long period of time. It can add up to hundreds of millions over, say, a year's time. There are some workloads that simply do lookups, which create a negative dentry when the file is not found, but others may be creating lots of temporary files and then deleting them, which also leaves negative dentries behind. 

John Hubbard said that part of the problem is that Linux wants to use all of the available memory if possible, because it generally leads to better performance, but there is a cost to getting that memory back when it is needed elsewhere. Making any kind of improvement on the reclaim side would really help, Brennan said. 

As time for the session wound down, Bacik asked Brennan what it is he would like the filesystems and memory-management developers to do to help. Brennan said that there were a lot of good ideas raised in the session and he hopes that those who care about the specific problem for negative dentries and the more general problem of finding better ways to reclaim memory from these caches would provide more eyeballs on the patches he would be posting. 

Weiner asked if a way to sidestep the problem would be to put the negative dentries on a separate shrinker list. There are some shrinkers that are called more aggressively and there is no need to protect specific entries; a more wholesale freeing would probably work just fine. Brennan agreed that might be a way forward. The possibility of making changes that were specific to negative dentries was his favorite part of the discussion, he said.
