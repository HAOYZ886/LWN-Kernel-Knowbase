---
title: Yet another approach to software suspend
url: https://lwn.net/Articles/242107/
date: "July 18, 2007"
category: Kexec; Software suspend
author: 
---

Back in early 2006, there was an ongoing, energetic debate over the future of the software suspend (to disk) code - a situation which remains true to this day. In the middle of it all, Andrew Morton had [jumped in](<http://lwn.net/Articles/170825/>) with a suggestion for a different approach: 

If you want my cheerfully uninformed opinion, we should toss both of them out and implement suspend3, which is based on the kexec/kdump infrastructure. There's so much duplication of intent here that it's not funny. And having them separate like this weakens both in the area where the real problems are: drivers. 

Eighteen months later, it looks like we might just get that "suspend3" in the form of the [kexec jump patch](<http://lwn.net/Articles/241806/>), posted by Ying Huang. 

Ying's patch builds on the existing kdump facility. The purpose of kdump is to provide safe and useful crash dumps in situations where the state of the operating system is uncertain. If the system panics it is nice to be able to save its current state for post-mortem debugging. It is important, however, that the buggy kernel - which is now in an untrustworthy state - not be used to do dangerous things like write crash dump data to disk. To avoid that situation, a small "dump kernel" is placed in a reserved area of memory where, most of the time, it lurks unnoticed and unneeded. Should a panic occur, a `kexec()` call is made to transfer control to the dump kernel, which will be able to start up in a known state. As long as the dump kernel stays within its reserved area of memory, it will be able to write the rest of the system state to disk (or wherever) in a relatively safe way. 

What Andrew recognized last year is that suspend-to-disk (which is slowly being rebranded "hibernation") does essentially the same thing: system activity is stopped and the current system state is written to disk. If the dump kernel could read that state back into memory and return to the original kernel, it would be able to hibernate (and resume) the system. An implementation along these lines would have the advantage of unifying much of the kdump and hibernation code, thus concentrating development effort and generally simplifying things. Plus it would be a way to eliminate the current code, which, despite many years' tenure in the mainline, remains somewhat unloved. 

The current patch does not do all of that; it is really just the first step: making it possible to jump from the secondary kernel back into the original kernel. The code is relatively simple; though it does rely on much of the existing infrastructure to properly suspend and power down all devices in the system for the jump in either direction. So if device drivers are interfering with hibernation now, that problem will still exist in a kexec-based implementation. But much of the other hibernation code, including the much-maligned process freezer, would be unneeded and could be removed. 

There's a few little details to take care of before one can take a hatchet to the current hibernation code, though. Powering-down devices between the two kernels is not really necessary or desirable; they just need to go into a quiet "hibernate" state. A kdump kernel needs to be placed in reserved memory from the beginning; trying to load it at panic time would be far too late. A kernel used for hibernation, instead, need not occupy system memory all the time, so some sort of on-demand secondary kernel loading is needed. The actual task of saving and restoring the system image is yet to be implemented - that can all be done easily in user space, however, with very little in the way of kernel support. Making the resume process fast enough will take some work - users might take a dim view of having to wait for two kernels to boot before getting their system back. And so on. 

So, in other words, nobody should be holding their breath for kexec-based hibernation in the near future. But the initial response to this approach was mostly positive; there seems to be a lot of interest in simply starting over in this area. Some of that enthusiasm might fade as work progresses and it turns out that, even with a new approach, hibernation is still a difficult and somewhat grungy problem. So only time will tell if this code will develop into a better hibernation implementation.
