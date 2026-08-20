---
title: Checkpoint/restart (mostly) in user space
url: https://lwn.net/Articles/452184/
date: "July 19, 2011"
category: Checkpointing; Containers
author: "By Jonathan Corbet July 19, 2011"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
July 19, 2011

There are numerous use cases for a checkpoint/restart capability in the kernel, but the highest level of interest continues to come from the containers area. There is clear value in being able to save the complete state of a container to a disk file and restarting that container's execution at some future time, possibly on a different machine. The kernel-based checkpoint/restart patch has been discussed here a number of times, including [a report](<https://lwn.net/Articles/412749/>) from last year's Kernel Summit and [a followup](<https://lwn.net/Articles/414264/>) published shortly thereafter. In the end, the developers of this patch do not seem to have been able to convince the kernel community that the complexity of the patch is manageable and that the feature is worth merging. 

As a result, there has been relatively little news from the checkpoint/restart community in recent months. That has changed, though, with the posting of [a new patch](<https://lwn.net/Articles/451916/>) by Pavel Emelyanov. Previous patches have implemented the entire checkpoint/restart process in the kernel, with the result that the patches added a lot of seemingly fragile (though the developers dispute that assessment) code into the kernel. Pavel's approach, instead, is focused on simplicity and doing as much as possible in user space. 

Pavel notes in the patch introduction that almost all of the information needed to checkpoint a simple process tree can already be found in `/proc`; he just needs to augment that information a bit. So his patch set adds some relevant information there: 

  * There is a new `/proc/_pid_ /mfd` directory containing information about files mapped into the process's address space. Each virtual memory area is represented by a symbolic link whose name is the area's starting virtual address and whose target is the mapped file. The bulk of this information already exists in `/proc/_pid_ /maps`, but the `mfd` directory collects it in a useful format and makes it possible for a checkpoint program to be sure it can open the exact same file that the process has mapped. 

  * `/proc/_pid_ /status` is enhanced with a line listing all of the process's children. Again, that is information which could be obtained in other ways, but having it in one spot makes life easier. 

  * The big change is the addition of a `/proc/_pid_ /dump` file. A process reading this file will obtain the information about the process which is not otherwise available: primarily the contents of the CPU registers and its anonymous memory. 

The `dump` file has an interesting format: it looks like a new binary executable format to the kernel. Another patch in Pavel's series implements the necessary logic to execute a "program" represented in that format; it restores the register and memory contents, then resumes executing where the process was before it was checkpointed. This approach eliminates the need to add any sort of special system call to restart a process. 

There is need for one other bit of support, though: checkpointed processes may become very confused if they are restarted with a different process ID than they had before. Various enhancements to (or replacements for) the `clone()` system call have been proposed to deal with this problem in the past. Pavel's answer is a new flag to `clone()`, called `CLONE_CHILD_USEPID`, which allows the parent process to request that a specific PID be used. 

With this much support, Pavel is able to create a set of tools which can checkpoint and restart simple trees of processes. There are numerous things which are not handled; the list would include network connections, SYSV IPC, security contexts, and more. Presumably, if this patch set looks like it can be merged into the mainline, support for other types of objects can be added. Whether adding that support would cause the size and complexity of the patch to grow to the point where it rivals its predecessors remains to be seen. 

Thus far, there has been little discussion of this patch set. The fact that it was posted to the containers list - not the largest or most active list in our community - will have something to do with that. The few comments which have been posted have been positive, though. If this patch is to go forward, it will need to be sent to a larger list where a wider group of developers will have the opportunity to review it. Then we'll be able to restart the whole discussion for real - and maybe actually get a solution into the kernel this time.
