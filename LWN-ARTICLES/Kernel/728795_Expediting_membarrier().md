---
title: Expediting membarrier()
url: https://lwn.net/Articles/728795/
date: "July 26, 2017"
category: membarrier
author: "By Jonathan Corbet July 26, 2017"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
July 26, 2017

The [`membarrier()` system call](<http://man7.org/linux/man-pages/man2/membarrier.2.html>) is arguably one of the strangest offered by the Linux kernel. It expensively emulates an operation that can be performed by a single unprivileged barrier instruction, using an invocation of the kernel's read-copy-update (RCU) machinery — all in the name of performance. But, it would seem, `membarrier()` is not fast enough, causing users to fall back to complex and brittle tricks. An attempt to fix the problem is now under discussion, but not everybody is convinced that the cure is better than the disease. 

#### membarrier()

`membarrier()` was first [discussed](<https://lwn.net/Articles/369567/>) in 2010. The initial use case was to support user-space RCU, which uses a shared-memory variable to indicate that a thread is running in an RCU critical section. Changes to RCU-protected objects (and, in particular, the freeing of the old version of a changed object) cannot happen while any thread is in an RCU critical section, so code that performs such an operation must check this shared flag to ensure that the change is safe. This scheme can be thwarted, though, if the processor reorders operations, causing the object to be freed _before_ the variable is checked. 

Processors provide memory-barrier instructions so that this kind of scenario can be prevented. Unfortunately, these instructions are relatively slow, since they must serialize access across the entire machine. Memory barriers must also occur in pairs to function properly; in this case, one barrier would be needed whenever a thread sets the "in RCU critical section" flag, while the other would happen after that flag is checked, but before any subsequent action is taken. This symmetric pairing of barriers works well in many situations, but it is poorly suited to the RCU use case in particular. 

The problem comes from the fact that entry into an RCU critical section is a frequent occurrence, while changes to RCU-protected objects can be quite rare. So it is possible that hundreds (or more) `rcu_read_lock()` calls will be made where no thread is trying to change the protected objects; in such cases, all of the overhead incurred by those memory barriers is wasted. In situations where this sort of asymmetrical access pattern pertains, it would be worthwhile to greatly increase the cost of a memory-barrier operation — if that cost could be moved entirely to the thread performing the change, allowing the read path to be fast. 

That is where `membarrier()` comes in. The initial version simply sent an inter-processor interrupt (IPI) causing every processor to execute a memory-barrier instruction. That approach was not entirely popular, since the IPIs wake every processor on the system and can cause unexpected latencies for realtime threads. Subsequent discussion caused the implementation to shift to calling `synchronize_sched()`, a kernel function that, among other things, ensures that every processor will have executed a memory barrier. At the time, the patches included an "expedited" option that would use IPIs instead, but when [`membarrier()` was merged](<https://git.kernel.org/linus/5b25b13ab08f616efd566347d809b4ece54570d1>) (many years later, in 2015), that option was not included. 

#### The expedited option

Recently, Paul McKenney posted [a patch](<https://lwn.net/Articles/728802/>) adding the expedited option back to `membarrier()`. This change raised some eyebrows, since the concerns about IPIs have not gone away. Mathieu Desnoyers, the original author of the `membarrier()` patch, [asked](<https://lwn.net/Articles/728803/>) how it was possible to offer the expedited option without impacting realtime processes, and Peter Zijlstra [worried](<https://lwn.net/Articles/728804/>) about the denial-of-service attack that can be carried out by code as simple as: 

```
for (;;)
            membarrier(MEMBARRIER_CMD_SHARED_EXPEDITED, 0);
```

At the moment, it would seem, there are no new answers to any of those questions, but there is a stronger incentive to add the expedited option, and appears that this option is not creating any problems that do not already exist. 

As McKenney [described it](<https://lwn.net/Articles/728805/>), there are a number of users who are finding that the existing `membarrier()` system call is too slow. That is perhaps unsurprising; `synchronize_sched()` will force the calling thread to block until every CPU in the system goes through an RCU grace period, so there is a certain amount of latency built in. These users have found a trick to get the desired behavior without calling `membarrier()`: they make a call to either [`mprotect()`](<http://man7.org/linux/man-pages/man2/mprotect.2.html>) or [`munmap()`](<http://man7.org/linux/man-pages/man2/mmap.2.html>) instead. Either of those system calls will, on an x86 system, cause an IPI to be issued to ensure that the affected address ranges are removed from each translation lookaside buffer (TLB). They also cause a certain amount of useless memory-management overhead but, evidently, the end result is still faster than calling `membarrier()`. 

Besides its fundamental inelegance, this approach has a couple of problems. One is that it could easily break in future kernels or on future hardware if those system calls can be made to work without IPIs; if such an optimization opportunity presents itself, the kernel developers are highly likely to take it. In fact, the IPIs are not necessary on all current hardware, leading McKenney to note that this trick ""has the slight disadvantage of not working at all on arm and arm64"". Adding the IPI capability to `membarrier()` will allow for better performance on all architectures without the need to resort to tricks. 

Since users can already create IPIs at will with the memory-management calls, McKenney does not believe that adding that ability to `membarrier()` will make things worse. But there are, he [said](<https://lwn.net/Articles/728808/>), a few things that could be done to reduce the potential for abuse of the expedited option. These range from complete ""defanging"" by disabling expedited grace periods at boot time to limiting the number of expedited `membarrier()` calls that can be made in a given time period. Various approaches to limiting the IPIs to the processors that actually need to receive them (those processors actually running threads from the application calling `membarrier()`) are also under consideration. Providing a mechanism for expedited barriers will, at least, give the kernel community the possibility of handling any abuse. 

This is a patch that is likely to go through further revisions and discussion before it makes it close to the mainline. Among other things, the people who have been calling for a faster `membarrier()` need to verify that the expedited option solves their problem. ""Obviously, unless there are good test results and some level of user enthusiasm, this patch goes nowhere"", McKenney said. The actual code, at the moment, fits on a single screen; the discussion around it seems unlikely to be so concise.
