---
title: "In defense of per-BDI writeback"
url: https://lwn.net/Articles/354851/
date: "September 30, 2009"
category: "Block layer-Writeback; Memory management-Writeback"
author: "By Jonathan Corbet September 30, 2009"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
September 30, 2009

Last week's [quotes of the week](<http://lwn.net/Articles/353253/>) included a complaint from Andrew Morton about the replacement of the writeback code in 2.6.32. According to Andrew, a bunch of critical code had been redone, replacing a well-tested implementation with new code without any hard justification. It's a complaint which should be taken seriously; replacing the writeback code has the potential to introduce performance regressions for specific workloads. It should not be done without a solid reason. 

Chris Mason has [tried to provide that justification](<https://lwn.net/Articles/354852/>) with a combination of benchmark results and explanations. The benchmarks show a clear - and large - performance improvement from the use of per-BDI writeback. That is good, but does not, by itself, justify the switch to per-BDI writeback; Andrew had suggested that the older code was slower as the result of performance regressions introduced over time by other changes. If the 2.6.31 code could be fixed, the performance improvement could be (re)gained without replacing the entire subsystem. 

What Chris is saying is that the old, per-CPU pdflush method could not be fixed. The fundamental problem with pdflush is that it would back off when the backing device appeared to be congested. But congestion is easy to cause, and no other part of the system backs off in the same way. So pdflush could end up not doing writeback for significant periods of time. Forcing all other writers to back off in the face of congestion could improve things, but that would be a big change which doesn't address the other problem: congestion-based backoff can defeat attempts by filesystem code and the block layer to write large, contiguous segments to disk. 

As it happens, there is a more general throttling mechanism already built into the block layer: the finite number of outstanding requests allowed for any specific device. Once requests are exhausted, threads generating block I/O operations are forced to wait until request slots become free again. Pdflush cannot use this mechanism, though, because it must perform writeback to multiple devices at once; it cannot block on request allocation. A per-device writeback thread _can_ block there, though, since it will not affect I/O to any other device. The per-BDI patch creates these per-device threads and, as a result, it is able to keep devices busier. That, it seems, is why the old writeback code needed to be replaced instead of patched.
