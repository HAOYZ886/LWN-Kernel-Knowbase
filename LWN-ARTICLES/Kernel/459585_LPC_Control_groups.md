---
title: "LPC: Control groups"
url: https://lwn.net/Articles/459585/
date: "September 20, 2011"
category: Control groups
author: "By Jonathan Corbet September 20, 2011"
---

By **Jonathan Corbet**  
September 20, 2011

Control groups remain a controversial topic in kernel circles; some developers like them, others hate them. The latter group would like to see the feature removed altogether, but that seems unlikely to happen; there are too many users for control groups already, with more to come. The 2011 Linux Plumbers Conference featured a discussion among those users that gave some insights into why control groups are useful and what could be done to make them more so. 

The session started with a brief talk by Kir Kolyshkin of Parallels; for him, control groups are all about implementing containers. Containers can be seen as a sort of poor user's virtualization; it enables the running of multiple, isolated user-space systems all on the same kernel. Containers tend to be more efficient than pure virtualization; they are also, he said, the only form of virtualization available for the ARM architecture at the moment. Control groups help in the implementation of containers by isolating groups of processes from each other and by allowing the imposition of resource limits on each group. 

The bulk of the session, though, centered around a presentation by Tim Hockin on Google's isolation and resource limitation needs. Google's cluster runs all kinds of jobs which, internally, are divided into "tier 1" and "tier 2" tasks. The general problem Google has is that tasks normally do not use 100% of the resources they request; that [![\[Tim
Hockin\]](https://static.lwn.net/images/conf/2011/lpc/TimHockin-sm.jpg)](<https://lwn.net/Articles/459606/>) means that systems in the cluster tend to be underutilized. Google would like to be able to pack more jobs onto each box, but they have to be very careful about overcommitting resources. If that is not done carefully, resource-intensive jobs can get in the way of urgent tasks like responding to search queries. 

Google uses its own form of containers to be able to overcommit systems safely. Containers let Google place limits on the CPU usage, memory usage, I/O bandwidth consumption, etc. of each group of processes on the system. The goal, when all goes well, is to isolate each group from the others, provide predictable resources to each, and to lose very little time on the container implementation itself. Control groups are used when they are available and suitable to the task; in other places, a lot of user-space control code is used instead. The user-space code is complex and racy, Tim said; they would like to be rid of it. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

There is a special daemon running on each system that wakes up about every 100ms to have a look at what is going on. Should it detect a load spike originating from the system's tier-1 work, it will stop or kill any tier-2 tasks needed to make room. This all works, but it could work better; more support from the kernel would be helpful. 

For example, memory use needs to be tightly controlled on these systems. At the moment, Google is using the "fake NUMA" feature to partition system memory and parcel it out as needed (see [this article](<https://lwn.net/Articles/436871/>) for a bit more information on how that works). Fake NUMA is a hack, though, with resource costs of its own. They are moving to the kernel's memory controller, but it is not yet suitable for their needs because it cannot work with nested control groups. They had similar problems with the disk bandwidth controller, but that problem [has been resolved](<https://lwn.net/Articles/427961/>) recently. In general, Tim said, anybody who is designing a controller for Linux should think about how it will nest from the beginning. 

One other problem with the memory controller is its handling of shared memory. Currently shared pages are billed to the control group that touches it first. That makes deterministic resource control hard, especially in situations where the limits are set tightly. Tim didn't like the idea of proportional billing (dividing the charge for each page across each group that has it mapped) any better. That, he said, takes memory billing out of the control of each group; if one control group exits, the others will suddenly find themselves over their limits as their portion of the shared pages grows. What he would like would be the ability to manually arrange for pages backed by certain files to be billed to specific groups. Then he could set up a system group to be billed for, say, the C library. 

There are some other problems as well. The memory overhead of the memory controller is painfully high, for example. Google would really like a way to query the size of the working set for each control group, but that capability is not currently there. They also really want per-control-group reclaim to focus the memory management code on the control groups that are currently exceeding their limits. And, if a container goes so far over its limits that the out-of-memory killer gets involved, it would be really nice to have a way to kill a whole control group at once instead of having to do it one process at a time. (It's worth noting that patches for many of these features exist; many of them come from Google). 

Beyond that, there is a lot of interest in the I/O bandwidth controller. A lot of Google jobs, he said, are "seek locked"; controlling how much I/O bandwidth they use is important. Controllers for other types of resources (number of threads, number of open file descriptors, network ports, etc.) would be useful. And so on. 

The session spent some time on other topics - primarily [user-space checkpoint/restart](<https://lwn.net/Articles/452184/>). It was agreed that everybody in the room was interested in better isolation, and that the memory controller was the area in need of the most work at the moment. The session was dominated by users of control groups, though; there were not a lot of implementers present. Even more notable in their absence were those developers who are opposed to control groups in their current form; it would have been interesting to hear their ideas about how the needs expressed there should really be met.
