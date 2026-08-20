---
title: Directed yield
url: https://lwn.net/Articles/419961/
date: "December 15, 2010"
category: Scheduler; Virtualization
author: "By Jonathan Corbet December 15, 2010"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
December 15, 2010

Contemporary CPUs have an interesting feature: they can detect when a virtualized guest is spinning on a lock and trap back into the host kernel. The idea is that the host can probably find better things to do with a processor than dedicate it to spinning in a single guest. KVM will currently respond to such a trap by sleeping briefly, allowing some other process to run outside of the virtualized system in question. But, as Rik van Riel [pointed out](<https://lwn.net/Articles/419755/>), that is not necessarily the right thing to do. 

If one thread in a virtualized system is spinning on a lock, another thread within that system must be holding the lock. Rather than pause the entire guest, it is better to run the lock-holding thread so that the lock can be released. Pausing the guest just delays the release of the lock, causing the virtual machine as a whole to be penalized; that, says Rik, ""results in eg. a 64 VCPU Windows guest taking forever and a bit to boot up."" Tempting as it may be to just blame Windows, it's probably better to fix the problem. 

Rik's fix is to change how the trap handler behaves; rather than yield the CPU entirely, it will take a spinning thread's remaining CPU time slice and give it to a process on another CPU. The hope is that the recipient of this gift (which is essentially a priority boost) will be the one holding the lock, but there is no real guarantee of that currently. This functionality is implemented with a new `yield_to()` function which, Rik says, could maybe be turned into a system call if it turns out to be useful in that context. 

The patch has been through a couple of rounds of review and may find its way into 2.6.38.
