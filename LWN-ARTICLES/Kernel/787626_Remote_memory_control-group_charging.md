---
title: "Remote memory control-group charging"
url: https://lwn.net/Articles/787626/
date: "May 7, 2019"
category: "Control groups-Memory controller; Memory management-Control groups"
author: "By Jonathan Corbet May 7, 2019 LSFMM"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
May 7, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

Memory control groups exist to track and limit the amount of memory used by sets of processes. Normally, one would not expect that memory used by one group would be charged to another but, as Shakeel Butt described in a memory-management track session at the 2019 Linux Storage, Filesystem, and Memory-Management Summit, that does happen in a number of different situations. It's often a problem, but occasionally it's also a useful feature. 

One of the most common examples of this sort of "remote charging" happens when a page of memory shared by more than one group is swapped out. When that page is swapped back in, it will be charged to the group that originally allocated it, even if that group has long since stopped using it. That can lead to surprising situations like a page fault in one group triggering reclaim or even an out-of-memory (OOM) response in another group. Remote charging can happen when [`userfaultfd()`](<http://man7.org/linux/man-pages/man2/userfaultfd.2.html>) is in use; the fault-handling process can be charged for memory used by the controlled process. It is seen when processes are traced with `ptrace()` and can even happen explicitly with a call to `[get_user_pages_remote()](<https://elixir.bootlin.com/linux/v5.0/source/mm/gup.c#L1045>)`. At times, memory allocated in kernel space can also be remotely charged. 

Michal Hocko asked about `userfaultfd()` in particular, wondering how often the control process runs in a different group than the processes it is handling faults for. Butt responded that he didn't know, but that the remote charging is documented. Hocko suggested that running in different groups was probably not a good idea in general, and said that he would be reluctant to complicate the code for a theoretical use case that may not happen in the real world. 

One of the in-kernel cases mentioned by Butt was the allocation of buffer heads, which can happen when writeback is triggered on a page. That can happen in the global reclaim process, as well as elsewhere. Those buffer heads will be charged to the control group that is being charged for the page being written back, even if that group had nothing to do with the activity that made writeback necessary. 

One interesting quirk of remote charging is that it bypasses the `memory.high` control knob, which defines when an allocating process should be throttled. That throttling will not happen on allocations that are remotely charged. Rik van Riel asked how often this problem comes about; Butt didn't have a precise answer but said that he had seen it. Hocko said that if 1% of allocations are remotely charged in this way it really doesn't matter; the processes within the control group will eventually be charged directly for a page and the throttling will happen. In more unbalanced cases it could be a problem, though. 

In some cases there might actually be a use case for remote charging, though. Some users want to separate virtualized workloads, with the virtual-machine monitor running in a different group than the virtual machines themselves. This is evidently done mostly as a way of knowing what the monitor overhead is. The way this is done currently is to create a tmpfs filesystem and mount it in its own control group; all pages allocated will then be charged to that group. There's just one little problem: if the virtual machine dies (from the OOM killer, for example) the tmpfs filesystem will remain, consuming memory. To deal with that, the kernel has been hacked to send a special notification on OOM kills so that somebody can go and clean up the tmpfs files. 

Butt described a new idea that he has been working on: the tmpfs files could be attached to a special dummy process that is subject to the out-of-memory killer's attention. When the process is killed, the files are truncated; it's essentially an OOM-killable tmpfs file. Johannes Weiner said that such a mechanism was far too specialized to go upstream. Van Riel, though, suggested that it might be possible to upstream a patch implementing a new mount flag for tmpfs. If the control group attached to a given mount is killed, the filesystem would automatically drop its contents. 

Hugh Dickins said that problems with OOM kills and tmpfs are common, so it would be good to have some sort of a solution there. Weiner said that perhaps the best place to implement it would be in the [oomd](<https://github.com/facebookincubator/oomd>) user-space OOM daemon. That kind of policy is hard to put into the kernel's OOM killer, which exists primarily to protect the machine as a whole. The implementation of specific resource-control policies, perhaps, is better placed in a user-space process. At that point, the session wound down.
