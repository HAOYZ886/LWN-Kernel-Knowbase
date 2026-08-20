---
title: Writeback and control groups
url: https://lwn.net/Articles/648292/
date: "June 17, 2015"
category: "Control groups-IO bandwidth controllers; Memory management-Writeback"
author: "By Jonathan Corbet June 17, 2015 LinuxCon Japan"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
June 17, 2015

* * *

[LinuxCon Japan](<https://lwn.net/Archives/ConferenceByYear/#2015-LinuxCon_Japan>)

"Writeback" is the process of writing dirty pages in memory back to permanent storage. It is a tricky job; the kernel must arbitrate the use of limited I/O bandwidth while ensuring that the system is not overwhelmed by dirty pages. Some years ago, writeback was a perennial discussion topic at gatherings of memory-management developers; the kernel did not do as good a job as anybody would have liked. Those problems have, for the most part, been solved in recent years — until one adds control groups into the mix. A solution to that problem is in the works, though, and should be hitting the mainline in the near future. 

Tejun Heo took some time to discuss the current situation during his LinuxCon Japan talk. The memory-management subsystem will, by default, try to limit dirty pages to a maximum of 15% of the memory on the system. There is a "magical function" called `balance_dirty_pages()` that will, if need be, throttle processes dirtying a lot of pages in order to match the rate at which pages are being dirtied and the rate at which they [![\[Tejun Heo\]](https://static.lwn.net/images/conf/2015/lcj/TejunHeo-sm.jpg)](<https://lwn.net/Articles/648306/>) can be cleaned. It works reasonably well in current kernels, but it only operates globally; it is not equipped to deal with control groups. 

On the control group side, the memory controller can regulate the amount of memory that is available to any given group, while the block controller is in charge of regulating I/O bandwidth use. Writeback is clearly related to both memory use and I/O bandwidth, but the control-group mechanism offers no way to enable controllers to work together — so these two controllers don't. The result, Tejun said, is a "really sad situation." 

The memory controller currently tags pages in memory with owner information so that it knows which control group to charge for each page. The block controller is unable to use that information, though, so it has no way of knowing which control group to charge for writeback I/O traffic. So control groups do not use the system's global throttling mechanism at all; instead, there is a "hacky" mechanism built into the memory controller itself that, according to Tejun, "does not throttle anything effectively." It ignores the global dirty-page watermarks that control throttling and is, he said, "completely broken." There has been talk of fixing the situation for at least five years but nothing has been done, leading to a certain amount of frustration. 

#### Fixing writeback in control groups

So Tejun set out to deal with the problem. His approach is driven by the idea that control-group features should not need completely new mechanisms for their implementation — writeback control in control groups should use the same mechanism that the system as a whole uses. The global mechanism should just be a degenerate form of the single-group case. 

There are two structures involved in writeback control in the kernel. `struct backing_dev_info` contains information about a specific device to which dirty pages are being written; it tracks the observed I/O bandwidth of the device and how it is being used. The `bdi_writeback` structure, instead, regulates writeback activity in particular. There is currently a single `bdi_writeback` structure for each `backing_dev_info` structure, and the separation of their roles is somewhat fuzzy. (Both of these structures are defined in `[include/linux/backing-dev.h](<https://lwn.net/Articles/648296/>)`) 

One of the first things Tejun's [control-group writeback support patch set](<https://lwn.net/Articles/645708/>) does is to move more writeback-specific information from `struct backing_dev_info` into the `bdi_writeback` structure. That structure then goes from a single instance per device to one instance for each control group, allowing for each group to be regulated separately. `balance_dirty_pages()` is changed to use the per-group `bdi_writeback` structure, as are other pieces of the writeback-control mechanism. Tejun described it as being mostly "a giant plumbing job." 

#### Details

The completion of that plumbing job allows the block bandwidth controller to regulate writeback I/O, but it is missing an important piece: the throttling of processes that are dirtying more memory than can be cleaned within their group's I/O bandwidth limits. Or, more precisely, while the system can throttle processes when the global dirty-page limit is reached, it cannot throttle those that have dirtied too much of the memory that is available to their specific control group. Solving that problem is the subject of [a separate patch set](<https://lwn.net/Articles/645707/>) adding per-group throttling. 

This patch set adds a new structure (`struct wb_domain`) for the control of dirty-page throttling. There is one global domain that implements the "15% of total memory" limit that exists in current kernels. Each control group gets its own `wb_domain` structure as well, to enforce limits specific to that group. When the memory-management code computes the number of pages that a process within a specific group is allowed to dirty, it looks at both the global and per-group `wb_domain` structures and uses the more restrictive of the two. A process will never be allowed to exceed the number of dirty pages allowed to its control group, but that limit may be lowered if the system as a whole has a lot of dirty pages. 

That is still not a complete solution to the problem, though. The writeback mechanism uses the inode (open file) as its fundamental unit of control, while the memory controller applies limits on a per-page basis. Tejun explained that each makes sense within its own context, but there is a mismatch between the two that makes it harder to make those mechanisms work well together. 

The writeback mechanism is designed to focus on a single inode at a time; among other things, writing out all of a single file's dirty pages together tends to improve disk I/O locality. When the I/O bandwidth controller first sees writeback activity for an inode, it assigns "ownership" of the inode to the control group responsible for that activity. Thereafter, all writeback activity for that inode is charged to that control group, regardless of who actually dirtied the pages. Tejun looked into making the accounting more fine-grained but, he said, the result was far too complex and wasn't worth it. In the end, one control group is usually responsible for the majority of writeback traffic to any given file. 

There is still a problem, though, that the initial assignment of responsibility for any given file might be incorrect. Or the file could move from one control group to another over time. In either case, the result could be that one group finds itself charged for large amounts of writeback created by another group entirely. 

To resolve that issue, Tejun has posted [yet another patch set](<https://lwn.net/Articles/645706/>) adding "foreign cgroup inode bdi_writeback switching." This mechanism watches the ownership of the pages (as tracked by the memory controller) being written back to each inode. Using the [Boyer-Moore majority vote algorithm](<https://en.wikipedia.org/wiki/Boyer-Moore_Majority_Vote_Algorithm>), it decides which control group is responsible for the most I/O traffic. If most traffic originates in a group other than the owner of that inode, and that pattern holds for a period of time (two seconds, in the current patch), the ownership of the inode will be switched to the new "winner". Over time, that mechanism should ensure that writeback I/O traffic is charged correctly without adding the need to track things on a sub-inode level. 

As for the status of all this work: Tejun said that it works and is currently slated for the 4.2 merge window. That said, it is still experimental and there are probably some issues to be shaken out. At the time of the talk, only the ext2 filesystem was supported; since then, [ext4 support](<https://lwn.net/Articles/648299/>) has been posted as well. Each filesystem will require changes to support the new writeback mechanism, but the changes tend to be quite small. Getting those pieces into place should not take too long; then, once this work stabilizes, another longstanding Linux memory-management shortcoming should be no more. 

[Your editor would like to thank the Linux Foundation for supporting his travel to LinuxCon Japan.]
