---
title: "Btrfs and high-speed devices"
url: https://lwn.net/Articles/698090/
date: "August 24, 2016"
category: "Btrfs; Filesystems-Btrfs"
author: "By Jake Edge August 24, 2016 LinuxCon North America"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
August 24, 2016

* * *

[LinuxCon North America](<https://lwn.net/Archives/ConferenceByYear/#2016-LinuxCon_North_America>)

At [LinuxCon North America](<http://events.linuxfoundation.org/events/linuxcon-north-america>) in Toronto, Chris Mason relayed some of the experiences that his employer, Facebook, has had using Btrfs, especially with regard to its performance on high-speed solid-state storage devices (SSDs). While Mason was the primary developer early on in the history of Btrfs, he is one of a few maintainers of the filesystem now, and the project has seen contributions from around 70 developers throughout the Linux community in the last year. 

[ ![\[Chris Mason\]](https://static.lwn.net/images/2016/lcna-mason-sm.jpg) ](<https://lwn.net/Articles/698107/>)

He is on the kernel team at Facebook; one of the main reasons the company wanted to hire him was because it wanted to use Btrfs in production. Being able to use Btrfs in that kind of environment is also the primary reason he chose to take the job, he said. As the company is rolling Btrfs out, it is figuring out which features it wants to use and finding things that work well and not so well. 

Mason went through the usual list of high-level Btrfs features, including efficient writable snapshots, internal RAID with restriping, online device management, online scrubbing to check in the background if the CRCs are the same as when the data was written, and so on. The CRCs for both data and metadata are a feature that "saved us a lot of pain" at Facebook, he said. 

The Btrfs CRC checking means that a read from a corrupted sector will cause an I/O error rather than return garbage. Facebook had some storage devices that would appear to store data correctly in a set of logical block addresses (LBAs) until the next reboot, at which point reads to those blocks would return GUID partition table (GPT) data instead. He did not name the device maker because it turned out to actually be a BIOS problem. In any case, the CRCs allowed the Facebook team to quickly figure out that the problem was not in Btrfs when it affected thousands of machines as they were rebooted for a kernel upgrade. 

Volume management in Btrfs is done in terms of "chunks", which are normally 1GB in size. That is part of what allows the filesystem to handle differently sized devices for RAID volumes, for example. Volumes can have specific chunks reserved for data or metadata and different RAID levels can be applied to each (e.g. RAID-1 for the metadata and RAID-5 for the data). 

But Btrfs has had some lock-contention problems; it still has some of them, he said, though there are improvements coming. The filesystem is optimized for use on SSDs, but he ran an [fs_mark](<https://sourceforge.net/projects/fsmark/>) benchmark in a virtual machine (for comparative rather than hard numbers) creating zero-length files and found that XFS could create roughly four times the number of files per second (33,000 versus 9,000). That was "not good", but before he started tuning Btrfs, he wanted to make XFS go as fast as he could. 

To that end, he looked at what XFS was blocked on, which turned out to be locks for allocating filesystem objects. By increasing the allocation groups in the filesystem when it was created (from four to sixteen to match the number of CPUs in his test system), he could increase its performance to 200,000 file-creations per second. At that point, it was mostly CPU bound and the function using the most CPU was one that could not be easily tweaked away with a `mkfs` option. 

So then he turned to Btrfs. Using perf, he was able to see that there was lock contention on the B-tree locks. The Btrfs B-tree stores all of its data in the leaves of the tree; when it is updating the tree, it has to lock non-leaf nodes on the way to the leaf, starting with the root node. For some operations, those locks have to be held as it traverses the tree. Hopefully only the leaf needs to be locked, but sometimes that is not the case and, since everything starts at the root, it is not surprising that there is contention for that lock. 

As an experiment to make Btrfs go faster, he used the subvolume feature to effectively create more root nodes. Instead of the usual one volume (with one root node), he created sixteen subvolumes so that there was one per CPU, each with its own root node and lock. That allowed Btrfs to get close to the XFS performance at 175,000 file-creations per second. 

But the goal was to make the filesystem faster without resorting to subvolumes, which led to a new B-tree locking scheme. By default, Btrfs has 16KB nodes, which is not changing, but instead of being treated as a single group, each node will now be broken up into sixteen groups, each with its own lock. 

He has not yet picked the best number of groups for each node, but the change allows a default Btrfs filesystem create 90,000 files per second. There are a lot of assumptions in Btrfs that there is only one lock per node, which he is working on removing. In addition, Btrfs switched to reader/writer locks a ways back and it turns out that those perform worse than expected, so he will be looking into that. 

By some other measures, though, Btrfs compares favorably with XFS on the benchmark. XFS writes 120MB/second and does 3000 I/O operations/second (IOPS) for the benchmark, while Btrfs does 50MB/second and 300 IOPS to accomplish the same amount of work. That means that Btrfs is ordering things better and doing less I/O, Mason said. 

The Gluster workloads at Facebook, which use rotational storage, are extremely sensitive to metadata latency to the point where one node's high latency can make the entire cluster slower than it should be. In the past, the company has used [flashcache](<https://github.com/facebookarchive/flashcache>) (which is similar to [bcache](<https://bcache.evilpiepirate.org/>)) for both XFS and Btrfs to cache some data and metadata on SSDs, which improves the metadata latencies, but not enough. 

To combat that, he has a set of patches to automatically put the Btrfs metadata on SSDs. The block layer provides information on whether the storage is rotational; for now, his patch assumes that if it is not rotational then it is fast. The patch has made a huge difference in the latencies and requires less flash storage (e.g. 450GB for 40TB filesystem) for Facebook's file workload that consists of a wide variety of file sizes. "You will need a lot more metadata if you have all 2KB files", he said. 

That patch set is small (73 lines of code added), which is nice, he said. It is not entirely complete, though, as btrfs-utils needs changes to support it, but that should be a similarly sized change. 

Another bottleneck he has encountered is in using the trim (or discard) command to tell SSDs about blocks that are no longer in use by the filesystem. That allows the flash translation layer to ignore those blocks when it is doing garbage collection and should, in theory, provide better performance. But many devices are slow when handling trim commands. Both XFS and Btrfs keep lists of blocks to trim, submit them as trim commands, and then must wait for those commands to complete during transaction commits, which stalls new filesystem operations. Those stalls can be huge, on the order of "tens of seconds", he said. 

Ric Wheeler spoke up to say that trim is simply a request that the drive is free to ignore. He suggested that trim should not be performed during regular filesystem operations. Ted Ts'o agreed and said that the best practice for ext4 and probably other filesystems was to run the `fstrim` batch-trimming command regularly out of `cron`. 

In answer to a question, Mason said that the disadvantages of not trimming are device-dependent. In some cases, it may reduce the lifetime of the device or add latencies during garbage collection, but it may also do nothing. Wheeler pointed out that if you are using thin provisioning, though, failing to trim could cause the storage to run out of space when there is actually space available. 

Though it is not a flash-specific change, there have been some problems with large (> 16TB) Btrfs filesystems because of the free-space cache. Originally, free extents were not tracked, but that required scanning the entire filesystem at mount time, which was slow. When free-space was added, the cache was per-block-group and large filesystems have a lot of block groups, which meant that there was more caching on each commit. In the 4.5 kernel, Omar Sandoval added a new free-space cache (which can be enabled with `-o space_cache=v2`) that is "dramatically faster", with commit latencies dropping from four to zero seconds. 

For the near future, he plans to finalize the new B-tree locking and improve some `fsync()` bottlenecks, though he thinks that the new space cache will help there. There are also some other spinlocks slowing things down that he wants to look at. 

He mentioned a few of the tools that he uses to find bottlenecks. Perf is the right tool when processing is "pegged in the CPU", but finding problems when things are blocking is much harder. For that, he recommended [BPF](<http://man7.org/linux/man-pages/man2/bpf.2.html>) and [BCC](<https://github.com/iovisor/bcc>). In particular, Brendan Gregg's [offcputime](<https://github.com/iovisor/bcc/blob/master/tools/offcputime.py>) BPF script is useful to show both kernel and application stack traces to help show the reasons why a process is blocked. In fact, Facebook likes offcputime so much that fellow Btrfs maintainer Josef Bacik has created a way to aggregate the output of the program across multiple systems. 

There were a few questions at the end of the session. One person asked whether Mason had seen any uptake of Btrfs for smaller devices. Mason said that the filesystem "needs love and care" when it is being used, which is why Facebook can use it. Someone with an ARM background would need to be working on Btrfs upstream in order to provide that kind of care if it were to be adopted on ARM-powered devices, he said. 

Another asked how much faster the current design of Btrfs could go. Mason seemed quite optimistic that it could go "much faster". The metadata format is flexible, so "if things are broken, we can fix them". 

The last two questions regarded two different benchmarks, both of which are interesting, but neither of which Mason has run. Flashcache versus bcache would likely provide similar numbers, he thought, but flashcache worked for Facebook so there was no need to try bcache. He also has not run benchmarks against ZFS. When he started Btrfs, ZFS was not available. There is no reason not to do so now, he said, but he hasn't, though he would be interested in the results. 

[I would like to thank the Linux Foundation for travel assistance to Toronto for LinuxCon North America.]
