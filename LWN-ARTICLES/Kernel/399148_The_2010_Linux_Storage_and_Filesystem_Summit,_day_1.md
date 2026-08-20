---
title: "The 2010 Linux Storage and Filesystem Summit, day 1"
url: https://lwn.net/Articles/399148/
date: "August 9, 2010"
category: "Block layer; Filesystems-Workshops"
author: "By Jonathan Corbet August 9, 2010"
---

By **Jonathan Corbet**  
August 9, 2010

The fourth Linux storage and filesystem summit was held August 8 and 9 in Boston, immediately prior to LinuxCon. This time around, a number of developers from the memory management community were present as well. Your editor was also there; what follows are his notes from the first day of the summit. 

#### Testing tools

The first topic of the workshop was "testing and tools," led by Eric Sandeen. The 2009 workshop identified a generic test suite as something that the community would very much like to have. One year later, quite a bit of progress has been made in the form of the xfstests package. As the name suggests, this test suite has its origins in the XFS filesystem, and it is still somewhat specific to XFS. But, with the addition of generic testing last May, about 70 of the 240 tests are now generic. Xfstests is concerned primarily with regression testing; it is not, generally, a performance-oriented test suite. Tests tend to get added when somebody stumbles across a bug and wants to verify that it's fixed - and that it stays fixed. Xfstests also does not have any real facility for the creation of test filesystems; tools like [Impressions](<http://lwn.net/Articles/385081/>) are best used for that purpose. 

About 40 new tests have been added to xfstests over the last year; it is now heavily used in ext4 development. Most tests look for specific bugs; there isn't a whole lot of coverage for extreme situations - millions of files in one directory and such. Those tests just tend to take too long. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

It was emphasized that just running xfstests is not enough on its own; tests must be run under most or all reasonable combinations of mount options to get good coverage. Ric Wheeler also pointed out that different types of storage have very different characteristics. Most developers, he fears, tend to test things on their laptops and call the result good. Testing on other types of storage, of course, requires access to the hardware; USB sticks are easy, but not all developers can test on enterprise-class storage arrays. 

