---
title: The problematic kthread freezer
url: https://lwn.net/Articles/705269/
date: "November 2, 2016"
category: Kernel threads
author: "By Jonathan Corbet November 2, 2016 2016 Kernel Summit"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
November 2, 2016

* * *

[2016 Kernel Summit](<https://lwn.net/Articles/KernelSummit2016/>)

The kernel thread ("kthread") freezer, as its name would suggest, is charged with freezing kernel threads during a system hibernation cycle. At the 2016 Kernel Summit, Jiri Kosina took the stage (for the second time) to say that the usage of the kthread freezer is "out of control" and "broken everywhere." It is time, he said, to bring things under control, then get rid of the freezer altogether. 

The first problem, he said, is that the freezer's semantics are not well defined; nobody really knows what it means for a kthread to be frozen. Most of the current uses of the freezer are superfluous. In many cases, the purpose is to have filesystems be in a consistent state during hibernation; that can be better achieved with the filesystem freeze mechanism. It doesn't make sense to freeze I/O operations in general, since they are needed to write out the hibernation image. There is a lot of freezing in drivers too, a situation which, he said, makes no sense. There is a well-defined set of power-management callbacks in place to put drivers into a suspended state during hibernation. 

The kernel, he said, is the victim of a massive copy-and-paste cargo cult. Uses of the kthread freezer are spreading like a disease, a situation that has to stop. 

There are two especially pathological uses that he called out. One is `try_to_freeze()` calls for threads that have not been marked freezable in the first place; those calls will never have any effect. The other is `try_to_freeze()` calls after starting I/O, but without waiting for that I/O to complete. 

The solution is to eliminate use of the kthread freezer wherever possible. It is not needed in threads that will not generate disk I/O. It is also not needed — indeed, its use is a bug — in I/O helper threads. The best solution would be to move the entire hibernation subsystem to use filesystem freezing instead, and simply get rid of the kthread freezer. It might be necessary to keep it around for NFS, he said, but there's not much else that should need it. But the first step is to stop its use from spreading. 

Ben Herrenschmidt spent a while talking about the history of the freezer, which, he said, was invented as "a big, fat band-aid" without which the system could not suspend properly. Now, instead, we simply need to make our drivers cope properly with I/O during a suspend operation. As the session closed, Linus agreed that the best approach was to get rid of the kthread freezer altogether and to use filesystem freezing where it is really needed. So one should expect development to go in that direction.
