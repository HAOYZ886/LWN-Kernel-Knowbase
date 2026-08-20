---
title: Resource beancounters
url: https://lwn.net/Articles/197433/
date: "August 29, 2006"
category: "Beancounters; Class-based resource management; Virtualization"
author: 
---

Your editor remembers a time when "the computer" was a single, large machine shared among many users. This large machine was, one might say, not quite as powerful as the systems we work on - or carry around to play music on - today, so sharing it between dozens (or more) people was bound to lead to conflicts. Accordingly, most timesharing systems in those days implemented complex resource quota mechanisms to keep users in bounds. When these systems worked well, they let people get their work done while minimizing violence in the hallways. 

It is probably safe to say that almost all deployed Linux systems spend most of their time serving a single user or task. There is little need to keep users from stepping on each others' toes within a single system; instead, they can fight over the use of external resources like network bandwidth. So patches which implement such mechanisms (such as the [class-based kernel resource management system](<http://lwn.net/Articles/145135/>)) have generally not gotten very far. The driving need to fence users within a portion of a system's resources just has not been there. 

Virtualization and containers may change that situation, however. The purpose of these systems is to isolate users from each other. But if one container is able to use a disproportionate amount of some vital system resource, the others will feel its presence. The illusion of having a machine to one's self loses some of its credibility if that machine, say, has no memory available to it. As these projects gather steam, they are motivating another look at resource usage management structures. 

CKRM, now known as [resource groups](<http://lwn.net/Articles/181857/>), may well make a resurgence. In the mean time, however, another approach has been proposed in the form of the [resource beancounters](<http://lwn.net/Articles/197377/>) patch. The beancounter developers appear to have tried to take a lighter-weight approach, but this patch still ends up touching a number of places in the kernel. 

The core object in this mechanism is, yes, the "beancounter." Each beancounter in the system tracks the resource usage of a group of processes - presumably all of the processes running within a specific container. Beancounters contain a reference count, a unique ID, and an array of resource values; for each tracked resource, this array contains a pair of limits, current usage, historical minimum and maximum use, and a count of how many times an attempt to increase usage of that resource was denied. Each process in the system contains a pointer to its (probably shared) beancounter object. There is also a second beancounter, called `fork_bc`, which is used for any child processes created with `fork()`. 

A new system call, `get_bcid()`, returns the ID number for the current process's beancounter object. A suitably privileged user can call: 

```
int set_bcid(bcid_t id);
```

to change its current and fork IDs to a new value. Privileged processes can also change any process's limits with: 

```
int set_bclimit(bcid_t id, unsigned long resource, unsigned long *limits);
```

Here, `resource` identifies which resource limit is being changed, and `limits` points to an array of two values holding the "barrier" and "limit" values. The barrier value is intended to be a sort of soft limit, where some allocations might fail, but others are allowed to proceed. 

In the posted patch, only one resource is tracked: kernel memory. For this resource, the "barrier" limit applies to most allocations; once the barrier is hit, allocation attempts will fail. The allocation of page tables and related structures, however, can go all the way to the "limit" value. So, while a process may start to see operations failing as a result of excessive kernel memory use, it should still be able to have its page faults handled normally while it tries to recover. 

The kernel allocates memory in many places, and not all of those should be charged to the process that happens to be running at the time. The beancounter patch adds a couple of new GFP flags to make the difference explicit. In the default case, memory allocations are not charged to any specific beancounter. Whenever an allocation function is called with the `__GFP_BC` flag set, however, the current beancounter will be charged. An additional flag (`__GFP_BC_LIMIT`) specifies that the higher limit value is to be used. There is also a `SLAB_BC` flag which can cause all allocations from a given slab cache to be charged. Finally, there is a new `vmalloc_bc()` function which performs the appropriate accounting. 

Needless to say, finding every allocation which should be tracked and charged to a beancounter would be a large task. The current patch does not even try; instead, it marks enough specific allocations to catch some of the larger uses of kernel memory and show how the whole system works. That may be as far as it gets; getting driver writers, for example, to think about whether their memory allocations should be charged seems like an uphill battle. 

Whether this patch set will get any further than CKRM (sorry, "resource groups") remains to be seen. There are some concerns about how accounting for shared resources are handled - does the process group which first faults in the C library get charged for the whole thing, giving others a free ride? Then, many developers will continue to see no real need for this sort of accounting structure. The growing use of virtualization techniques may just be the factor which pushes this kind of patch into the kernel, however.
