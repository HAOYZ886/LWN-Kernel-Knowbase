---
title: Restartable sequences
url: https://lwn.net/Articles/662946/
date: "November 4, 2015"
category: Restartable sequences
author: "By Jonathan Corbet November 4, 2015 2015 Kernel Summit"
---

By **Jonathan Corbet**  
November 4, 2015

* * *

[2015 Kernel Summit](<https://lwn.net/Articles/KernelSummit2015/>)

As computers incorporate more processors, the concurrency concerns that were once mostly limited to the kernel are pushing out into user space. So user-space developers are increasingly wanting to use many of the techniques found in the kernel for concurrency management. Per-CPU variables are of interest, because they avoid contention between processors, but there is a catch: the kernel's per-CPU variables depend on the ability to disable preemption to serialize access — an ability that user space lacks. An alternative approach is thus needed; one such is [restartable sequences](<https://lwn.net/Articles/650333/>), which were covered here in July. At the 2015 Kernel Summit, Andy Lutomirski and Paul Turner led a session about whether support for restartable sequences should be added to the kernel. 

Some workloads found at Google make use of per-thread free lists for the `malloc()` function, Paul said. This technique performs well, but it also eats up a lot of memory; that has led to an interest in using per-CPU free lists instead. The idea is to let threads detect if they have been interrupted while in a critical section and, if so, restart their operation from the beginning. Restartable sequences allow this kind of [![\[Paul and Andy\]](https://static.lwn.net/images/conf/2015/klf-ks/Paul+Andy-sm.jpg)](<https://lwn.net/Articles/662955/>) pattern with no locking and with no need for atomic variables. Paul suggested that it might prove useful for realtime developers as well. 

Andy then said that he really didn't like the early attempts at support for restartable sequences. He likes it when debuggers work and context switches have sane semantics; the patches ran counter to both of those. He also was not a fan of accessing user-space memory during scheduling. The work has progressed, though, and could benefit from more review. In particular, there is an interest in having non-x86 developers look at the patches to see whether this functionality could be supported on their architectures. 

Chris Mason noted that using restartable sequences cuts memory usage by 20% in a workload he has looked at; he described it as "a big deal." David Howells asked what was required from the kernel to support this functionality. Andy's answer was that user space needs to be able to register a critical section with the kernel. If a process is interrupted while executing within that region, it jumps to a specific recovery address when it resumes executing. That recovery code can then do whatever is needed, which usually is a matter of just restarting the operation from the beginning. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Ben Herrenschmidt asked whether it was possible to register more than one critical section; that would be important for libraries to be able to use this facility. The answer was that critical sections can be nested, so library use should be possible. 

Paul noted that [a new patch series](<https://lwn.net/Articles/662228/>) had been posted that morning. Are there, he asked, any objections to the concept in general or to the patches in particular? 

Andy responded that he still doesn't like the idea of context switches having side effects. The current patches seem to be getting better in that regard. Josh Triplett noted that restartable sequences could be useful for timing sections of code; Paul agreed, and said they could be used for user-space read-copy-update implementations as well. In general, objections were scarce, but the real proof will be in how and when the patches are accepted.
