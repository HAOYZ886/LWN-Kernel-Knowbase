---
title: "Thread-level control with resource groups"
url: https://lwn.net/Articles/679940/
date: "March 16, 2016"
category: "Control groups-Thread-level control"
author: "By Jonathan Corbet March 16, 2016"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
March 16, 2016

The kernel's [control-group mechanism](<https://lwn.net/Articles/604609/>) allows processes to be divided into groups for the purposes of tracking and resource control. Both the API and underlying implementation of this mechanism have been going through considerable change in recent years. As part of that change, the newer control-group API has lost the ability to separately manage threads within a process, a loss that is not welcome in some quarters. Current work to replace that functionality is not finding an entirely warm reception either, though. 

#### No threads need apply

Within the kernel, the distinction between a "thread" and a "process" is not entirely clear; for the most part, everything that can run is just a thread. At the user level, though, the two are seen as separate, with processes having their own address space while threads share a single address space (to oversimplify things slightly). Within the kernel, the "everything is a thread" notion led to the first version of the control-group interface (the "v1 API") managing everything at the thread level. 

When control-group maintainer Tejun Heo designed the new (v2) control-group interface, one of the key changes he made is that the individual threads that make up a process cannot be assigned to different groups; the entire process is assigned (or moved) as a unit. This restriction comes about for a couple of reasons: 

  * Many of the resources managed by control groups belong to the process as a whole; think about memory usage in a setting where all threads share the same address space, for example. Putting different threads into different control groups makes it impossible to say what the memory-usage policy actually is. Bringing in multiple controllers with overlapping responsibilities — memory and I/O bandwidth, for example — muddies the picture even further. By keeping threads together (and in a single group hierarchy as well, another change made by the v2 API) the new mechanism makes a coherent resource-management policy possible. 

  * Control groups are seen to be a mechanism by which the system administrator can manage resources, and the API is designed around that use case. Resource management for threads, though, is more of a concern for an application itself rather than the administrator. Using the same interface for both is seen as conflating two use cases, creating possible security issues, and leading to possible performance problems — an API designed for occasional use by administrators may not perform well if an application uses it to make frequent changes. 

Additionally, in the v1 API, giving a process access to a portion of the virtual filesystem tree used to manage control groups requires the intervention of an outside process, possibly leading to a situation where applications have to be written for any management scheme that distributions may adopt. Tejun sees that as a cost imposed by the kernel's failure to provide a proper interface for this type of control. 

Limiting control groups to processes steps around all of these problems without, it was thought, creating any new ones. But there are indeed use cases for managing resources at the thread level. Most of those use cases seem to be oriented around scheduling; an application may well want to use control groups to manage the division of its available CPU time among its component threads. In the v2 API, that ability has been lost. 

#### Enter resource groups

