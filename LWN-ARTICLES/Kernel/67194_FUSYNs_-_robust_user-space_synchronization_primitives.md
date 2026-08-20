---
title: "FUSYNs - robust user-space synchronization primitives"
url: https://lwn.net/Articles/67194/
date: "January 21, 2004"
category: FUSYN
author: 
---

The FUTEX subsystem, which is part of the 2.6 kernel, provides fast mutual exclusion primitives for user space. The FUTEX functionality is similar to that of the longstanding semaphores, but with a nicer interface and better performance. A FUTEX lock can be acquired (in the non-contention case) without going into the kernel at all. FUTEXes are a part of the high-performance native POSIX threading implementation. 

FUTEXes are an improvement on what came before, but they do not yet provide the functionality that some users - particularly real-time system implementers - would like to have. To help fill in the gap, Iñaky Pérez-González has been working (with others) on a new set of "robust mutexes" which go by the name of FUSYNs. The project has [a simple web site](<http://developer.osdl.org/dev/robustmutexes/>) based at OSDL and a set of patches. Some information can be found in [fusyn.txt](<https://lwn.net/Articles/67195/>), which is included with the patch. 

FUSYNs enhance FUTEXes with: 

  * Priority-based locks. When a lock is released, it is not handed over to a random process. Instead, the highest-priority process waiting for the lock will be allowed to proceed. If a process changes priority while waiting for a lock, the system will take the change into account properly. 

  * Priority inheritance. Processes which take out FUSYN locks ("fulocks") can have their priority raised to a specified level while they hold the lock. This mechanism is an attempt to avoid priority inversion problems, where a low-priority process can obtain a lock, lose the processor, and keep a high-priority process from running for a long time. 

  * Robustness features. The kernel can take remedial action when a process dies while holding a lock. There is also deadlock protection code which looks at the chains of locks held by various processes and reacts when a deadlock situation is detected. 

Future plans include the addition of features like condition variables, reader/writer locks, spinlocks, etc. 

Inside the kernel, this functionality is implemented through the addition of some new facilities which could be useful beyond the FUSYN code. The "vlocator" structure allows the kernel to associate objects with user-space processes via a hash table. In the longer term, vlocators could be used to provide some relief for the ever-growing task structure. The unfortunately-named "fuqueue" functions much like an ordinary kernel wait queue, except that wakeups take process priority into account - only the highest-priority process is awakened. To support this functionality, a new "plist" type is added; it implements a general, priority-sorted, doubly-linked list capability. 

The reaction to posts of FUSYN patches on linux-kernel has tended to be quiet. There does not appear to be any strong opposition to the addition of this capability to the kernel. Whether FUSYNs go into 2.6, or have to wait for 2.7, however, remains to be seen.
