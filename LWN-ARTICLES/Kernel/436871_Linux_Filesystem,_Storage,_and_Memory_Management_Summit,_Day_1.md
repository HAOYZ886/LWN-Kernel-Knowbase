---
title: "Linux Filesystem, Storage, and Memory Management Summit, Day 1"
url: https://lwn.net/Articles/436871/
date: "April 5, 2011"
category: "Block layer; Filesystems-Workshops; Memory management-Conference sessions"
author: "By Jonathan Corbet April 5, 2011"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
April 5, 2011

It has been a mere eight months since the [2010 Linux Filesystem, Storage, and Memory Management Summit](<https://lwn.net/Articles/399148/>) was held in Boston, but that does not mean that there is not much to talk about - or that there has not been time to get a lot done. A number of items discussed at the 2010 event, including writeback improvements, better error handling, transparent huge pages, the I/O bandwidth controller, and the block-layer barrier rework, have been completed or substantially advanced in those eight months. Some other tasks remain undone, but there is hope: Trond Myklebust, in the introductory session of the 2011 summit, said that this might well be the last time that it will be necessary to discuss pNFS - a prospect which caused very little dismay. 

The following is a report from the first day of the 2011 meeting, held on April 4\. This coverage is necessarily incomplete; when the group split into multiple tracks, your editor followed the memory management discussions. 

#### Writeback

The Summit started with a plenary session to review the writeback problem. Writeback is the process of writing dirty pages back to persistent storage; it has been identified as one of the most significant performance issues with recent kernels. This session, led by Jan Kara, Jens Axboe, and Johannes Weiner, made it clear that a lot of thought is going into the issue, but that a full understanding of the problem has not yet been reached. 

One aspect of the problem that is well understood is that there are too many places in the kernel which are submitting writeback I/O. As a result, the different I/O streams conflict with each other and cause suboptimal I/O patterns, even if the individual streams are well organized - which is not always the case. So it would be useful to reduce the number of submission points, preferably to just one. 

Eliminating "direct reclaim," where processes which are allocating pages take responsibility for flushing other pages out to disk, is at the top of most lists. Direct reclaim cannot easily create large, well-ordered I/O, is computationally expensive, and leads to excessive lock contention. Some patches have been written in an attempt to improve direct reclaim by, for example, performing "write-around" of targeted pages to create larger contiguous operations, but none have passed muster for inclusion into the mainline. 

> [![\[Writeback
discussion\]](https://static.lwn.net/images/conf/2011/lfs-cs/writeback-sm.jpg)](<https://lwn.net/Articles/436977/>)

There was some talk of improving the I/O scheduler so that it could better merge and organize the I/O stream created by direct reclaim. One problem with that idea is that the request queue length is limited to 128 requests or so, which is not enough to perform reordering when there are multiple streams of heavy I/O. There were suggestions that the I/O scheduler might be considered broken and in need of fixing, but that view did not go very far. The problem with increasing the request queue length is that there would be a corresponding increase in I/O latencies, which is not a desirable result. Christoph Hellwig summed things up by saying that it was a mistake to generate bad I/O patterns in the higher levels of the system and expect the lower layers to fix them up, especially when it's relatively easy to create better I/O patterns in the first place. 

Many filesystems already try to improve things by generating larger writes than the kernel asks them to. Each filesystem has its own, specific hacks, though, and there is no generic solution to the problem. One thing that some filesystems could apparently do better has to do with their response to writeback on files where delayed allocation is being done. The kernel will often request writeback on a portion of the delayed allocation range; if the filesystem only completes allocation for the requested range, excessive fragmentation of the file may result. So, in response to writeback requests where the destination blocks have not yet been allocated, the filesystem should always allocate everything it can to create larger contiguous chunks. 

There are a couple of patches out there aimed at the goal of eliminating direct reclaim; both are based on the technique of blocking tasks which are dirtying pages until pages can be written back elsewhere in the system. The first of these was written by Jan; he was, he said, aiming at making the code as simple as possible. With this patch, a process which goes over its dirty page limit will be put on a wait queue. Occasionally the system will look at the I/O completions on each device and "distribute" those completions among the processes which are waiting on that device. Once a process has accumulated enough completions, it will be allowed to continue executing. Processes are also resumed if they go below their dirty limit for some other reason. 

The task of distributing completions runs every 100ms currently, leading to concerns that this patch could cause 100ms latencies for running processes. That could happen, but writeback problems can cause worse latencies now. Jan was also asked if control groups should be used for this purpose; his response was that he had considered the idea but put it aside because it added too much complexity at this time. There were also worries about properly distributing completions to processes; the code is inexact, but, as Chris Mason put it, getting more consistent results than current kernels is not a particularly hard target to hit. 

Evidently this patch set does not yet play entirely well with network filesystems; completions are harder to track and the result ends up being bursty. 

The alternative patch comes from Wu Fengguang. The core idea is the same, but it works by attempting to limit the dirtying of pages based on the amount of I/O bandwidth which is available for writeback. The bandwidth calculations are said to be complex to the point that mere memory management hackers have a hard time figuring out how it all works. When all else fails, and the system goes beyond the global dirty limit, processes will simply be put to sleep for 200ms at a time to allow the I/O subsystem to catch up. 

Mike Rubin complained that this patch would lead to unpredictable 200ms latencies any time that 20% of memory (the default global dirty limit) is dirtied. It was agreed that this is unfortunate, but it's no worse than what happens now. There was some talk of making the limit higher, but in the end that would change little; if pages are being dirtied faster than they can be written out, any limit will be hit eventually. Putting the limit too high, though, can lead to livelocks if the system becomes completely starved of memory. 

Another problem with this approach is that it's all based on the bandwidth of the underlying block device. If the I/O pattern changes - a process switches from sequential to random I/O, for example - the bandwidth estimates will be wrong and the results will not be optimal. The code also has no way of distinguishing between processes with different I/O patterns, with the result that those doing sequential I/O will be penalized in favor of other processes with worse I/O patterns. 

Given that, James Bottomley asked, should we try to limit I/O operations instead of bandwidth? The objection to that idea is that it requires better tracking of the ownership of pages; the control group mechanism can do that tracking, but it brings a level of complexity and overhead that is not pleasing to everybody. It was asserted that control groups are becoming more mandatory all the time, but that view has not yet won over the entire community. 

A rough comparison of the two approaches leads to the conclusion that Jan's patch can cause bursts of latency and occasional pauses. Fengguang's patch, instead, has smoother behavior but does not respond well to changing workloads; it also suffers from the complexity of the bandwidth estimation code. Beyond that, from the (limited) measurements which were presented, the two seem to have similar performance characteristics. 

What's the next step? Christoph suggested rebasing Fengguang's more complex patch on top of Jan's simpler version, merging the simple patch, and comparing from there. Before that comparison can be done, though, there need to be more benchmarks run on more types of storage devices. Ted Ts'o expressed concerns that the patches are insufficiently tested and might cause regressions on some devices. The community as a whole still lacks a solid idea of what benchmarks best demonstrate the writeback problems, so it's hard to say if the proposed solutions are really doing the job. That said, Ted liked the simpler patch for a reason that had not yet been mentioned: by doing away with direct reclaim, it at least gets rid of the [stack space exhaustion problem](<https://lwn.net/Articles/384093/>). Even in the absence of performance benefits, that would be a good reason to merge the code. 

But it would be good to measure the patch's performance effects, so better benchmarks are needed. Getting those benchmarks is easier said than done, though; at least some of them need to run on expensive, leading-edge hardware which must be updated every year. There are also many workloads to test, many of which are not easily available to the public. Mike Rubin did make a promise that Google would post at least three different benchmarks in the near future. 

There was some sympathy for the idea of merging Jan's patch, but Jan would like to see more testing done first. Some workloads will certainly regress, possibly as a result of performance hacks in specific filesystems. Andrew Morton said that there needs to be a plan for integration with the memory control group subsystem; he would also like a better analysis of what writeback problems are solved by the patch. In the near future, the code will be posted more widely for review and testing. 

#### VFS summary

James Bottomley started off the session on the virtual filesystem layer with a comment that, despite the fact that there were a number of "process issues" surrounding the merging of Nick Piggin's virtual filesystem scalability patches, this session was meant to be about technical issues only. Nick was the leader of the session, since the planned leader, Al Viro, was unable to attend due to health problems. As it turned out, Nick wanted to talk about process issues, so that is where much of the time was spent. 

The merging of the scalability work, he said, was not ideal. The patches had been around for a year, but there had not been enough review of them and he knew it at the time. Linus knew it too, but decided to merge that work anyway as a way of forcing others to look at it. This move worked, but at the cost of creating some pain for other VFS developers. 

Andrew Morton commented that the merging of the patches wasn't a huge problem; sometimes the only way forward is to "smash something in and fix it up afterward." The real problem, he said, was that Nick disappeared after the patches went in; he wasn't there to support the work. Developers really have to be available after merging that kind of change. If necessary, Andrew said, he is available to lean on a developer's manager if that's what it takes to make the time available. 

In any case, the code is in and mostly works. The autofs4 automounter is said to be "still sick," but that may not be entirely a result of the VFS work - a lot of automounter changes went in at the same time. 

An open issue is that lockless dentry lookup cannot happen on a directory which has extended attributes or when there is any sort of security module active. Nick acknowledged the problem, but had no immediate solution to offer; it is a matter of future work. Evidently supporting lockless lookup in such situations will require changes within the filesystems as well as at the VFS layer. 

A project that Nick seemed more keen to work on was adding per-node lists for data structures like dentries and inodes, enabling them to be reclaimed on a per-node basis. Currently, a memory shortage on one NUMA node can force the eviction of inodes and dentries on all nodes, even though memory may not be in short supply elsewhere in the system. Christoph Hellwig was unimpressed by the idea, suggesting that Nick should try it and he would see how badly it would work. Part of the problem, it seems, is that, for many filesystems, an inode cannot be reclaimed until all associated pages have been flushed to disk. Nick suggested that this was a problem in need of fixing, since it makes memory management harder. 

A related problem, according to Christoph, is that there is no coordination when it comes to the reclaim of various data structures. Inodes and dentries are closely tied, for example, but they are not reclaimed together, leading to less-than-optimal results. There were also suggestions that more of the reclaim logic for these data structures should be moved to the slab layer, which already has a fair amount of node-specific knowledge. 

#### Transcendent memory

Dan Magenheimer led a session on his [transcendent memory](<https://lwn.net/Articles/340080/>) work. The core idea behind transcendent memory - a type of memory which is only addressable on a page basis and which is not directly visible to the kernel - remains the same, but the uses have expanded. Transcendent memory has been tied to virtualization (and to Xen in particular) but, Dan says, he has been pushing it toward more general uses and has not written a line of Xen code in six months. 

So where has this work gone? It has resulted in "zcache," a mechanism for in-RAM memory compression which was merged into the staging tree for 2.6.39. There is increasing support for devices meant to extend the amount of available RAM - solid-state storage and phase-change memory devices, for example. The "ramster" module is a sort of peer-to-peer memory mechanism allowing pages to be moved around a cluster; systems which have free memory can host pages for systems which are under memory stress. And yes, transcendent memory can still be used to move RAM into and out of virtual machines. 

All of the above can be supported behind the [cleancache and frontswap patches](<https://lwn.net/Articles/386090/>), which Linus didn't get around to merging for 2.6.39-rc1. He has not yet said "no," but the chances of a 2.6.39 merge seem to be declining quickly. 

Hugh Dickins voiced a concern that frontswap is likely to be filled relatively early in a system's lifetime, and what will end up there is application initialization code which is unlikely to ever be needed again. What is being done to avoid filling the frontswap area with useless stuff? Dan acknowledged that it could be a problem; one solution is to have a daemon which would fetch pages back out of frontswap when memory pressure is light. Hugh worried that, in the long term, the kernel was going to need new data structures to track pages as they are put into hierarchical swap systems and that frontswap, which lacks that tracking, may be a step in the wrong direction. 

Andrea Arcangeli said that a feature like zcache could well be useful for guests running under KVM as well. Dan agreed that it would be nice, but that he (an Oracle employee) could not actually do the implementation. 

#### How memory control groups are used

This session was a forum in which representatives of various companies could talk about how they are making use of the [memory controller](<https://lwn.net/Articles/243795/>) control groups functionality. It turns out that this feature is of interest to a wide range of users. 

Ying Han gave the surprising news that Google has a lot of machines to manage, but also a lot of jobs to run. So the company is always trying to improve the utilization of those machines, and memory utilization in particular. Lots of jobs tend to be packed into each machine, but that leads to interference; there simply is not enough isolation between tasks. Traditionally, Google has used a "fake NUMA" system to partition memory between groups of tasks, but that has some problems. Fake NUMA suffers from internal fragmentation, wasting a significant amount of memory. And Google is forced to carry a long list of fake NUMA patches which are not ever likely to make it upstream. 

So Google would rather make more use of memory control groups, which are upstream and which, thus, will not be something the company has to maintain on its own indefinitely. Much work has gone upstream, but there are still unmet needs. At the top of the list is better accounting of kernel allocations; the memory controller currently only tracks memory allocations made from user space. There is also a need for "soft limits" which would be more accommodating of bursty application behavior; this topic was discussed in more detail later on. 

Hiroyuki Kamezawa of Fujitsu said that his employer deals with two classes of customers in particular: those working in the high-performance computing area, and government. The memory controller is useful in both situations, but it has one big problem: performance is not what they really need it to be. Things get especially bad when control groups begin to push up against the limits. So his work is mostly focused on improving the performance of memory control groups. 

Larry Woodman of Red Hat, instead, deals mostly with customers in the financial industry. These customers are running trading systems with tight deadlines, but they also need to run backups at regular intervals during the day. The memory controller allows these backups to be run in a small, constrained group which enables them to make forward progress without impeding the trading traffic. 

Michal Hocko of SUSE deals with a different set of customers, working in an area which was not well specified. These customers have large datasets which take some time to compute, and which really need to be kept in RAM if at all possible. Memory control groups allow those customers to protect that memory from reclaim. They work like a sort of "clever `mlock()`" which keeps important memory around most of the time, but which does not get in the way of memory overcommit. 

Pavel Emelyanov of Parallels works with customers operating as virtual hosting Internet service providers. They want to be able to create containers with a bounded amount of RAM to sell to customers; memory control groups enable that bounding. They also protect the system against memory exhaustion denial-of-service attacks, which, it seems, are a real problem in that sector. He would like to see more graceful behavior when memory runs out in a specific control group; it should be signaled as a failure in a `fork()` or `malloc()` call rather than a segmentation fault signal at page fault time. He would also like to see better accounting of memory use to help with the provisioning of containers. 

David Hansen of IBM talked about customers who are using memory control groups with KVM to keep guests from overrunning their memory allocations. Control groups are nice because they apply limits while still allowing the guests to use their memory as they see fit. One interesting application is in cash registers; these devices, it seems, run Linux with an emulation layer that can run DOS applications. Memory control groups are useful for constraining the memory use of these applications. Without this control, these applications can grow until the OOM killer comes into play; the OOM killer invariably kills the wrong process (from the customer's point of view), leading to the filing of bug reports. The value of the memory controller is not just that it constrains memory use - it also limits the number of bug reports that he has to deal with. 

Coly Li, representing Taobao, talked briefly about that company's use of memory control groups. His main wishlist item was the ability to limit memory use based on the device which is providing backing store. 

#### What's next for the memory controller

The session on future directions for the memory controller featured contributors who were both non-native English speakers and quite soft-spoken, so your editor's notes are, unfortunately, incomplete. 

One topic which came up repeatedly was the duplication of the least-recently used (LRU) list. The core VM subsystem maintains LRU lists in an attempt to track which pages have gone unused for the longest time and which, thus, are unlikely to be needed in the near future. The memory controller maintains its own LRU list for each control group, leading to a wasteful duplication of effort. There is a strong desire to fix this problem by getting rid of the global LRU list and performing all memory management with per-control-group lists. This topic was to come back later in the day. 

Hiroyuki Kamezawa complained that the memory controller currently tracks the sum of RAM and swap usage. There could be value, he said, in splitting swap usage out of the memory controller and managing it separately. 

Management of kernel memory came up again. It was agreed that this was a hard problem, but there are reasons to take it on. The first step, though, should be simple accounting of kernel memory usage; the application of limits can come later. Pavel noted that it will never be possible to track all kernel memory usage, though; some allocations can never be easily tied to specific control groups. Memory allocated in interrupt handlers is one example. It also is often undesirable to fail kernel allocations even when a control group is over its limits; the cost would simply be too high. Perhaps it would be better, he said, to focus on specific problem sources. Page tables, it seems, are a data structure which can soak up a lot of memory and a place where applying limits might make sense. 

The way shared pages are accounted for was discussed for a bit. Currently, the first control group to touch a page gets charged for it; all subsequent users get the page for free. So if one control group pages in the entire C library, it will find its subsequent memory use limited while everybody else gets a free ride. In practice, though, this behavior does not seem to be a problem; a control group which is carrying too much shared data will see some of it reclaimed, at which point other users will pick up the cost. Over time, the charging for shared pages is distributed throughout the system, so there does not seem to be a need for a more sophisticated mechanism for accounting for them. 

#### Local filesystems in the cloud

Mike Rubin of Google ran a plenary session on the special demands that cloud computing puts on filesystems. Unfortunately, the notes on this talk are also incomplete due to a schedule misunderstanding. 

What cloud users need from a filesystem is predictable performance, the ability to share systems, and visibility into how the filesystem works. Visibility seems to be the biggest problem; it is hard, he said, to figure out why even a single machine is running slowly. Trying to track down problems in an environment consisting of thousands of machines is a huge problem. 

Part of that problem is just understanding a filesystem's resource requirements. How much memory does an ext4 filesystem really need? That number turns out to be 2MB of RAM for every terabyte of disk space managed \- a number which nobody had been able to provide. Just as important is the metadata overhead - how much of a disk's bandwidth will be consumed by filesystem metadata? In the past, Google has been surprised when adding larger disks to a box has caused the whole system to fall over; understanding the filesystem's resource demands is important to prevent such things from happening in the future. 

Tracing, he said, is important - he does not know how people ever lived without it. But there need to be better ways of exporting the information; there is a lack of user-space tools which can integrate the data from a large number of systems. Ted Ts'o added that the "blktrace" tool is fine for a single system where root access is available. In a situation where there are hundreds or thousands of machines, and where developers may not have root access on production systems, blktrace does not do the job. There needs to be a way to get detailed, aggregated tracing information - with file name information - without root access. 

Michael said that he is happy that Google's storage group has been upstreaming almost everything they have done. But, he said, the group has a "diskmon" tool which still needs to see the light of day. It can create histograms of activity and latencies at all levels of the I/O stack, showing how long each operation took and how much of that time was consumed by metadata. It is all tied to a web dashboard which can highlight problems down to the ID of the process which is having trouble. This tool is useful, but it is not yet complete. What we really need, he said, is to have that kind of visibility designed into kernel subsystems from the outset. 

Michael concluded by saying that, in the beginning, his group was nervous about engaging with the development community. Now, though, they feel that the more they do it, the better it gets. 

#### Memory controller targeted reclaim

Back in the memory management track, Ying Han led a session on improving reclaim within the memory controller. When things get tight, she said, the kernel starts reclaiming from the global LRU list, grabbing whatever pages it finds. It would be far better to reclaim pages specifically from the control groups which are causing the problem, limiting the impact on the rest of the system. 

One technique Google uses is soft memory limits in the memory controller. Hard limits place an absolute upper bound on the amount of memory any group can use. Soft limits, instead, can be exceeded, but only as long as the system as a whole is not suffering from memory contention. Once memory gets tight at the global level, the soft limits are enforced; that automatically directs reclaim at the groups which are most likely to be causing the global stress. 

Adding per-group background reclaim, which would slowly clean up pages in the background, would help the situation, she said. But the biggest problem is the global LRU list. Getting rid of that list would eliminate contention on the per-zone LRU lock, which is a problem, but, more importantly, it would improve isolation between groups. Johannes Weiner worried that eliminating the global LRU list would deprive the kernel of its global view of memory, making zone balancing harder; Rik van Riel responded that we are able to balance pages between zones using per-zone LRU lists now; we should, he said, be able to do the same thing with control groups. 

The soft limits feature can help with global balancing. There is a problem, though, in that configuring those limits is not an easy task. The proper limits depend on the total load on the system, which can change over time; getting them right will not be easy. 

Andrea Arcangeli made the point that whatever is done with the global LRU list cannot be allowed to hurt performance on systems where control groups are configured out. The logic needs to transparently fall back to something resembling the current implementation. In practice, that fallback is likely to take the form of a "global control group" which contains all processes which are not part of any other group. If control groups are not enabled, the global group would be the only one in existence. 

#### Shrinking struct page_cgroup

The system memory map contains one `struct page` for every page in the system. That's a lot of structures, so it's not surprising that `struct page` is, perhaps, the most tightly designed structure in the kernel. Every bit has been placed into service, usually in multiple ways. The memory controller has its own per-page information requirements; rather than growing `struct page`, the memory controller developers created a separate `struct page_cgroup` instead. That structure looks like this: 

```
struct page_cgroup {
    	unsigned long flags;
    	struct mem_cgroup *mem_cgroup;
    	struct page *page;
    	struct list_head lru;
        };
```

The existence of one of these structures for every page in the system is part of why enabling the memory controller is expensive. But Johannes Weiner thinks that he can reduce that overhead considerably - perhaps to zero. 

Like the others, Johannes would like to get rid of the duplicated LRU lists; that would allow the `lru` field to be removed from this structure. It should also be possible to remove the `struct page` backpointer by using a single LRU list as well. The `struct mem_cgroup` pointer, he thinks, is excessive; there will usually be a bunch of pages from a single file used in any given control group. So what is really needed is a separate structure to map from a control group to the `address_space` structure representing the backing store for a set of pages. Ideally, he would point to that structure (instead of `struct address_space`) in `struct page`, but that would require some filesystem API changes. 

The final problem is getting rid of the `flags` field. Some of the flags used in this structure, Johannes thinks, can simply be eliminated. The rest could be moved into `struct page`, but there is little room for more flags there on 32-bit systems. How that problem will be resolved is not yet entirely clear. One way or the other, though, it seems that most or all of the memory overhead associated with the memory controller can be eliminated with some careful programming. 

#### Memory compaction

Mel Gorman talked briefly about the current state of the [memory compaction](<https://lwn.net/Articles/368869/>) code, which is charged with the task of moving pages around to create larger, physically-contiguous ranges of free pages. The rate of change in this code, he said, has reduced "somewhat." Initially, the compaction code was relatively [![\[Mel Gorman\]](https://static.lwn.net/images/conf/2011/lfs-cs/MelGorman-sm.jpg)](<https://lwn.net/Articles/436978/>) primitive; it only had one user (hugetlbfs) to be concerned about. Since then, the [lumpy reclaim](<https://lwn.net/Articles/211505/>) code has been mostly pushed out of the kernel, and [transparent huge pages](<https://lwn.net/Articles/359158/>) have greatly increased the demands on the compaction code. 

Most of the problems with compaction have been fixed. The last was one in which interrupts could be disabled for long periods - up to about a half second, a situation which Mel described as "bad." He also noted that it was distressing to see how long it took to find the bug, even with tools like ftrace available. There are more interrupt-disabling problems in the kernel, he said, especially in the graphics drivers. 

One remaining problem with compaction is that pages are removed from the LRU list while they are being migrated to their new location; then they are put back at the head of the list. As a result, the kernel forgets what it knew about how recently the page has actually been used; pages which should have been reclaimed can live on as a result of compaction. A potential fix, suggested by Minchan Kim, is to remember which pages were on either side of the moved page in the LRU list; after migration, if those two pages are still together on the LRU, it probably makes sense to reinsert the moved page between them. Mel asked for comments on this approach. 

Rik van Riel noted that, when transparent huge pages are used, the chances of physically-contiguous pages appearing next to each other in the LRU list are quite high; splitting a huge page will create a whole set of contiguous pages. In that situation, compaction is likely to migrate several contiguous pages together; that would break Minchan's heuristic. So Mel is going to investigate a different approach: putting the destination page into the LRU in the original page's place while migration is underway. There are some issues that need to be resolved - what happens if the destination page falls off the LRU and is reclaimed during migration, for example - but that approach might be workable. 

Mel also talked briefly about some experiments he ran writing large trees to slow USB-mounted filesystems. Things have gotten better in this area, but the sad fact is that generating lots of dirty pages which must be written back to a USB stick can still stall the system for a long time. He was surprised to learn that the type of filesystem used on the device makes a big difference; VFAT is very slow, ext3 is better, and ext4 is better yet. What, he asked, is going on? 

There was a fair amount of speculation without a lot of hard conclusions. Part of the problem is probably that the filesystem (ext3, in particular) will end up blocking processes which are waiting on buffers until a big journal commit frees some buffers. That can cause writes to a slow device to stall unrelated processes. It seems that there is more going on, though, and the problem is not yet solved. 

#### Per-CPU variables

Christoph Lameter and Tejun Heo discussed per-CPU data. For the most part, the session was a beginner-level introduction to this feature and its reason for existence; see [this article](<https://lwn.net/Articles/258238/>) if a refresher is needed. There was some talk about future applications of per-CPU variables; Christoph thinks that there is a lot of potential for improving scalability in the VFS layer in particular. Further in the future, it might make sense to confine certain variables to specific CPUs, which would then essentially function as servers for the rest of the kernel; LRU scanning was one function which could maybe be implemented in this way. 

There was some side talk about the limitations placed on per-CPU variables on 32-bit systems. Those limits exist, but 32-bit systems also create a number of other, more severe limits. It was agreed that the limit to scalability with 32 bits was somewhere between eight and 32 CPUs. 

#### Lightning talks

The final session of the day was a small set of lightning talks. Your editor will plead "incomplete notes" one last time; perhaps the long day and the prospect of beer caused a bit of inattention. 

David Howells talked about creating a common infrastructure for the handling of keys in network filesystems. Currently most of these filesystems handle keys for access control, but they all have their own mechanisms. Centralizing this code could simplify a lot of things. He would also like to create a common layer for the mapping of user IDs while he is at it. 

David also talked about a scheme for the attachment of attributes to directories at any level of a network filesystem. These attributes would control behavior like caching policies. There were questions as to why the existing extended attribute mechanism could not be used; it came down to a desire to control policy on the client side when root access to the server might not be available. 

Matthew Wilcox introduced the "NVM Express" standard to the group. This standard describes the behavior of solid-state devices connected via PCI-Express. The standard was released on March 1; a Linux driver, he noted with some pride, was shipped on the same day. The Windows driver is said to be due within 6-9 months; actual hardware can be expected within about a year. 

The standard seems to be reasonably well thought out; it provides for all of the functionality one might expect on these devices. It allows devices to implement multiple "namespaces" - essentially separate logical units covering parts of the available space. There are bits for describing the expected access patterns, and a "this data is already compressed so don't bother trying to compress it yourself" bit. There is a queued "trim" command which, with luck, won't destroy performance when it is used. 

How the actual hardware will behave remains to be seen; Matthew disappointed the audience with his failure to have devices to hand out. 

#### Day 2

See [this page](<https://lwn.net/Articles/437066/>) for reporting from the second day of the summit.
