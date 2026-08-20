---
title: "Reviving linux-tiny"
url: https://lwn.net/Articles/251573/
date: "September 26, 2007"
category: Embedded systems
author: "By Jake Edge September 26, 2007"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
September 26, 2007

An [announcement of the revival of linux-tiny](<http://lwn.net/Articles/250533/>), a set of patches aimed at reducing the footprint of the kernel, mainly for the embedded world, has led to a number of linux-kernel threads. The conversations range from the proper place for linux-tiny to reside to the removal of the enormous number of `printk()` strings in the kernel. They provide an interesting glimpse into the kernel development process. 

The linux-tiny project was [started by Matt Mackall](<http://lwn.net/Articles/62858/>) in December 2003 with the aim to ""collect patches that reduce kernel disk and memory footprint as well as tools for working on small systems."" LWN [covered the announcement](<http://lwn.net/Articles/63516/>) at the time and [tried out the patches](<http://lwn.net/Articles/191955/>) more than a year ago. Many of the linux-tiny features have found their way into the mainline, but quite a few still remain outside. 

The Consumer Electronics Linux Forum (CELF) is behind the effort to revive the project, with Tim Bird, architecture group chair, announcing the plan, including a new maintainer, Michael Opdenacker. The first step has been mostly completed, bringing the patches forward from the 2.6.14 kernel to 2.6.22. A [status page](<http://elinux.org/Linux_Tiny_Patch_Details>) has been established to track the progress of updating the patches, but it is clear that moving them into the mainline, rather than maintaining them as patches, is a big motivation behind the revival. 

Andrew Morton immediately volunteered to manage the linux-tiny patches in an [answer to the revival message](<http://lwn.net/Articles/251626/>): 

Seriously, putting this stuff into some private patch collection should be a complete last resort - you should only do this with patches which you (and the rest of us) agree have no hope of ever getting into mainline. 

Reactions were quite favorable, with the maintainer, Opdenacker [responding](<http://lwn.net/Articles/251627/>): 

Andrew, you're completely right... The patches should all aim at being included into mainline or die. 

I'm finishing a sequence of crazy weeks and I will have time to send you patches one by one next week, starting with the easiest ones. 

The full patchset will live in a separate repository as the individual patches are being worked on for inclusion, but it is clear that no one wants to continuously maintain and out-of-tree patchset for a long time. The cost of ensuring that the patches do not bitrot is large and their inclusion in the mainline will get them in the hands of more developers. 

From there, more detailed discussion of how to structure the patches - and tiny features in general - ensued. A separate discussion also came about regarding `printk()` and the large amounts of memory it consumes with all of its static strings. `printk()` has long been seen as an area that could be improved to reduce the memory footprint of the kernel. 

All sorts of kernel messages are printed to logfiles or the console via `printk()`; there are something on the order of 60,000 calls in 2.6. There can be a severity level associated with a specific call, which provides a primitive syslog-style categorization of the messages. Unfortunately, in the mainline, those calls are either present, with all the associated memory for the strings, or completely absent, compiled out via a config option. It is rather difficult to diagnose problems without at least some `printk()` information, but keeping all of the data in can increase the size of the kernel 5-10%. 

Rob Landley [started things off](<http://lwn.net/Articles/251655/>) with a way to make it possible to only compile in messages based on their severity level. An embedded developer could remove `KERN_NOTICE`, `KERN_DEBUG` and similar low severity messages while keeping the more critical messages: 

[...] the compiler's dead code eliminator zaps the printks you don't care about so they don't bloat the kernel image. But this doesn't _completely_ eliminate printks, so you can still get the panic() calls and such. You tweak precisely how much bloat you want, using the granularity information that's already there in the source code. 

Landley's suggestion has a drawback in that it would require a flag day for `printk()` or the creation of a new function that implemented his suggestion with relevant changes trickling into the kernel over time. In the meantime, small-system developers would still be looking for ways to get the messages they want, while removing the others from the code. There was also discussion of using separate calls for each severity level, where `pr_info()`, or some similar name, would produce messages with that level. The preprocessor could then be used to remove those that a developer is not interested in. 

The discussion led Vegard Nossum to put together an RFC for a [new kernel-message logging API](<http://lwn.net/Articles/251650/>). He starts with requirements that the API be backwards-compatible with the existing `printk()` usage, with the output format being extensible at either compile or run time. The RFC also tries to handle the case of multiple `printk()` calls to emit what is essentially a single message, but it seems like an over-engineered solution to what should be a fairly straightforward problem. 

Another contender, one that is already part of the linux-tiny patchset, is Tim Bird's [DoPrintk](<http://tree.celinuxforum.org/CelfPubWiki/DoPrintk>) patch. This allows developers to selectively choose source code files for which `printk()` will be enabled, removing it from the rest of the code and resulting kernel image. While not allowing fine-grained selection of messages based on severity, it does put more control into the hands of developers. 

It is too early to say which, if any, `printk()` changes are coming down the pike. There does seem to be a lot of interest in helping small systems reduce their kernel footprint without sacrificing all diagnostic messages. `printk()` is claimed to be one of the lowest hanging fruit for significant kernel size reduction, which would seem to make it a likely candidate for change.
