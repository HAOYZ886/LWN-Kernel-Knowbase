---
title: Changing the default shared memory limits
url: https://lwn.net/Articles/595638/
date: "April 23, 2014"
category: "Development model-User-space ABI"
author: "By Nathan Willis April 23, 2014"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Nathan Willis**  
April 23, 2014

The Linux kernel's System V shared-memory limit has, by default, been fixed at the same value since its inception. Although users can increase this limit, as the amount of memory expected by modern applications has risen over the years, the question has become whether or not it makes sense to simply increase the default setting--including the option of removing the limit altogether. But, as is often the case, it turns out that there are users who have come to expect the shared-memory limit to behave in a particular way, so altering it would produce unintended consequences. Thus, even though no one seems happy with the default setting as it is, how exactly to fix it is not simple. 

System V-style shared memory (SHM) is commonly used as an interprocess communication resource; a set of cooperating processes (such as database sessions) can share a segment of memory up to the maximum size allowed by the operating system. That limit that can be expressed in terms of bytes per shared segment (`SHMMAX`), and in terms of the total number of pages used for all SHM segments (`SHMALL`). On Linux, the default value of `SHMMAX` has always been set at 32MB, and the default value of `SHMALL` is defined as: 

```
#define SHMALL (SHMMAX/getpagesize()*(SHMMNI/16))
```

where `SHMMNI` is the maximum number of SHM segments--4096 by default--which in turn gives a default `SHMALL` of 2097152 pages. Though they have well-known defaults, both `SHMMAX` and `SHMALL` can be adjusted with `sysctl`. There is also a related parameter setting the minimum size of a shared segment (`SHMMIN`); unlike the other resource limits, it is set to to one byte and cannot be changed. 

While most users seem to agree that one byte is a reasonable minimum segment size, the same cannot be said for `SHMMAX`; 32MB does not go far for today's resource-hungry processes. In fact, it has been routine procedure for several years for users to increase `SHMMAX` on production systems, and it is standard practice to [recommend](<http://rhaas.blogspot.com/2012/06/absurd-shared-memory-limits.html>) increasing the limit for most of the popular applications that make use of SHM. 

Naturally, many in the community have speculated that it is high time to bump the limit up to some more suitable value, and on March 31, Davidlohr Bueso posted [a patch](<https://lwn.net/Articles/595668/>) that increased `SHMMAX` to 128MB. Bueso admitted that the size of the increase was an essentially arbitrary choice (a four-fold bump), but [noted](<https://lwn.net/Articles/595669/>) in the ensuing discussion that, in practice, users will probably prefer to make their own choice for `SHMMAX` as a percentage of the total system RAM; bumping up the default merely offers a more sensible starting point for contemporary hardware. 

But Andrew Morton [argued](<https://lwn.net/Articles/595872/>) that increasing the size of the default parameter did not address the underlying issue--that users were frequently hitting what was, fundamentally, an artificial limit with no real reason behind it: 

Look. The 32M thing is causing problems. Arbitrarily increasing the arbitrary 32M to an arbitrary 128M won't fix anything - we still have the problem. Think bigger, please: how can we make this problem go away for ever? 

One way to make the problem go away forever would be to eliminate `SHMMAX` entirely, but as was pointed out in the discussion, administrators probably do want to be able to set _some_ limit to ensure that no user creates a SHM segment that eats up all of the system memory. Motohiro Kosaki [suggested](<https://lwn.net/Articles/595674/>) setting the default to zero, to stand for "unlimited." Bueso then [adopted](<https://lwn.net/Articles/595676/>) that approach for the second version of his patch. Since `SHMMIN` is hardcoded to one, the reasoning goes, `SHMMAX` cannot ever be misinterpreted by users as a valid value--either it is the default ("unlimited"), or it is the result of an overflow. 

The updated patch also set the default value of `SHMALL` to zero--again representing "unlimited". But removing the limit on the total amount of SHM in this manner revealed a second wrinkle: as Manfred Spraul [pointed out](<https://lwn.net/Articles/595786/>), setting `SHMALL` to zero is currently a move that system administrators (quite reasonably) use to disable SHM allocations entirely; the patch has the unwanted effect of completely reversing the outcome of that move--enabling unlimited SHM allocation. 

Spraul subsequently wrote his own alternative patch set that attempts to avoid this issue by instead [setting the defaults](<https://lwn.net/Articles/595787/>) for `SHMMAX` and `SHMALL` to `ULONG_MAX`, which amounts to setting them to infinity. This solution is not without its risks, either; in particular there are [known](<http://marc.info/?l=linux-mm&m=139638334330127>) cases where an application simply tries to increment the value `SHMMAX` rather than setting it, which causes an overflow. The result would be that applications would encounter the wrong value for `SHMMAX`--most likely a value far smaller than they need, causing their SHM allocation attempts to fail. 

Nevertheless, Bueso concurred that avoiding the reversal of behavior for manually setting `SHMALL` to zero was a good thing, and signed off on Spraul's approach. The latest [version](<https://lwn.net/Articles/595791/>) of Spraul's patch set attempts to avoid the overflow issue by using `ULONG_MAX - 1L<<24` instead, but he admits that ultimately there is nothing preventing users from causing overflows when left to their own devices. 

One final concern stemming from this change is that if a system implements no upper limits on SHM allocation, it will be possible for users to consume all of the available memory as SHM segments. If such a greedy allocation happens, however, the out-of-memory (OOM) killer will not be able to free that memory. The solution is for administrators to either enable the `shm_rmid_forced` option (which forces each SHM segment to be created with the `IPC_RMID` flag--guaranteeing that it is associated with at least one process, which in turn ensures that when the OOM killer kills the guilty process, the SHM segment vanishes with it) or to manually set SHM limits. 

Since the desire to avoid manually configuring SHM limits was the original goal of the effort, it might seem as if the effort has come full circle. But, for the vast majority of users, removing the ancient defaults is a welcome improvement. Rogue users attempting to allocate all of the memory in a shared segment are at best an anomaly (and certainly something that administrators should stay on the lookout for), whereas the old default 32MB SHM size has long been problematic for a wide variety of users in need of shared memory.
