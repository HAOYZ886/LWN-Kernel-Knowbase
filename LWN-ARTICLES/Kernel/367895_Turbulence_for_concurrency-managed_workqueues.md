---
title: "Turbulence for concurrency-managed workqueues"
url: https://lwn.net/Articles/367895/
date: "December 23, 2009"
category: Workqueues
author: "By Jonathan Corbet December 23, 2009"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
December 23, 2009

Tejun Heo's [concurrency-managed workqueues patch](<http://lwn.net/Articles/355700/>) was discussed here in October. This work has been progressing, with some of the associated clean-up patches being merged for 2.6.33; the main part of the work would appear to be on a path for merging in 2.6.34. Or maybe not: some developers are starting to express some qualms. 

The loudest complaints come from Peter Zijlstra, who would rather see effort go into converting workqueue users to using threaded interrupt handlers instead. To developers like Peter, the new workqueues look like a bunch of new complexity which could create new problems (management of CPU-intensive workqueue tasks, for example) while failing to address other issues, including the locking problems which can plague workqueue users now. 

Tejun has [responded](<https://lwn.net/Articles/367896/>) with a description of some of the problems being solved by the redone workqueues, concluding: 

Shifting complexity out of peripheral code to better crafted and managed core code is the right thing to do and it will shift a lot of complexity out of peripheral codes. 

That may actually be where some of the trouble lies: the patch set, in its current state, does not really demonstrate this shift in complexity. So Ingo Molnar has [requested](<https://lwn.net/Articles/367898/>) some example conversions that show the advantages of concurrency-managed workqueues: 

For this particular patchset it should be possible to identify existing patterns of code in the existing code base of 6+ millions lines of Linux driver code that would make the advantages of this +2000 lines of core kernel code plain obvious. There were multiple claims of problems with the current abstractions - so there sure must be a way to show off the new code in a few places. 

Tejun has indicated that he will work to provide this demonstration. Should the next version of the patch set prove convincing on this front, the new workqueues might still be on-track for 2.6.34.
