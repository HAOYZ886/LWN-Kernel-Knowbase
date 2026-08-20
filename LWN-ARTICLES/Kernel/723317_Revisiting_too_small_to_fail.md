---
title: "Revisiting \"too small to fail\""
url: https://lwn.net/Articles/723317/
date: "May 20, 2017"
category: "Memory management-Page allocator"
author: "By Jonathan Corbet May 20, 2017"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 20, 2017

Back in 2014, the [revelation](<https://lwn.net/Articles/627419/>) that the kernel's memory-management subsystem would not allow relatively small allocation requests to fail created a bit of a stir. The discussion has settled down since then, but the "too small to fail" rule still clearly creates a certain amount of confusion in the kernel community, as is evidenced by a recent discussion inspired by the 4.12 merge window. It would appear that the rule remains in effect, but developers are asked to act as if it did not. 

At the start of the 2014 discussion, memory-management developer Michal Hocko [described](<https://lwn.net/Articles/627434/>) the ""unwritten rule"" that small allocations never fail. "Small" is determined by the kernel's `PAGE_ALLOC_COSTLY_ORDER` constant, which is generally set to three; that puts the threshold at eight pages, or 32KB on most systems. Almost all memory allocations in the kernel are smaller than that (much effort has gone into keeping most of them no larger than a single page), so the end result is that memory allocation attempts almost never fail. 

That created some unhappiness for a couple of reasons. One is that kernel developers have been told since the beginning that any memory allocation can fail, so they have been carefully writing failure-recovery paths that will never be used. This policy can also lead the kernel to do unpleasant things — such as summoning the dreaded out-of-memory killer — rather than fail a request, even if the requesting code is prepared to deal gracefully with an allocation failure. Proposals to change this policy have always foundered on the fear that enabling allocation failures would expose bugs throughout the kernel. The bulk of that failure-recovery code may have never been executed — or it may not exist at all. So the "too small to fail" behavior remains in place. 

Trond Myklebust's [NFS client fixes pull request](<https://lwn.net/Articles/723318/>) included a line item reading: ""Cleanup and removal of some memory failure paths now that GFP_NOFS is guaranteed to never fail"". The description was inaccurate: the code in question is using a mempool, which pre-allocates memory and, if used properly, can indeed guarantee that allocation failures will not occur. But it was enough to prompt Nikolay Borisov to [ask](<https://lwn.net/Articles/723319/>) whether success was truly guaranteed. If so, there would be an opportunity to clean up a lot of unneeded error-handling code throughout the kernel. Hocko [replied](<https://lwn.net/Articles/723320/>) that, while ""small allocations never fail _practically_"", the behavior was in no way guaranteed and that removing checks for allocation failures is ""just wrong"". 

Myklebust was not entirely pleased with that response; he [asked](<https://lwn.net/Articles/723321/>) for a clear statement that small allocation requests can fail. He didn't get one. Instead, Hocko [replied:](<https://lwn.net/Articles/723322/>)

We would really like to fail those requests instead. I've tried that in the past but it was deemed too dangerous because _all_ kernel paths would have to be checked for a sane failure behavior. So we are keeping status quo instead. 

The status quo — telling developers to be prepared for allocation failures while not actually failing allocation requests — is less than pleasing for many involved in these discussions. In many parts of the kernel, error handling makes up a large portion of the total amount of code. This code can be tricky to write and even trickier to test; it can be frustrating to be asked to do this work to prepare for a situation that is not ever going to happen. 

The memory-management developers cannot just change this behavior, though. There can be little doubt that, in a kernel with thousands of never-executed, never-tested error-handling paths, some of those paths will contain bugs. Auditing the kernel and validating all of those paths would not be a small task, to put it lightly; it may not be feasible to do at all. What _can_ be done is to validate and fix the code one piece at a time. This is how the big kernel lock (BKL) was finally [removed](<https://lwn.net/Articles/424657/>) in 2011. That job proceeded by getting rid of the BKL dependencies in one small bit of code at a time until, eventually, nothing needed it anymore. It took many years, but it got the job done. 

In the case of memory-allocation failures, validating code will not always be easy. The [fault injection framework](<https://lwn.net/Articles/209257/>) can be used to force allocation errors, though, which can help in the testing of recovery paths. For code that is deemed to be properly prepared, the no-fail behavior can be turned off in any given allocation request by adding the `__GFP_NORETRY` flag; this has been done for roughly 100 allocation calls in the 4.12-rc1 kernel. Whether that flag will spread to larger parts of the kernel remains to be seen; as with the BKL removal, it will probably require the help of a group of developers who are willing to put a lot of time into the task. 

The kernel community makes internal API changes on a regular basis; most of the time, it is a simple matter of a bunch of editing work or a [Coccinelle](<https://lwn.net/Articles/315686/>) script. But subtle semantic changes are harder, and eliminating the too-small-to-fail behavior certainly qualifies as that kind of change. The longer it remains, the more entrenched it is likely to become, but there are no signs that it will be able to change anytime soon.
