---
title: "KS2007: Realtime and syslets"
url: https://lwn.net/Articles/248929/
date: "September 9, 2007"
category: Realtime; Syslets
author: "By Jonathan Corbet September 9, 2007 LWN.net Kernel Summit 2007 coverage"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
September 9, 2007

* * *

LWN.net [Kernel Summit 2007](<https://lwn.net/Articles/KernelSummit2007/>) coverage 

Thomas Gleixner, Ingo Molnar, and Zach Brown ran a brief session on scheduler-related topics - the realtime patches and syslets in particular. On the real-time side, the bulk of the work has been merged at this point, with a lot of the patches (dynamic tick, mutexes, etc.) proving to be useful far beyond the realtime arena. Perhaps the most useful part so far, says Ingo, has been the [lock validator](<http://lwn.net/Articles/185666/>), a tool which enables the checking of locking patterns in search of potential deadlocks. 

The biggest pending pieces are threaded interrupt handlers, sleeping spinlocks, and preemptible read-copy-update. The developers would like to get those technologies merged soon, but they do not believe that most users will ever want to turn them on. Linus noted that, while he does not like threaded interrupt handlers in general, they do have one big advantage: they can make debugging interrupt-related problems easier. If an interrupt handler crashes in a current kernel, it brings the whole kernel down with it. Pushing handlers into their own threads makes it possible for the kernel to continue after a crash (possibly in a degraded mode) so that the problem can be investigated. Ben Herrenschmidt claimed that threaded interrupt handlers make life easier for driver writers in general, and should maybe be the default. It will always be possible to turn off threading for situations where direct interrupt handling is required, but most drivers probably do not require that ability. 

The other realtime issue which came up was resource guarantees. It seems that there is demand for this feature, but that it is not really being worked on yet. Resource guarantees will probably be implemented through the container framework once that project gets started. 

On the syslets side, Zach Brown characterized the technology as "promising but unproven." The basic code works, but there are a lot of loose ends which need to be tied up before it will be anywhere near ready for consideration for the mainline. More testing would also help, but Zach says that not many people are trying this patch set at the moment. 

Ingo said that one of the real advantages of syslets it that it enables the implementation of just about any sort of concurrency model one might like to try. If somebody wants the VMS I/O model, it's just a matter of a bit of hacking. Your editor, who still has all twelve `sys$qio()` arguments burned into his brain, finds this prospect a little worrisome - though the ability to handle _any_ I/O operation asynchronously would certainly be helpful in some situations. 

Before that can happen, though, there are a number of places in the kernel which must be fixed to support full concurrency. The direct I/O code is one example, as are readahead and the CFQ I/O scheduler, which currently gets confused by syslets. These parts of the kernel simply do not expect one process to have multiple operations going simultaneously; fixing them up could be a significant bit of work. There may also be some performance issues in the syslet code, but there has not, yet, been a lot of real measurement work done.
