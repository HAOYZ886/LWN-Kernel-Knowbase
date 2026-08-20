---
title: Stopping unwanted OOM killer experiences
url: https://lwn.net/Articles/111408/
date: "November 17, 2004"
category: "Memory management-Token-based thrashing control; OOM killer"
author: 
---

There has, in recent times, been a small increase in the number of complaints from users who have seen processes killed by the kernel in response to an out-of-memory (OOM) situation. The only problem is that the system should not have been quite that hard up for memory at the time. Even if the user is doing something which requires completely irrational amounts of memory ("`yum update`", say), it seems like the system should have been able to muddle along without killing low-priority processes, like the ssh server. These unwanted OOM killer experiences have driven a few developers to take a closer look at what was going on. 

Marcelo Tosatti has been working on the problem for a bit; he put together [a patch](<https://lwn.net/Articles/111409/>) which tries to avoid invocations of the OOM killer if things might get better soon. The idea is that, while a full scan of a memory zone may have failed to turn up any free pages, it may have kicked I/O into motion that will, very soon, make some pages free. So the OOM killer is kept in its cage until the no-memory situation has persisted for a few seconds. Marcelo reported that this patch improved things significantly for his test cases. 

It turns out, though, that the real problem was elsewhere; the [token-based thrashing control patch](<https://lwn.net/Articles/96621/>) appears to be the real culprit. This patch, remember, tries to reduce system thrashing in memory-constrained situations by exempting one process at a time from the page reclaim mechanism. That process will, in theory, make use of its sheltered time to make some real progress before the token moves on and its pages are, once again, subject to eviction. The token-based mechanism has been shown to truly improve the situation when memory is tight. 

Until it gets too tight, as it turns out. A process which needs a page, but which does not hold the token, may find that all of the (otherwise) reclaimable pages belong to the process currently holding the token. The unlucky process thus finds no pages to grab, and pushes the big red OOM button. The system is not truly out of memory, however; it has simply been told that all the good pages are temporarily off limits. 

Rik van Riel put his finger on the problem, and Andrew Morton put together [a simple patch](<https://lwn.net/Articles/111417/>) to fix it. Essentially, the VM subsystem will now ignore the swap token when finding reclaimable pages gets too hard. During normal operation, the token-based mechanism holds sway, but it can be set aside as a preferable alternative to killing random processes in the system. The patch appears to have solved the problems without taking away the benefits of the token-based approach. 

Marcelo acknowledged that this was the right fix, grumbled that he had wasted a bunch of time, and [promised](<https://lwn.net/Articles/111418/>) ""Next time I should be looking into the easy stuff before trying miraculous solutions."" It was his work, however, which shone a light on the problem in the first place, and led to its eventual solution.
