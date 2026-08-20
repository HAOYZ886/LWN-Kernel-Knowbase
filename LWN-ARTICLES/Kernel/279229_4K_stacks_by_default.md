---
title: "4K stacks by default?"
url: https://lwn.net/Articles/279229/
date: "April 23, 2008"
category: Kernel stack
author: "By Jake Edge April 23, 2008"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
April 23, 2008

The kernel stack is a rather important chunk of memory in any Linux system. The unpleasant kernel memory corruption that results from overflowing it is something that is to be avoided at all costs. But the stack is allocated for each process and thread in the system, so those who are looking to reduce memory usage target the 8K stack used by default on x86. In addition, an 8K stack requires two physically contiguous pages (an "order 1" allocation) which can be difficult to satisfy on a running system due to fragmentation. 

Linux has had optional support for 4K stacks for nearly four years now, with Fedora and RHEL enabling it on the kernels they ship, but a recent patch to make it the default for x86 has raised some eyebrows. Andrew Morton sees it as [bypassing the normal patch submission process](<https://lwn.net/Articles/279238/>): 

This patch will cause kernels to crash. 

It has no changelog which explains or justifies the alteration. 

afaict the patch was not posted to the mailing list and was not discussed or reviewed. 

It is not surprising that patch author Ingo Molnar [sees things a little differently](<https://lwn.net/Articles/279278/>): 

what mainline kernels crash and how will they crash? Fedora and other distros have had 4K stacks enabled for years [ ... ] and we've conducted tens of thousands of bootup tests with all sorts of drivers and kernel options enabled and have yet to see a single crash due to 4K stacks. So basically the kernel default just follows the common distro default now. (distros and users can still disable it) 

As described in an earlier [LWN article](<http://lwn.net/Articles/150580/>), the main concerns about only providing 4K for the kernel stack are for complicated storage configurations or for people using [NDISwrapper](<http://ndiswrapper.sourceforge.net/>). There is fairly high disdain for the latter case--as it is done to load proprietary Windows drivers into the kernel--but it could lead to a pretty hideous failure in the former. Data corruption certainly seems like a possibility, but, regardless, a kernel crash is definitely not what an administrator wants to have to deal with. 

Arjan van de Ven [summarized the current state](<https://lwn.net/Articles/279302/>), noting that NDISwrapper really requires 12K stacks, so having 8K only makes it less likely those kernels will crash. The stacking of multiple storage drivers (network filesystems, device mapper, RAID, etc.) is a bigger issue: 

we need to know which they are, and then solve them, because even on x86-64 with 8k stacks they can be a problem (just because the stack frames are bigger, although not quite double, there). 

Proponents of default 4K stacks seem to be puzzled why there is objection to the change since there have been no problems with Red Hat kernels. But Andi Kleen [notes](<https://lwn.net/Articles/279340/>): 

One way they do that is by marking significant parts of the kernel unsupported. I don't think that's an option for mainline. 

The xfs filesystem, which is not supported in RHEL or Fedora, can potentially use a great deal of stack. This leads some kernel hackers to worry that a complicated configuration that uses it, an "nfs+xfs+md+scsi writeback" configuration as Eric Sandeen [puts it](<https://lwn.net/Articles/279350/>), could overflow. Work is already proceeding to reduce the xfs stack usage, but it clearly is a problem that xfs hackers have seen. David Chinner [responds to a question about stack overflows](<https://lwn.net/Articles/279352/>): 

We see them regularly enough on x86 to know that the first question to any strange crash is "are you using 4k stacks?". In comparison, I have never heard of a single stack overflow on x86_64.... 

It would seem premature to make 4K stacks the default. There is good reason to believe that folks using xfs could run into problems. But there is a larger issue, one that Morton brought up in his initial message, then [reiterated later in the thread](<https://lwn.net/Articles/279355/>): 

Anyway. We should be having this sort of discussion _before_ a patch gets merged, no? 

The memory savings can be significant, especially in the embedded world. Coupled with the elimination of order 1 allocations each time a process gets created, there is good reason to keep working toward 4K stacks by default. As of this writing, the default remains for 4K stacks in Linus's tree, but that could change before long.
