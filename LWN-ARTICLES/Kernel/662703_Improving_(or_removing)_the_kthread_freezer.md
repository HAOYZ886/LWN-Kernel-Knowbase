---
title: Improving (or removing) the kthread freezer
url: https://lwn.net/Articles/662703/
date: "November 2, 2015"
category: Kernel threads
author: "By Jonathan Corbet November 2, 2015 2015 Kernel Summit"
---

By **Jonathan Corbet**  
November 2, 2015

* * *

[2015 Kernel Summit](<https://lwn.net/Articles/KernelSummit2015/>)

The process freezer (or just "freezer") holds processes in a suspended state; it is used, for example, when suspending or hibernating the system. Jiri Kosina started his 2015 Kernel Summit session by noting that he had thought he could use the freezer for live patching as well. But, in the process of making that work, he found out that the semantics of the freezer are not particularly well defined; there is no definition of what it means for a process to be "frozen." As a result, the implementation of frozen behavior varies and, in the case of a number of kernel threads ("kthreads"), things are broken entirely. 

The original purpose of the freezer was to get processes out of the way so that the system could be suspended and to ensure that no unwanted I/O activity causes filesystem corruption. The freezer also prevents user space from taking kernel locks. Kernel threads, being processes, also are [![\[Jiri Kosina\]](https://static.lwn.net/images/conf/2015/klf-ks/JiriKosina-sm.jpg)](<https://lwn.net/Articles/662704/>) subject to freezing unless they have the special `PF_NOFREEZE` flag set. A freezable kthread that fails to freeze when requested can block the system from suspending or hibernating. 

One of the first things Jiri noticed was that kthreads will call `try_to_freeze()` (an indication that this would be a good time to freeze the thread if needed) without having first called `set_freezable()` (which clears the `PF_NOFREEZE` flag). Such threads can never be frozen, so the `try_to_freeze()` call will always fail. That suggested to him that the freezer for kthreads is overused and unneeded. Some kthreads (those that participate in the I/O needed to suspend or hibernate the system) cannot be frozen anyway. For the rest, the important part is to get them to the point where they schedule, and that is happening anyway without `try_to_freeze()`. 

Jiri said that the arguments for freezing kernel threads do not hold much water when examined. There are some fears that a kthread could hold a lock and deadlock the hibernation process; if that were true, he said, it would deadlock the system anyway. The other reason to freeze kthreads is to prevent them from initiating I/O, but it is, in fact, important for kthread I/O to succeed while the system is changing state. So, rather than worrying about freezing kthreads, Jiri suggested, why not just restrict the freezer to user-space processes? It is unclear why the freezer even exists for kthreads; it appears to serve no purpose and is easy to get wrong. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

An alternative idea, Jiri said, would be to annotate I/O that is needed for the suspend or hibernate process to complete. Only specially marked requests would be allowed through; the rest would be blocked until the system is resumed. But, he said, this approach seems complex, fragile, and just as error-prone as what's there now. 

The best solution, he said, might be to get rid of the kthread freezer entirely. Its main reason for existing is to ensure that filesystems do not get corrupted when the system suspends and resumes (or fails to resume). That could be just as easily accomplished by freezing the filesystems instead. The virtual filesystem layer already has support for freezing filesystems, and the freeze operation ensures that the filesystems are consistent on disk. Freezing filesystems would also have the advantage of preventing bootloaders from trying to replay the journal — something that overly clever bootloaders evidently do — because there would be no journal to replay. 

There was a quick consensus in the room that abolishing the freezer looked like the right way to go. Thus, chances are, we'll see patches toward that goal in a future development cycle.
