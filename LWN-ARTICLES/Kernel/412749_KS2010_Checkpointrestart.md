---
title: "KS2010: Checkpoint/restart"
url: https://lwn.net/Articles/412749/
date: "November 2, 2010"
category: Checkpointing
author: "By Jonathan Corbet November 2, 2010 2010 Kernel Summit"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
November 2, 2010

* * *

[2010 Kernel Summit](<https://lwn.net/Articles/KernelSummit2010/>)

The Kernel Summit discussion on the out-of-tree code problem deliberately avoided checkpoint/restart for a simple reason: a separate session had been scheduled to talk about that subsystem. Checkpoint developer Oren Laadan gave an overview of the code and led a discussion about its prospects for inclusion into the mainline kernel. He left with a clear set of next steps for getting this feature merged. 

Checkpoint/restart allows the state of a set of processes to be saved to persistent storage, then restarted at some future time, possibly on a different system. It has a number of potential uses, including fault-tolerant systems, debugging (it's a sort of "super core dump"), fast application startup, testing, and as a kind of "generic time machine." That last one allows for the important use case of checkpointing a game, then restoring it after a move which proves to be a mistake. Checkpoint/restart can also be used as a sort of application-level suspend feature; it can function as a kind of "smart swap" which can move an application entirely out of memory when the need arises. There is also the interesting prospect of saving a desktop session on a USB key, then restarting it on an entirely different system in a different location. 

There is a lot of interest in this feature. NCSA is incorporating it into its [BlueWaters](<http://www.ncsa.illinois.edu/BlueWaters/>) supercomputer. Canonical is also working it into its Ubuntu Enterprise Cloud distribution. 

In the past, concerns have been raised about the long-term maintainability of the checkpoint/restart subsystem, so Oren spent a fair amount of time talking about that topic. It currently consists of about 100 patches, with about 23,000 lines of code spread throughout the kernel tree. There is a big test suite which goes along with it and which can catch many kinds of regressions. In general, Oren says, keeping up with kernel changes has not been a big problem; it's mostly a matter of supporting new kernel features. Maintaining it in the long term will require that developers understand it well enough at least to understand when a change will affect checkpoint/restore code and notify the maintainers. 

Andrew Morton said that this feature will impact many subsystems in the kernel. It's not something which can be imposed through a big, central merge. So, he asked the room to consider whether, as a whole, the feature is worth the cost that will come with merging it into the mainline. 

Tony Luck tried to get a sense for the limits of the code by asking what types of processes will never be supported for checkpointing. The answer was that anything which works directly with hardware will be hard to checkpoint properly, so they don't plan to try. There are also some parts of `/proc` which will remain forever off-limits. 

James Bottomley asked what the smallest useful patch would be. After all, 100 patches is an awful lot to dump on the community; it's unlikely that such a set will ever be reviewed. Oren said that the project started with a minimal series some years ago; the response they got was that it looked like an interesting toy, but that the developers wanted to see what the whole solution would look like. Eventually they got to something which looks like a reasonably complete implementation and they are now hearing that it's too large and they should post something minimal instead. Ted Ts'o responded that a minimal patch makes more sense now that the larger implementation is available to anybody who wants to look at it. 

Andrew said that, once we start merging the checkpoint/restart code, we're committing ourselves to the whole series. So there may not be any real point to starting with a minimal patch. The only reason to go that way would be limitations on how much code can be reviewed at any given time. 

Linus asked Oren to start with a small patch which is focused mainly on changes to existing code within the kernel. Oren will post this patch, possibly within the week, and the discussion on merging should begin in earnest. 

[Next: Lightning talks](<https://lwn.net/Articles/412750/>).
