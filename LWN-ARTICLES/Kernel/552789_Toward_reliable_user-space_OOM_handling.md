---
title: "Toward reliable user-space OOM handling"
url: https://lwn.net/Articles/552789/
date: "June 5, 2013"
category: "Memory management-Out-of-memory handling; OOM killer"
author: "By Jonathan Corbet June 5, 2013"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
June 5, 2013

A visit from the kernel's out-of-memory (OOM) killer is usually about as welcome as a surprise encounter with the tax collector. The OOM killer is called in when the system runs out of memory and cannot progress without killing off one or more processes; it is the embodiment of a frequently-changing set of heuristics describing which processes can be killed for maximum memory-freeing effect and minimal damage to the system as a whole. One would not think that this would be a job that is amenable to handling in user space, but there are some users who try to do exactly that, with some success. That said, user-space OOM handling is not as safe as some users would like, but there is not much consensus on how to make it more robust. 

#### User-space OOM handling

The heaviest user of user-space OOM handling, perhaps, is Google. Due to the company's desire to get the most out of its hardware, Google's internal users tend to be packed tightly into their servers. Memory control groups (memcgs) are used to keep those users from stepping on each others' toes. Like the system as a whole, a memcg can go into the OOM condition, and the kernel responds in the same way: the OOM killer wakes up and starts killing processes in the affected group. But, since an OOM situation in a memcg does not threaten the stability of the system as a whole, the kernel allows a bit of flexibility in how those situations are handled. Memcg-level OOM killing can be disabled altogether, and there is a mechanism by which a user-space process can request notification when a memcg hits the OOM wall. 

Said notification mechanism is designed around the needs of a global, presumably privileged process that manages a bunch of memcgs on the system; that process can respond by raising memory limits, moving processes to different groups, or doing some targeted process killing of its own. But Google's use case turns out to be a little different: each internal Google user is given the ability (and responsibility) to handle OOM conditions within that user's groups. This approach can work, but there are a couple of traps that make it less reliable than some might like. 

One of those is that, since users are doing their own OOM handling, the OOM handler process itself will be running within the affected memcg and will be subject to the same memory allocation constraints. So if the handler needs to allocate memory while responding to an OOM problem, it will block and be put on the list of processes waiting for the OOM situation to be resolved; this is, essentially, a deadlocking of the entire memcg. One can try to avoid this problem by locking pages into memory and such, but, in the end, it is quite hard to write a user-space program that is guaranteed not to cause memory allocations in the kernel. Simply reading a `/proc` file to get a handle on the situation can be enough to bring things to a halt. 

The other problem is that the process whose allocation puts the memcg into an OOM condition in the first place may be running fairly deeply within the kernel and may hold any number of locks when it is made to wait. The `mmap_sem` semaphore seems to be especially problematic, since it is often held in situations where memory is being allocated — page fault handling, for example. If the OOM handler process needs to do anything that might acquire any of the same locks, it will block waiting for exactly the wrong process, once again creating a deadlock. 

The end result is that user-space OOM killing is not 100% reliable and, arguably, can never be. As far as Google is concerned, somewhat unreliable OOM handling is acceptable, but deadlocks when OOM killing goes bad are not. So, back in 2011, David Rientjes [posted a patch](<https://lwn.net/Articles/432223/>) establishing a user-configurable OOM killer delay. With that delay set, the (kernel) OOM killer will wait for the specified time for an OOM situation to be resolved by the user-space handler before it steps in and starts killing off processes. This mechanism gives the user-space handler a window within which it can try to work things out; should it deadlock or otherwise fail to get the job done in time, the kernel will take over. 

David's patch was not merged at that time; the general sentiment seemed to be that it was just a workaround for user-space bugs that would be better fixed at the source. At the time, David said that Google would carry the patch internally if need be, but that he thought others would want the same functionality as the use of memcgs increased. More than two years later, he is [trying again](<https://lwn.net/Articles/552749/>), but the response is not necessarily any friendlier this time around. 

#### Alternatives to delays

Some developers responded that running the OOM handler within the control group it manages is a case of "don't do that," but, once David explained that users are doing their own OOM handling, they seemed to back down a bit on that one. There does still seem to still be a bit of a sentiment that the OOM handler should be locked into memory and should avoid performing memory allocations. In particular, OOM time seems a bit late to be reading `/proc` files to get a picture of which processes are running in the system. The alternative, though, is to trace process creation in each memcg, which has performance issues of its own. 

Some constructive thoughts came from Johannes Weiner, who had a couple of ideas for improving the current situation. One of those was [a patch](<https://lwn.net/Articles/552795/>) intended to solve the problem of processes waiting for OOM resolution while holding an arbitrary set of locks. This patch makes two changes, the first of which comes into play when a problematic allocation is the direct result of a system call. In this case, the allocating process will not be placed in the OOM wait queue at all; instead, the system call will simply fail with an ENOMEM error. This solves most of the problem, but at a cost: system calls that might once have worked will start returning an error code that applications might not be expecting. That could cause strange behavior, and, given that the OOM situation is rare, such behavior could be hard to uncover with testing. 

The other part of the patch changes the page fault path. In this case, just failing with ENOMEM is not really an option; that would result in the death of the faulting process. So the page fault code is changed to make a note of the fact that it hit an OOM situation and return; once the call stack has been unwound and any locks are released, it will wait for resolution of the OOM problem. With these changes in place, most (or all) of the lock-related deadlock problems should hopefully go away. 

That doesn't solve the other problem, though: if the OOM handler itself tries to allocate memory, it will be put on the waiting list with everybody else and the memcg will still deadlock. To address this issue, Johannes [suggested](<https://lwn.net/Articles/552796/>) that the user-space OOM handler could more formally declare its role to the kernel. Then, when a process runs into an OOM problem, the kernel can check if it's the OOM handler process; in that case, the kernel OOM handler would be invoked immediately to deal with the situation. The end result in this case would be the same as with the timeout, but it would happen immediately, with no need to wait. 

Michal Hocko favors Johannes's changes, but had [an additional suggestion](<https://lwn.net/Articles/552872/>): implement a global watchdog process. This process would receive OOM notifications at the same time the user's handler does; it would then start a timer and wait for the OOM situation to be resolved. If time runs out, the watchdog would kill the user's handler and re-enable kernel-provided OOM handling in the affected memcg. In his view, the problem can be handled in user space, so that's where the solution should be. 

With some combination of these changes, it is possible that the problems with user-space OOM-handler deadlocks will be solved. In that case, perhaps, Google's delay mechanism will no longer be needed. Of course, that will not be the end of the OOM-handling discussion; as far as your editor can tell, that particular debate is endless.
