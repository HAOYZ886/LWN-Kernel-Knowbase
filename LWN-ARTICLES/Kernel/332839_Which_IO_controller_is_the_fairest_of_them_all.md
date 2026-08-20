---
title: "Which I/O controller is the fairest of them all?"
url: https://lwn.net/Articles/332839/
date: "May 12, 2009"
category: "Block layer-IO scheduling; Containers; Device mapper"
author: "By Jonathan Corbet May 12, 2009"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 12, 2009

An I/O controller is a system component intended to arbitrate access to block storage devices; it should ensure that different groups of processes get specific levels of access according to a policy defined by the system administrator. In other words, it prevents I/O-intensive processes from hogging the disk. This feature can be useful on just about any kind of system which experiences disk contention; it becomes a necessity on systems running a number of virtualized (or containerized) guests. At the moment, Linux lacks an I/O controller in the mainline kernel. There is, however, no shortage of options out there. This article will look at some of the I/O controller projects currently pushing for inclusion into the mainline. 

![\[Block layer structure\]](https://static.lwn.net/images/ns/kernel/io-controller.png) For the purposes of this discussion, it may be helpful to refer to your editor's bad artwork, as seen on the right, for a simplistic look at how block I/O happens in a Linux system. At the top, we have several sources of I/O activity. Some requests come from the virtual memory layer, which is cleaning out dirty pages and trying to make room for new allocations. Others come from filesystem code, and others yet will originate directly from user space. It's worth noting that only user-space requests are handled in the context of the originating process; that creates complications that we'll get back to. Regardless of the source, I/O requests eventually find themselves at the block layer, represented by the large blue box in the diagram. 

Within the block layer, I/O requests may first be handled by one or more virtual block drivers. These include the device mapper code, the MD RAID layer, etc. Eventually a (perhaps modified) request heads toward a physical device, but first it goes into the I/O scheduler, which tries to optimize I/O activity according to a policy of its own. The I/O scheduler works to minimize seeks on rotating storage, but it may also implement I/O priorities or other policy-related features. When it deems that the time is right, the I/O scheduler passes requests to the physical block driver, which eventually causes them to be executed by the hardware. 

All of this is relevant because it is possible to hook an I/O controller into any level of this diagram - and the various controller developers have done exactly that. There are advantages and disadvantages to doing things at each layer, as we will see. 

#### dm-ioband

The [dm-ioband patch](<https://lwn.net/Articles/332934/>) by Ryo Tsuruta (and others) operates at the virtual block driver layer. It implements a new device mapper target (called "ioband") which prioritizes requests passing through. The policy is a simple proportional weighting system; requests are divided up into groups, each of which gets bandwidth according to the weight assigned by the system administrator. Groups can be determined by user ID, group ID, process ID, or process group. Administration is done with the `dmsetup` tool. 

dm-ioband works by assigning a pile of "tokens" to each group. If I/O traffic is low, the controller just stays out of the way. Once traffic gets high enough, though, it will charge each group for every I/O request on its way through. Once a group runs out of tokens, its I/O will be put onto a list where it will languish, unloved, while other groups continue to have their requests serviced. Once all groups which are actively generating I/O have exhausted their tokens, everybody gets a new set and the process starts anew. 

The basic dm-ioband code has a couple of interesting limitations. One is that it does not use the control group mechanism, as would normally be expected for a resource controller. It also has a real problem with I/O operations initiated asynchronously by the kernel. In many cases - perhaps the majority of cases - I/O requests are created by kernel subsystems (memory management, for example) which are trying to free up resources and which are not executing in the context of any specific process. These requests do not have a readily-accessible return label saying who they belong to, so dm-ioband does not know how to account for them. So they run under the radar, substantially reducing the value of the whole I/O controller exercise. 

The good news is that there's a solution to both problems in the form of the [blkio-cgroup](<http://lwn.net/Articles/330533/>) patch, also by Ryo. This patch interfaces between dm-ioband and the control group mechanism, allowing bandwidth control to be applied to arbitrary control groups. Unlike some other solutions, dm-ioband still does not use control groups for bandwidth control policy; control groups are really only used to define the groups of processes to operate on. 

The other feature added by blkio-cgroup is a mechanism by which the owner of arbitrary I/O requests can be identified. To this end, it adds some fields to the array of `page_cgroup` structures in the kernel. This array is maintained by the memory usage controller subsystem; one can think of `struct page_cgroup` as a bunch of extra stuff added into `struct page`. Unlike the latter, though, `struct page_cgroup` is normally not used in the kernel's memory management hot paths, and it's generally out of sight, so people tend not to notice when it grows. But, there is one `struct page_cgroup` for every page of memory in the system, so this is a large array. 

This array already has the means to identify the owner for any given page in the system. Or, at least, it will identify _an_ owner; there's no real attempt to track multiple owners of shared pages. The blkio-cgroup patch adds some fields to this array to make it easy to identify which control group is associated with a given page. Given that, and given that block I/O requests include the address of the memory pages involved, it is not too hard to look up a control group to associate with each request. Modules like dm-ioband can then use this information to control the bandwidth used by all requests, not just those initiated directly from user space. 

The advantages of dm-ioband include device-mapper integration (for those who use the device mapper), and a relatively small and well-contained code base - at least until blkio-cgroup is added into the mix. On the other hand, one must use the device mapper to use dm-ioband, and the scheduling decisions made there are unlikely to help the lower-level I/O scheduler implement its policy correctly. Finally, dm-ioband does not provide any sort of quality-of-service guarantees; it simply ensures that each group gets something close to a given percentage of the available I/O bandwidth. 

#### io-throttle

The [io-throttle patches](<http://lwn.net/Articles/330531/>) by Andrea Righi take a different approach. This controller uses the control group mechanism from the outset, so all of the policy parameters are set via the control group virtual filesystem. The main parameter for each control group is the maximum bandwidth that group can consume; thus, io-throttle enforces absolute bandwidth numbers, rather than dividing up the available bandwidth proportionally as is done with dm-ioband. (Incidentally, both controllers can also place limits on the number of I/O operations rather than bandwidth). There is a "watermark" value; it sets a level of utilization below which throttling will not be performed. Each control group has its own watermark, so it is possible to specify that some groups are throttled before others. 

Each control group is associated with a specific block device. If the administrator wants to set identical policies for three different devices, three control groups must still be created. But this approach does make it possible to set different policies for different devices. 

One of the more interesting design decisions with io-throttle is its placement in the I/O structure: it operates at the top, where I/O requests are initiated. This approach necessitates the placement of calls to `cgroup_io_throttle()` wherever block I/O requests might be created. So they show up in various parts of the memory management subsystem, in the filesystem readahead and writeback code, in the asynchronous I/O layer, and, of course, in the main block layer I/O submission code. This makes the io-throttle patch a bit more invasive than some others. 

There is an advantage to doing throttling at this level, though: it allows io-throttle to slow down I/O by simply causing the submitting process to sleep for a while; this is generally preferable to filling memory with queued BIO structures. Sleeping is not always possible - it's considered poor form in large parts of the virtual memory subsystem, for example - so io-throttle still has to queue I/O requests at times. 

The io-throttle code does not provide true quality of service, but it gets a little closer. If the system administrator does not over-subscribe the block device, then each group should be able to get the amount of bandwidth which has been allocated to it. This controller handles the problem of asynchronously-generated I/O requests in the same way dm-ioband does: it uses the blkio-cgroup code. 

The advantages of the io-throttle approach include relatively simple code and the ability to throttle I/O by causing processes to sleep. On the down side, operating at the I/O creation level means that hooks must be placed into a number of kernel subsystems - and maintained over time. Throttling I/O at this level may also interfere with I/O priority policies implemented at the I/O scheduler level. 

#### io-controller

Both dm-ioband and io-throttle suffer from a significant problem: they can defeat the policies (such as I/O priority) being implemented by the I/O scheduler. Given that a bandwidth control module is, for all practical purposes, an I/O scheduler in its own right, one might think that it would make sense to do bandwidth control at the I/O scheduler level. The [io-controller patches](<http://lwn.net/Articles/331857/>) by Vivek Goyal do just that. 

Io-controller provides a conceptually simple, control-group-based mechanism. Each control group is given a weight which determines its access to I/O bandwidth. Control groups are not bound to specific devices in io-controller, so the same weights apply for access to every device in the system. Once a process has been placed within a control group, it will have bandwidth allocated out of that group's weight, with no further intervention needed - at least, for any block device which uses one of the standard I/O schedulers. 

The io-controller code has been designed to work with all of the mainline I/O controllers: CFQ, Deadline, Anticipatory, and no-op. Making that work requires significant changes to those schedulers; they all need to have a hierarchical, fair-scheduling mechanism to implement the bandwidth allocation policy. The CFQ scheduler already has a single level of fair scheduling, but the io-controller code needs a second level. Essentially, one level implements the current CFQ fair queuing algorithm - including I/O priorities - while the other applies the group bandwidth limits. What this means is that bandwidth limits can be applied in a way which does not distort the other I/O scheduling decisions made by CFQ. The other I/O schedulers lack multiple queues (even at a single level), so the io-controller patch needs to add them. 

Vivek's patch starts by stripping the current multi-queue code out of CFQ, adding multiple levels to it, and making it part of the generic elevator code. That allows all of the I/O schedulers to make use of it with (relatively) little code churn. The CFQ code shrinks considerably, but the other schedulers do not grow much. Vivek, too, solves the asynchronous request problem with the blkio-cgroup code. 

This approach has the clear advantage of performing bandwidth throttling in ways consistent with the other policies implemented by the I/O scheduler. It is well contained, in that it does not require the placement of hooks in other parts of the kernel, and it does not require the use of the device mapper. On the other hand, it is by far the largest of the bandwidth controller patches, it cannot implement different policies for different devices, and it doesn't yet work reliably with all I/O schedulers. 

#### Choosing one

The proliferation of bandwidth controllers has been seen as a problem for at least the last year. There is no interest in merging multiple controllers, so, at some point, it will become necessary to pick one of them to put into the mainline. It has been hoped that the various developers involved would get together and settle on one implementation, but that has not yet happened, leading Andrew Morton to [proclaim](<https://lwn.net/Articles/332849/>) recently: 

I'm thinking we need to lock you guys in a room and come back in 15 minutes. 

Seriously, how are we to resolve this? We could lock me in a room and come back in 15 days, but there's no reason to believe that I'd emerge with the best answer. 

At the Storage and Filesystem Workshop in April, the [storage track](<http://lwn.net/Articles/328347/>) participants appear to have been leaning heavily toward a solution at the I/O scheduler level - and, thus, io-controller. The cynical among us might be tempted to point out that Vivek was in the room, while the developers of the competing offerings were not. But such people should also ask why an I/O scheduling problem should be solved at any other level. 

In any case, the developers of dm-ioband and io-throttle have not stopped their work since this workshop was held, and the wider kernel community has not yet made a decision in this area. So the picture remains only slightly less murky than before. About the only clear area of consensus would appear to be the use of blkio-cgroup for the tracking of asynchronously-generated requests. For the rest, the locked-room solution may yet prove necessary.