Tejun has been pondering the thread-level control problem for a while now. In early January he posted [a lengthy writeup](<http://article.gmane.org/gmane.linux.kernel.cgroups/15225>) of what he intended to do, but got no responses. That is often the case in kernel development; developers would rather see the code than a lot of words about what somebody intends to do. Besides, many were probably still recovering from their new-year hangovers on January 5\. So the real discussion had to wait for the posting of Tejun's ["resource group" mechanism patch set](<https://lwn.net/Articles/679774/>) in March. 

A "resource group" can be thought of as a special kind of control group designed for use within a process to control its own threads. They are thus distinct from "system groups," which are control groups as implemented in current kernels. Tejun uses the terms "rgroup" and "sgroup" for those two types of groups, but that leads to language like ""a top-level rgroup of a process is a rgroup whose parent cgroup is a sgroup,"" so it might be better to spell things out. 

Unlike control groups, resource groups are essentially invisible; they cannot be managed with the regular control-group API. That is done, in part, to separate the management interfaces, but it is also done because there is quite a bit less management to be done with resource groups. A process may be moved from one control group (system group) to another at any time, but resource-group membership for threads is forever. Among other things, that addresses the performance issues that can come from frequent group changes: if such changes cannot be done, the performance problems go away. 

Resource-group membership is managed, instead, at process-creation time via a new flag (`CLONE_NEWRGRP`) to the `clone()` system call. A thread thus can never change its own resource-group membership, but it can create new threads in a different group. Thus resource groups, unlike system groups, can only cover specific subtrees of the process tree. 

Controllers ordinarily operate on system groups only, but they can be made available at the resource-group level as well. Tejun's patch set does that for only one controller: the CPU (scheduling) controller. This controller is enabled on a specific resource group by calling `setpriority()` with the new `PRIO_RGRP` flag. For the top-level resource groups within a process, the priority can only be set as high as the priority of the process itself. Lower-level resource groups can have any priority, since they only affect the relative scheduling within the group itself. 

Scheduling of resource groups works the same way it does with system groups. If a given system group contains two processes and a resource group with a few processes of its own, the CPU time given to the system group will, by default, be divided in three and split equally among those two processes and that one resource group. All of the threads within the resource group will then contend for that one-third of the available CPU time allotted to the group. 

Another area of interest for thread-level control is the "cpuset" mechanism that allows threads to be restricted to specific CPUs. Tejun has chosen not to address this problem quite yet; as he said in January: ""cpuset can also benefit from thread granularity; however, the situation around cpuset is murkier, so let's stay away from it for now."" A dive into the murk will eventually become necessary, but there are enough thorny issues to deal with even without addressing the cpuset problem. 

#### On the existence of processes

Like many other aspects of the control-group refactoring, the resource-group patches are going to have to overcome some resistance before getting into the mainline kernel. Mike Galbraith [led the resistance](<https://lwn.net/Articles/679958/>) in this case by complaining that the scheduler has no notion of processes. Everything there is just a `sched_entity` structure that can represent either a thread or a (system) control group. According to Mike, pushing the "process" concept into the scheduler is not a good idea. He is particularly concerned about the inability to move threads between resource groups, citing the thread-pool use case where threads do work for different users (and thus should run in different groups) over their lifetime. Ingo Molnar [took the criticism further](<https://lwn.net/Articles/679959/>), accusing Tejun of ignoring concerns that have been expressed in the past and pushing forward with a problematic design. 

Tejun [responded](<https://lwn.net/Articles/679960/>) that the resource-group patch set is his attempt to address the concerns that have been raised, most of which had to do with the loss of thread-level management. The scheduler may not recognize processes as such, but users do. So, he said, a process is a good place to separate system-level administration from user-level control: 

Decoupling system management and in-application operations makes hierarchical resource grouping and control easily accessible to individual applications without worrying about how the system is managed in larger scope. Process is a fairly good approximation of this boundary. 

Beyond that, he said, for some resources it makes no sense to go below the process level. 

For the thread-group use case, he said, the best solution remains to put the CPU controller on a separate system-group hierarchy. That remains possible with the v1 API, and that capability is not going away anytime soon. But, he noted, doing so sacrifices the ability to manage all resources in a unified manner. In [another response](<https://lwn.net/Articles/679961/>) he described this organization as ""completely alien to how the rest of the system is organized""; such modes can be useful, and they will remain supported, but there will be costs to using them. 

Part of Mike's disagreement [appears](<https://lwn.net/Articles/679963/>) to be a desire to not allow processes to control resource management on his systems at all. As he put it: ""That's what happens when control freak meets control freak, one of them ends up in pieces. There can be only one, and that one is me, the administrator."" That view clearly runs contrary to what other users would like, though; there are constituencies out there for process-level resource-management control. 

That is where the conversation stands as of this writing, but it seems certain that the discussion is not yet done. There is a fundamental disagreement here over how control groups should work, especially with regard to how they interact with the CPU scheduler. Tejun remains confident, though, that his design meets the requirements of those who are making real use of control groups now, and who need thread-level scheduling. Thus far, there have not been any well-developed alternatives proposed.
