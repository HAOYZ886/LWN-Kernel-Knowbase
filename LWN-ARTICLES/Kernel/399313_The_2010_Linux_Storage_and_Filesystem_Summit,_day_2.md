---
title: "The 2010 Linux Storage and Filesystem Summit, day 2"
url: https://lwn.net/Articles/399313/
date: "August 10, 2010"
category: "Block layer; Filesystems-Workshops"
author: "By Jonathan Corbet August 10, 2010"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
August 10, 2010

The second day of the 2010 Linux Storage and Filesystem Summit was held on August 9 in Boston. Those who have not yet read [the coverage from day 1](<http://lwn.net/Articles/399148/>) may want to start there. This day's topics were, in general, more detailed and technical and less amenable to summarization here. Nonetheless, your editor will try his best. 

#### Writeback

The first session of the day was dedicated to the writeback issue. Writeback, of course, is the process of writing modified pages of files back to persistent store. There have been numerous complaints over recent years that writeback performance in Linux has regressed; the curious reader can refer to [this article](<http://lwn.net/Articles/396561/>) for some details, or [this bugzilla entry](<https://bugzilla.kernel.org/show_bug.cgi?id=12309>) for many, many details. The discussion was less focused on this specific problem, though; instead, the developers considered the problems with writeback as a whole. 

Sorin Faibish started with a discussion of some research that he has done in this area. The challenges for writeback are familiar to those who have been watching the industry; the size of our systems - in terms of both memory and storage - has increased, but speed of those systems has not increased proportionally. As a result, writing back a given percentage of a system's pages takes longer than it once did. It is always easier for the writeback system to fail to keep up with processes which are dirtying pages, leading to poor performance. 

His assertion is that the use of watermarks to control writeback is no longer appropriate for contemporary systems. Writeback should not wait until a certain percentage of memory is dirty; it should start sooner, and, crucially, be tied to the rate with which processes are dirtying pages. The system, he says, should work much more aggressively to ensure that the writeback rate matches the dirty rate. 

From there, the discussion wandered through a number of specific issues. Linux writeback now works by flushing out pages belonging to a specific file (inode) at a time, with the hope that those pages will be located nearby on the disk. The memory management code will normally ask the filesystem to flush out up to 4MB of data for each inode. One poorly-kept secret of Linux memory management is that filesystems routinely ignore that request - they typically flush far more data than requested if there are that many dirty pages. It's only by generating much larger I/O requests that they can get the best performance. 

Ted Ts'o wondered if blindly increasing writeback size is the best thing to do. 4MB is clearly too small for most drives, but it may well be too large for a filesystem located on a slow USB drive. Flushing large amounts of data to such a filesystem can stall any other I/O to that device for quite some time. From this discussion came the idea that writeback should not be based on specific amounts of data, but, instead, should be time-based. Essentially, the backing device should be time-shared between competing interests in a way similar to how the CPU is shared. 

James Bottomley asked if this idea made sense - is it ever right to cut off I/O to an inode which still has contiguous, dirty pages to write? The answer seems to be "yes." Consider a process which is copying a large file - a DVD image or something even larger. Writeback might not catch up with such a process until the copy is done, which may not be for a long time into the future; meanwhile, all other users of that device will be starved. That is bad for interactivity, and it can cause long delays before other files are flushed to disk. Also, the incremental performance benefit of extending large I/O operations tend to drop off over time. So, in the end, it's necessary to switch to another inode at some point, and making the change based on wall-clock time seems to be the most promising approach. 

Boaz Harrosh raised the idea of moving the I/O scheduler's intelligence up to the virtual memory management level. Then, perhaps, application priorities could be used to give interactive processes privileged access to I/O bandwidth. Ted, instead, suggested that there may be value in allowing the assignment of priorities to individual file descriptors. It's fairly common for an application to have files it really cares about, and others (log files, say) which matter less. The problem with all of these ideas, according to Christoph Hellwig, is that the kernel has far too many I/O submission paths. The block layer is the only place where all of those I/O operations come together into a single place, so it's the only place where any sort of reasonable I/O control can be applied. A lot of fancy schemes are hard to implement at that level, so, even if descriptor-based priorities are a good idea (not everybody was convinced), it's not something that can readily be done now. Unifying the I/O submission paths was seen as a good idea, but it's not something for the near future. 

Jan Kara asked about how results can be measured, and against what requirements will they be judged? Without that information, it is hard to know if any changes have had good effects or not. There are trivial cases, of course - changes which slow down kernel compiles tend to be caught early on. But, in general, we have no way to measure how well we are doing with writeback. So, in the end, the first action item is likely to be an attempt to set down the requirements and to develop some good test cases. Once it's possible to decide whether patches make sense, there will probably an implementation of some sort of time-based writeback mechanism. 

#### Solid-state storage devices

There were two sessions on solid-state storage devices (SSDs) at the summit; your editor was able to attend only the first. The situation which was described there is one we have been hearing about for a couple of years at least. These devices are getting faster: they are heading toward a point where they can perform one million I/O operations per second. That said, they still exhibit significant latency on operations (though much less than rotating drives do), so the only way to get that kind of operation count is to run a lot of operations in parallel. "A lot" in this case means having something like 100 operations in flight at any given time. 

Current SSDs work reasonably well with Linux, but there are certainly some problems. There is far too much overhead in the ATA and SCSI layers; at that kind of operation rate, microseconds hurt. The block layer's request queues are becoming a bottleneck; it's currently only possible to have about 32 concurrent operations outstanding on a device. The system needs to be able to distribute I/O completion work across multiple CPUs, preferably using smart controllers which can direct each completion interrupt to the CPU which initiated a specific operation in the first place. 

For "storage-attached" SSDs (those which look like traditional disks), there are not a lot of problems at the filesystem level; things work pretty well. Once one gets into bus-attached devices which do not look like disks, though, the situation changes. One participant asserted that, on such devices, the ext4 filesystem could not be expected to get reasonable performance without a significant redesign. There is just too much to do in parallel. 

Ric Wheeler questioned the claim that SSDs are bringing a new challenge for the storage subsystem. Very high-end enterprise storage arrays have achieved this kind of I/O rate for some years now. One thing those arrays do is present multiple devices to the system, naturally helping with parallelism; perhaps SSDs could be logically partitioned in the same way. 

#### Resizing guest memory

A change of pace was had in the memory management track, where Rik van Riel talked about the challenges involved in resizing the memory available to virtualized guests. There are four different techniques in use currently: 

  * Memory hotplug by way of simulated hardware hotplug events. This mechanism works well for adding memory to guests, but it cannot really be used to take memory back. Hot remove simply does not work well, because there's always some sort of non-movable allocation which ends up in the space which would be removed. 

  * Ballooning, wherein a special driver in the guest allocates pages and retires them from use, essentially handing them back to the host. Memory can be fed back into the guest by having the balloon driver free the pages it has allocated. This mechanism is simple, if somewhat slow, but simple management policies are scarce. 

  * Transcendent memory techniques like [cleancache and frontswap](<http://lwn.net/Articles/386090/>), which can be used to adjust memory availability between virtual guests. 

  * Page hinting, whereby guests mark pages which can be discarded by the host. These pages may be on the guest's free list, or they may simply be clean pages. Should the guest try to access such a page after the host has thrown it away, that guest will receive a special page fault telling it that it needs to allocate the page anew. Hinting techniques tend to bring a lot of complexity with them. 

The real question of interest in this session seemed to be the "rightsizing" of guests - giving each guest just enough memory to optimize the performance of the system as a whole. Google is also interested in this problem, though it is using cgroup-based containers instead of full virtualization. It comes down to figuring out what a process's minimal working set size is - a problem which has resisted attempts at solution for decades. 

Mel Gorman proposed one approach to determine a guest's working set size. Place that guest under memory pressure, slowly shrinking its available memory over time. There will come a point where the kernel starts scanning for reclaimable pages, and, as the pressure grows, a point where the process starts paging in pages which it had previously used. That latter point could be deemed to be the place where the available memory had fallen below the working set size. It was also suggested that page reactivations \- when pages are recovered from the inactive list and placed back into active use - could also be the metric by which the optimal size is determined. 

Nick Piggin was skeptical of such schemes, though. He gave the example of two processes, one of which is repeatedly working through a 1GB file, while the other is working through a 1TB file. If both processes currently have 512MB of memory available, they will both be doing significant amounts of paging. Adjusting the memory size will not change that behavior, leading to the conclusion that there's not much to be done - until the process with the smaller file gets 1GB of memory to work with. At that point, its paging will stop. The process working with the larger file will never reach that point, though, at least on contemporary systems. So, even though both processes are paging at the same rate, the initial 512MB memory size is too small for one process, but is just fine for the other. 

The fact that the problem is hard has not stopped developers from trying to improve the situation, though, so we are likely to see attempts made at dynamically resizing guests in an attempt to work out their optimal sizes. 

#### I/O bandwidth controllers

Vivek Goyal led a brief session on the I/O bandwidth controller problem. Part of that problem has been solved - there is now a proportional-weight bandwidth controller in the mainline kernel. This controller works well for single-spindle drives, perhaps a bit less so with large arrays. With larger systems, the single dispatch queue in the CFQ scheduler becomes a bottleneck. Vivek has been working on [a set of patches](<http://lwn.net/Articles/396530/>) to improve that situation for a little while now. 

The real challenge, though, is the desired maximum bandwidth controller. The proportional controller which is there now will happily let a process consume massive amounts of bandwidth in the absence of contention. In most cases, that's the right result, but there are hosting providers out there who want to be able to keep their customers within the bandwidth limits they have paid for. The problem here is figuring out where to implement this feature. Doing it at the I/O scheduler level doesn't work well when there are devices stacked higher in the chain. 

One suggestion is to create a special device mapper target which would do maximum bandwidth throttling. There was some resistance to that idea, partly because some people would rather avoid the device mapper altogether, but also due to practical problems like the inability of current Linux kernels to insert a DM-based controller into the stack for an already-mounted disk. So we may see an attempt to add this feature at the request queue level, or we may see a new hook allowing a block I/O stream to be rerouted through a new module on the fly. 

The other feature which is high on the list is support for controlling buffered I/O bandwidth. Buffered I/O is hard; by the time an I/O request has made it to the block subsystem, it has been effectively detached from the originating process. Getting around that requires adding some new page-level accounting, which is not a lightweight solution. 

#### Reclaim topics

Back in the memory management track, a number of reclaim-oriented topics were covered briefly. The first of these is per-cgroup reclaim. Control groups can be used now to limit total memory use, so reclaim of anonymous and page-cache pages works just fine. What is missing, though, is the sort of lower-level reclaim used by the kernel to recover memory: shrinking of slab caches, trimming the inode cache, etc. A cgroup can consume considerable resources with this kind of structure, and there is currently no mechanism for putting a lid on such usage. 

Zone-based reclaim would also be nice; that is evidently covered in the VFS scalability patch set, and may be pushed toward the mainline as a standalone patch. 

Reclaim of smaller structures is a problem which came up a few times this afternoon. These structures are reclaimed individually, but the virtual memory subsystem is really only concerned with the reclaim of full pages. So reclaiming individual inodes (or dentries, or whatever) may just serve to lose useful cached information and increase fragmentation without actually freeing any memory for the rest of the system. So it might be nice to change the reclaim of structures like dentries to be more page-focused, so that useful chunks of memory can be returned to the system. 

The ability to move these structures around in memory, freeing pages through defragmentation, would also be useful. That is a hard problem, though, which will not be amenable to a quick solution. 

There is an interesting problem with inode reclaim: cleaning up an inode also clears all related page cache pages out of the system. There can be times when that's not what's really called for. It can free vast amounts of memory when only small amounts are needed, and it can deprive the system of cached data which will just need to be read in again in the near future. So there may be an attempt to change how inode reclaim works sometime soon. 

There are some difficulties with how the page allocator works on larger systems; free memory can go well below the low watermark before the system notices. That is the result of how the per-CPU queues work; as the number of processors grows, the accounting of the size of those queues gets fuzzier. So there was talk of sending inter-processor interrupts on occasion to get a better count, but that is a very expensive solution. Better, perhaps, is just to iterate over the per-CPU data structures and take the locking overhead. 

#### Slab allocators

Christoph Lameter ran a discussion on slab allocators, talking about the three allocators which are currently in the kernel and the attempts which are being made to unify them. This is a contentious topic, but there was a relative lack of contentious people in the room, so the discussion was subdued. What happens will really depend on what patches Christoph posts in the future. 

#### O_DIRECT

A brief session touched on a few problems associated with direct I/O. The first of these is an obscure race between `get_user_pages()` (which pins user-space pages in memory so they can be used for I/O) and the `fork()` system call. In some cases, a `fork()` while the pages are mapped can corrupt the system. A number of fixes have been posted, but they have not gotten past Linus. The real fix will involve fixing all `get_user_pages()` callers and (the real point of contention) slowing down `fork()`. The race is a real problem, so some sort of solution will need to find its way into the mainline. 

Why, it was asked, do applications use direct I/O instead of just mapping file pages into their address space? The answer is that these applications know what they want to do with the hardware and do not want the virtual memory system getting in the way. This is generally seen as a valid requirement. 

There is some desire for the ability to do direct I/O from the virtual memory subsystem itself. This feature could be used to support, for example, swapping over NFS in a safe way. Expect patches in the near future. 

Finally, there is a problem with direct I/O to transparent hugepages. The kernel will go through and call `get_user_pages_fast()` for each 4K subpage, but that is unnecessary. So 512 mapping calls are being made when one would do. Some kind of fix will eventually need to be made so that this kind of I/O can be done more efficiently. 

#### Lightning talks

Once again, the day ended with lightning talk topics. Matthew Wilcox started by asking developers to work at changing more uninterruptible waits into "killable" waits. The difference is that uninterruptible waits can, if they wait for a long time, create unkillable processes. System administrators don't like such processes; "`kill -9`" should really work at all times. 

The problem is that making this change is often not straightforward; it turns a function call which cannot fail into one which can be interrupted. That means that, for each change, a new error path must be added which properly unwinds any work which had been done so far. That is typically not a simple change, especially for somebody who does not intimately understand the code in question, so it's not the kind of job that one person can just take care of. 

It was suggested that iSCSI drives - which can cause long delays if they fall off the net - are a good way of testing this kind of code. From there, the discussion wandered into the right way of dealing with the problems which result when network-attached drives disappear. They can often hang the system for long periods of time, which is unfortunate. Even worse, they can sometimes reappear as the same drive after buffers have been dropped, leading to data corruption. The solution to all of this is faster and better recovery when devices disappear, especially once it becomes clear that they will not be coming back anytime soon. Additionally, should one of those devices reappear after the system has given up on it, the storage layer should take care that it shows up as a totally new device. Work will be done to this end in the near future. 

Mike Rubin talked a bit about how things are done at Google. There are currently about 25 kernel engineers working there, but few of them are senior-level developers. That, it was suggested, explains some of the things that Google has tried to do in the kernel. 

There are two fundamental types of workload at Google. "Shared" workloads work like classic mainframe batch jobs, contending for resources while the system tries to isolate them from each other. "Dedicated workloads" are the ones which actually make money for Google - indexing, searching, and such - and are very sensitive to performance degradation. In general, any new kernel which shows a 1% or higher performance regression is deemed to not be good enough. 

The workloads exhibit a lot of big, sequential writes and smaller, random reads. Disk I/O latencies matter a lot for dedicated workloads; 15ms latencies can cause phone calls to the development group. The systems are typically doing direct I/O on not-too-huge files, with logging happening on the side. The disk is shared between jobs, with the I/O bandwidth controller used to arbitrate between them. 

Why is direct I/O used? It's a decision which dates back to the 2.2 days, when buffered I/O worked less well than it does now. Things have gotten better, but, meanwhile, Google has moved much of its buffer cache management into user space. It works much like enterprise database systems do, and, chances are, that will not change in the near future. 

Google uses the "fake NUMA" feature to partition system memory into 128MB chunks. These chunks are assigned to jobs, which are managed by control groups. The intent is to firmly isolate all of these jobs, but writeback still can cause interference between them. 

Why, it was asked, does Google not use xfs? Currently, Mike said, they are using ext2 everywhere, and "it sucks." On the other hand, ext4 has turned out to be everything they had hoped for. It's simple to use, and the migration from ext2 is straightforward. Given that, they feel no need to go to a more exotic filesystem. 

Mark Fasheh talked briefly about "cluster convergence," which really means sharing of code between the two cluster filesystems (GFS2 and OCFS2) in the mainline kernel. It turns out that there is a surprising amount of sharing happening at this point, with the lock manager, management tools, and more being common to both. The biggest difference between the two, at this point, is the on-disk format. 

The cluster filesystems are in a bit of a tough place. Neither has a huge group dedicated to its development, and, as Ric Wheeler pointed out, there just isn't much of a hobbyist community equipped with enterprise-level storage arrays out there. So these two projects have struggled to keep up with the proprietary alternatives. Combining them into a single cluster filesystem looks like a good alternative to everybody involved. Practical and political difficulties could keep that from happening for some years, though. 

There was a brief discussion about the DMAPI specification, which describes an API to be used to control hierarchical storage managers. What little support exists in the kernel for this API is going away, leaving companies with HSM offerings out in the cold. There are a number of problems with DMAPI, starting with the fact that it fails badly in the presence of namespaces. The API can't be fixed without breaking a range of proprietary applications. So it's not clear what the way forward will be. 

#### Closing

[![\[Group photo\]](https://static.lwn.net/images/conf/2010/linuxcon/lsf-group-sm.jpg)](<https://lwn.net/Articles/399357/>) The summit was widely seen as a successful event, and the participation of the memory management community was welcomed. So there will be a joint summit again for storage, filesystem, and memory management developers next year. It could happen as soon as early 2011; the participants would like to move the event back to the (northern) spring, and waiting for 18 months for the next gathering seemed like too long.
