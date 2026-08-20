---
title: Finishing out the 4.10 merge window
url: https://lwn.net/Articles/710493/
date: "January 3, 2017"
category: "Releases-4.10"
author: "By Jonathan Corbet January 3, 2017"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
January 3, 2017

As expected, the 4.10 merge window ended on December 25 with the [4.10-rc1](<https://lwn.net/Articles/710185/>) release. In the end, 11,455 non-merge changesets were pulled into the mainline repository for 4.10, making this a reasonably busy development cycle, even if it falls far short of 4.9. Less than 400 of those changes were pulled after [the December 22 summary](<https://lwn.net/Articles/709556/>) was written, so the list of additional changes is short. 

That list includes: 

  * The PA-RISC architecture has gained support for kernel address-space layout randomization. 

  * The [cache-allocation technology](<https://lwn.net/Articles/694800/>) patch set has been merged. These patches provide access to a new mechanism in Intel processors by which the processor's memory cache can be partitioned between processes. It can be used to keep one group of processes (a container, say) from dominating the cache, or to set aside a portion of the cache for a set of privileged processes. 

  * There are new drivers for QLogic QEDI 25/40/100Gb iSCSI initiators and Loongson1 SoC hardware watchdogs. 

  * The `cycle_t` type used for clock values inside the kernel has been removed; a plain `u64` type is now used instead. 

The 4.10 stabilization period got off to a slow start due to the holidays; only 27 non-merge changesets were applied between 4.10-rc1 and 4.10-rc2. The pace of change can be expected to pick up, though, as developers return to work and the final 4.10 release date (probably February 12 or 19) approaches.