Tests which exercise more of the virtual memory and I/O paths would also be nice. There is one package which covers much of this ground: FIO, available from [kernel.dk](<http://brick.kernel.dk/snaps/>). Destructive power failure testing is another useful area which Red Hat (at least) is beginning to do. There has also been some work done using hdparm to corrupt individual sectors on disk to see how filesystems respond. A wishlist item was better latency measurement, with an emphasis on seeing how long I/O requests sit within drivers which do their own queueing. It was suggested that what is really needed is some sort of central site for capturing wishlist ideas for future tests; then, whenever somebody has some time available, those ideas are available. 

In an attempt to better engage the memory management developers in the room, it was asked: how can we make tests which cover writeback? The key, it seems, is to choose a workload which is large enough to force writeback, but not so large that it pushes the system into heavy swapping. One simple test is a large tar run; while that is happening, monitor `/proc/vmstat` to see when writeback kicks in, and when things get bad enough that direct reclaim is invoked. An arguably more representative test can be had with sysbench; again, the key is to tune it so that the shared buffers fit within physical memory. 

But, as Nick Piggin pointed out, anything that dirties memory is, in the end, a writeback test. The key is to find ways of making tests which adequately model real-world workloads. 

#### Memory-management testing

Your editor is messing with the timing here, but the session on testing of memory management changes fits well with the above. So please ignore the fact that this session actually happened after lunch. 

The question here is simple: how can memory management changes be tested to satisfy everybody? This is a subject which has been coming up for years; memory management changes seem to be especially subject to "have you tested with this other kind of workload?" questions. Developers find this frustrating; it never seems to be possible to do enough testing to satisfy everybody, especially since people asking for testing of different workloads are often unable or unwilling to supply an actual test program. 

It was suggested that the real test should be "put the new code on the Google cluster and see if the Internet breaks." There are certain practical difficulties with this approach, however. So the question remains: how can a developer conclude that a memory management change actually works? Especially in a situation where "works" means different things to different people? There is far too wide a variety of workloads to test them all. Beyond that, memory management changes often involve tradeoffs - making one workload better may mean making another one worse. Changes which make life better for everybody are rare. 

Still, it was agreed that a standard set of tests would help. Some suggestions were made, including hackbench, netperf, various database benchmarks (pgbench or sysbench, for example), and the "compilebench" test which is popular with kernel developers. There was also some talk of microbenchmarks; Nick Piggin noted that microbenchmarks are terrible when it comes to arguing for the inclusion of a change, but they can be useful for the detection of performance regressions. 

Sometimes running a single benchmark is not enough; many memory management problems are only revealed when the system comes under a combination of stresses. And Andrea Arcangeli made the point that, in the end, only one test really matters: how much time does it take the system to complete running a workload which exceeds the amount of physical RAM available? 

There was some discussion of the challenges involved in tracking down problems; Mel Gorman stated that the debugability of the virtual memory subsystem is "a mess." Tracepoints can be useful for this purpose, but they are hard to get merged, partly due to Andrew Morton's [hostility to tracepoints in general](<http://lwn.net/Articles/330402/>). There is also ongoing concern about the ABI status of tracepoints; what happens when programs (perhaps being run by large customers of enterprise distributions) depend on tracepoints which expose low-level kernel details? Those tracepoints may no longer make sense after the code changes, but breaking them may not be an option. 

#### Filesystem freeze/thaw

The [filesystem freeze feature](<http://lwn.net/Articles/287435/>) enables a system administrator to suspend writes to a filesystem, allowing it to be backed up or snapshotted while in a consistent state. It had its origins in XFS, but has since become part of the Linux VFS layer. There are a few issues with how freeze works in current kernels, though. 

The biggest of these problems is unmounting - what happens when the administrator unmounts a frozen filesystem? In current kernels, the whole thing hangs, leaving the system with an unusable, un-unmountable filesystem \- behavior which does not further the Linux World Domination goal. So four possible solutions were proposed: 

  1. Simply disallow the unmounting of frozen filesystems. Al Viro stated that this solution is not really an option; there are cases where the unmount cannot be disallowed. When the final process exits the namespace in which the filesystem is mounted is one of those cases. Disallowing unmounts would also break the useful `umount -l` option, which is meant to work at all times. 

  2. Keep the filesystem frozen across the unmount, so that the filesystem would still be frozen after the next mount. The biggest problem here is that there may be changes that the filesystem code needs to write to the device; if the system reboots before that can happen, bad things can result. 

  3. Automatically thaw filesystems on unmount. 

  4. Add a new `ioctl()` command which will cause the thawing of an unmounted filesystem. 

Al suggested a variant on #3, in the form of a new freeze command. The proper way to handle freeze is to return a file descriptor; as long as that file descriptor is held open, the filesystem remains frozen. This solves the "last process exits" problem because the file descriptor will be closed as the process exits, automatically causing the filesystem to be thawed. Also, as Al noted, the `kill` command is often the system recovery tool of choice for system administrators, so having a suitably-targeted `kill` cause a frozen filesystem to be thawed makes sense. 

There seemed to be a consensus that the file descriptor approach is the best long-term solution. Meanwhile, though, there are tools based on the older `ioctl()` commands which will take years to replace in the field. So we might also see an implementation of #4, above, to help in the near future. 

#### Barriers

Contemporary filesystems go to great lengths to avoid losing data - or corrupting things - if the system crashes. To that end, quite a bit of thought goes into writing things to disk in the correct order. As a simple example, operations written to a filesystem journal must make it to the media before the commit record which marks those operations as valid. Otherwise, the filesystem could end up replaying a journal with random data, with an outcome that few people would love. 

All of that care is for nothing, though, if the storage device reorders writes on their way to the media. And, of course, reordering is something that storage devices do all the time in the name of increasing performance. The solution, for years now, has been "barrier" operations; all writes issued before a barrier are supposed to complete before any writes issued after the barrier. The problem is that barriers have not always been well supported in the Linux block subsystem, and, when they are supported, they have a significant impact on performance. So, even now, many systems run with barriers disabled. 

Barriers have been discussed among the filesystem and storage developers for years; it was hoped that this year, with the memory management developers present as well, some better solutions might be found. 

There was some discussion about the various ways of implementing barriers and making them faster. The key point in the discussion, though, was the assertion that barriers are not really the same as the memory barriers they were patterned after. There are, instead, two important aspects to block subsystem barriers: request ordering and forcing data to disk. That led, eventually, to one of the clearest decisions in the first day of the summit: barriers, as such, will be no more. The problem of ordering will be placed entirely in the hands of filesystem developers, who will ensure ordering by simply waiting for operations to complete when needed. There will be no ordering issues as such in the block layer, but block drivers will be responsible for explicitly flushing writes to physical media when needed. 

Whether this decision will lead to better-performing and more robust filesystem I/O remains to be seen, but it is a clearer description of the division of responsibilities than has been seen in the past. 

#### Transparent hugepages

At this point, the summit split into three tracks for storage, filesystem, and memory management developers. Your editor followed the memory management track; with luck, we'll eventually have writeups from the other tracks as well. 

Andrea Arcangeli presented his [transparent hugepages](<http://lwn.net/Articles/359158/>) work, starting with a discussion of the advantages of hugepages in general. Hugepages are a feature of many contemporary processors; they allow the memory management subsystem to use larger-than-normal page sizes in parts of the virtual address range. There are a number of advantages to using hugepages in the right places, but it all comes down to performance. 

A hugepage takes much of the pressure off the processor's translation lookaside buffer (TLB), speeding memory access. When a TLB miss happens anyway, a 2MB hugepage requires traversing three levels of page table rather than four, saving a memory access and, again, reducing TLB pressure. The result is a doubling of the speed with which initial page faults can be handled, and better application performance in general. There can be some costs, especially when entire hugepages must be cleared or copied; that can wipe out much of the processor's cache. But this cost tends to be overwhelmed by the performance advantages that hugepages bring. 

Those advantages, incidentally, are multiplied when hugepages are used on systems hosting virtualized guests. Using hugepages in this situation can eliminate much of the remaining cost of running virtualized systems. 

Hugepages on Linux are currently accessed through the hugetlbfs filesystem, which was [discussed in great detail by Mel Gorman](<http://lwn.net/Articles/374424/>) on LWN earlier this year. There are some real limitations associated with hugetlbfs, though: hugepages are not swappable, they must be reserved at boot time, there is no mixing of page sizes in the same virtual memory area, etc. Many of these problems could be fixed, but, as Andrea put it, hugetlbfs is becoming a sort of secondary - and inferior \- Linux virtual memory subsystem. It is time to turn hugepages into first-class citizens in the real Linux VM. 

Transparent hugepages eliminate much of the need for hugetlbfs by automatically grouping together sections of a process's virtual address space into hugepages when warranted. They take away the hassles of hugetlbfs and make it possible for the system to use hugepages with no need for administrator intervention or application changes. There seems to be a fair amount of interest in the feature; a Google developer said that the feature is attractive for internal use. 

At the core of the patch is a new thread called `khugepaged`, which is charged with scanning memory and creating hugepages where it makes sense. Other parts of the VM can split those hugepages back into normally-sized pieces when the need arises. Khugepaged works by allocating a hugepage, then using the migration mechanism to copy the contents of the smaller component pages over. There was some talk of trying to defragment memory and "collapse in place" instead, but it doesn't seem worth the effort at this point. The amount of work to be done would be about the same except in the special case where a hugepage had been split and was being grouped back together before much had changed - a situation which is expected to be relatively rare. 

Andrea put up a number of benchmarks showing how transparent hugepages improve performance; the all-important kernel compile benchmark (seen as a sort of worst case for hugepages) is 2.5% faster. Various other benchmarks show bigger improvements. 

Transparent hugepages, it seems, will be enabled by default in the RHEL 6 kernel. Andrea would really like to get the feature into 2.6.36, but the merge window is already well advanced and it's not clear that things will work out that way. There is still a need to convince Linus that the feature is worthwhile, and, perhaps, some work to be done to enable the feature on SPARC systems. 

#### mmap_sem

The memory map semaphore (`mmap_sem`) is a reader-writer semaphore which protects the tree of virtual memory area (VMA) structures describing each address space. It is, Nick Piggin says, one of the last nasty locking issues left in the virtual memory subsystem. Like many busy, global locks, `mmap_sem` can cause scalability problems through cache line bouncing. In this case, though, simple contention for the lock can be a problem; `mmap_sem` is often held while disk I/O is being performed. With some workloads, the amount of time that `mmap_sem` is held can slow things down significantly. 

Various groups, including developers at HP and Google, have chipped away at the `mmap_sem` problem in the past, usually by trying to drop the semaphore in various paths. These patches have all run into the same problem, though: Linus hates them. In particular, he seems to dislike the additional complication added to the retry paths which must be followed when things change while the lock is dropped. So none of this work has gotten into the mainline. 

There have also been some [unfair rwsem](<http://lwn.net/Articles/387720/>) proposals aimed at reducing `mmap_sem` contention; these have run aground over fears of writer starvation. 

According to Nick, the real problem is the red-black tree used to track allocated address space; the data structure is cache-unfriendly and requires a global lock for its protection. His idea is to do away with this rbtree and associate VMAs directly with the page table entries, protecting them with the PTE locks. This approach would eliminate much of the locking entirely, since the page tables must be traversable without locks, and solve the `mmap_sem` problem. 

That said, there are some challenges. A VMA associated with a page table entry can cover a maximum of 2MB of address space; larger areas would have to be split into (possibly a large number of) smaller VMAs. It's not clear how this mechanism would then interact with hugepages. The instantiation of large VMAs would require the creation of the full range of PTEs, which is not required now; that could hurt applications with very sparsely-populated memory areas. Growing VMAs would have its own challenges. There is also the issue of free space allocation, a problem which might be solved by preallocating ranges of addresses to each thread sharing an address space. In summary, the list of obstacles to be overcome before this idea becomes practical looks somewhat daunting. 

The developers in the room seemed to not be entirely comfortable with this approach, but nobody could come up with a fundamental reason why it would not work. So we'll probably be seeing patches from Nick exploring this idea in the future. 

#### copyfile()

The [`reflink()`](<http://lwn.net/Articles/333783/>) system call was originally proposed as a sort of fast copy operation; it would create a new "copy" of a file which shared all of the data blocks. If one of the files were subsequently written to, a copy-on-write operation would be performed so that the other file would not change. LWN readers last heard about this patch last September, when Linus [refused to pull it](<http://lwn.net/Articles/353048/>) for 2.6.32. Among other things, he didn't like the name. 

So now `reflink()` is back as `copyfile()`, with some proposed additional features. It would make the same copy-on-write copies on filesystems that support it, but `copyfile()` would also be able to delegate the actual copy work to the underlying storage device when it makes sense. For example, if a file is being copied on a network-mounted filesystem, it may well make sense to have the server do the actual copy work, eliminating the need to move the data over the network twice. The system call might also do ordinary copies within the kernel if nothing faster is available. 

The first question that was asked is: should `copyfile()` perhaps be an asynchronous interface? It could return a file descriptor which could be polled for the status of the operation. Then, graphical utilities could start a copy, then present a progress bar showing how things were going. Christoph Hellwig was adamant, though, that `copyfile()` should be a synchronous operation like almost all other Linux system calls; there is no need to create something weird and different here. Progress bars neither justify nor require the creation of asynchronous interfaces. 

There was also opposition to the mixing of the old `reflink()` idea with that of copying a file. There is little perceived value in creating a bad version of `cp` within the kernel. The two ideas were mixed because it seems that Linus seems to want it that way, but, after this discussion, they may yet be split apart again. 

#### Dirty limits

Jan Kara led a short discussion on the problem of dirty limits. The tuning knob found at `/proc/sys/vm/dirty_ratio` contains a number representing a percentage of total memory. Any time that the number of dirty pages in the system exceeds that percentage, processes which are actually writing data will be forced to perform some writeback directly. This policy has a couple of useful results: it helps keep memory from becoming entirely filled with dirty pages, and it serves to throttle the processes which are creating dirty pages in the first place. 

The default value for `dirty_ratio` is 20, meaning that 20% of memory can be dirty before processes are conscripted into writeback duties. But that turns out to be too low for a number of applications. In particular, it seems that many Berkeley DB applications exhibit behavior where they dirty a lot of pages all over memory; setting `dirty_ratio` too low causes a lot of excessive I/O and serious performance issues. For this reason, distributions like RHEL raise this limit to 40% by default. 

But 40% is not an ideal number either; it can lead to a lot of wasted memory when the system's workloads are mostly sequential. Lots of dirty pages can also cause `fsync()` calls to take a very long time, especially with the ext3 filesystem. What's really needed is a way to set this parameter in a more automatic, adaptive way, but exactly how that should be done is not entirely clear. 

What is likely to happen in the short term is that a user-space daemon will be written to experiment with various policies for `dirty_ratio`. Some VM tracepoints can be used to catch events and tune things accordingly. A system which is handling a lot of `fsync()` calls should probably have a lower value of `dirty_ratio`, for example. In the absence of reasons to the contrary, the daemon can try to nudge the limit higher and try to see if applications perform better. This kind of heuristic experimentation has its hazards, but there does not seem to be a better method on offer at the moment. 

#### Topology and alignment

There was a brief session on storage device topology issues; unfortunately, it was late in the day and your editor's notes are increasingly fuzzy. Much of the discussion had to do with 4K-sector disks. There are still issues, it seems, with drives which implement strange sector alignments in an attempt to get better performance with some versions of Windows. Linux can cope with these drives, but only if the drives themselves export information on what they are doing. Not all hardware provides that information, unfortunately. 

Meanwhile, the amount of software which does make use of the topology information exported through the kernel is increasing. Partitioning tools are getting smarter, and the device mapper now uses this information properly. The readahead code will be tweaked to create properly-aligned requests when possible. 

#### Lightning talks

The last session of the day was dedicated to three lightning talks. The first, by Matthew Wilcox, had to do with merging of git trees. Quite a bit of work in the VM/filesystem/storage area depends on changes made in a number of different trees. Making those trees fit together can be a bit of a challenge. That problem can be solved in linux-next, but those solutions do not necessarily carry over into the mainline, where trees may be pulled in just about any order - or not at all. The result is a lot of work and merge-window scrambling by developers, who are getting a little tired of it. 

So, it was asked, is it time for a git tree dedicated to storage as a whole, and a storage maintainer to go with it? The idea was to create something like David Miller's networking tree, which is the merge point for almost all networking-related patches. James Bottomley made the mistake of suggesting that this kind of discussion could not go very far without a volunteer to manage that tree; he was then duly volunteered for the job. 

The discussion moved on to how this tree would work, and, in particular, whether its maintainer would become the "overlord of storage," or whether it would just be a more convenient place to work out merge conflicts. If its maintainer is to be a true overlord, a fairly hardline approach will need to be taken with regard to when patches would have to be ready for merging. It's not clear whether the storage community is ready to deal with such a maintainer. So, for the near future, James will run the tree as a merge point to see whether that helps developers get their code into the mainline. If it seems like there is need for a real storage maintainer, that question will be addressed after a development cycle or two. 

Dan Magenheimer presented his [Cleancache](<http://lwn.net/Articles/386090/>) proposal, mostly with an eye toward trying to figure out a way to get it merged. There is still some opposition to it, and its per-filesystem hooks in particular. It's hard to see how those hooks can be avoided, though; Cleancache is not suitable for all filesystems and, thus, may not be a good fit for the VFS layer. The crowd seemed reasonably amenable to merging the patches, but the chief opponent - Christoph Hellwig - was not in the room at the time. So no real conclusions have been reached. 

The final lightning talk came from Boaz Harrosh, who talked about "stable pages." Currently, pages which are currently under writeback can be modified by filesystem code. That's potentially a data integrity problem, and it can be fatal in situations where, for example, checksums of page contents are being made. That is why the RAID5 code must copy all pages being written to an array; changing data would break the RAID5 checksums. What, asked Boaz, would break if the ability to change pages under writeback were withdrawn? 

The answer seems to be that nothing would break, but that some filesystems might suffer performance impacts. The only way to find out for sure, though, is to try it. As it happens, this is a relatively easy experiment to run, so filesystem developers will probably start playing with it sometime soon. 

That was the end of the first day of the summit; reports from the second day will be posted as soon as they are ready.
