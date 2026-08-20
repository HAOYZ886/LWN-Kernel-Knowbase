---
title: "Robust futexes - a new approach"
url: https://lwn.net/Articles/172149/
date: "February 15, 2006"
category: Futex
author: 
---

One of the many features added during the 2.5 development series was the "futex" - a sort of fast, user-space mutual exclusion primitive. In the non-contended case, futexes can be obtained and released with no kernel involvement at all, making them quite fast. When contention does happen (one process tries to obtain a futex currently owned by another), the kernel is called in to queue any waiting processes and wake them up when the futex becomes available. When queueing is not needed, however, the kernel maintains no knowledge of the futex, keeping its overhead low. 

There is one problem with keeping the kernel out of the picture, however. If a process comes to an untimely end while holding a futex, there is no way to release that futex and let other processes know about the problem. The SYSV semaphore mechanism - a much more heavyweight facility - has an "undo" mechanism which can be called into play in this sort of situation, but there is no such provision for futexes. As a result, a few different "robust futex" patches have been put together over the past years; LWN [looked at one of them](<http://lwn.net/Articles/67194/>) in January, 2004\. These patches have tended to greatly increase the cost of futexes, however, and none have been accepted into the mainline. 

Ingo Molnar, working with Thomas Gleixner and Ulrich Drepper, has tossed aside those years' worth of work and, in a couple of days, produced [a new robust futex patch](<http://lwn.net/Articles/172134/>) which, he hopes, will find its way into the mainline. The new patch has the advantage of being fast, but, as Ingo notes: 

Be warned though - the patchset does things we normally dont do in Linux, so some might find the approach disturbing. Parental advice recommended ;-) 

The fundamental problem to solve is that the kernel must, somehow, know about all futexes held by an exiting process in order to release them. A past solution has been the addition of a system call to notify the kernel of lock acquisitions and releases. That approach defeats one of the main features of futexes - their speed. It also adds a record-keeping and resource limiting problem to the kernel, and suffers from some problematic race conditions. 

So Ingo's patch takes a different approach. A list of held futexes is maintained for each thread, but that list lives in user space. All the thread has to do is to make a single call to a new system call: 

```
long set_robust_list(struct robust_list_head *head, size_t size);
```

That call informs the kernel of the location of a linked list of held futexes in the calling process's address space; there is also a `get_robust_list()` call for retrieving that information. Typically, this call would be made by glibc, and never seen by the application. Glibc would also take on the task of maintaining the list of futexes. 

When a process dies, the kernel looks for a pointer to a user-space futex list. Should that pointer be found, the kernel will carefully walk through it, bearing in mind that, as a user-space data structure, it could be accidentally or maliciously corrupt. For each held futex, the kernel will release the lock and set it to a special value indicating that the previous holder made a less-than-graceful exit. It will then wake a waiting process, if one exists. That process will be able to see that it has obtained the lock under dubious circumstances (user-space functions like `pthread_mutex_lock()` are able to return that information) and take whatever action it deems to be necessary. The kernel will release a maximum of one million locks; that keeps the kernel from looping forever on a circular list. Given the practical difficulties of making a million-lock application work at all, that limit should not constrain anybody for quite some time. 

There is still a race condition here: if a process dies between the time it acquires a lock and when it updates the list, that lock might not be released by the kernel. Getting around that problem involves a bit of poor kernel hacker's journaling. The head of the held futex list contains a single-entry field which can be used to point to a lock which the application is about to acquire. The kernel will check that field on exit, and, if it points to a lock actually held by the application, that lock will be released with the others. So, if glibc sets that field before acquiring a lock (and clears it after the list is updated), all locks held by the application will be covered. 

The discussion on this patch was just beginning when this article was written. There is some concern about having the kernel walking through user-space data structures; the chances of trouble and security problems are certainly higher when that is going on. Other issues may yet come up as well. But, since this is clearly not a 2.6.16 feature in any case, there will be time to talk about them.
