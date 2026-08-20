---
title: Implementing alignment guarantees for kmalloc()
url: https://lwn.net/Articles/802469/
date: "October 18, 2019"
category: "Memory management-Internal API"
author: "October 18, 2019 This article was contributed by Marta Rybczyńska"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

October 18, 2019

This article was contributed by Marta Rybczyńska

`kmalloc()` is a frequently used primitive for the allocation of small objects in the kernel. During the [2019 Linux Storage, Filesystem, and Memory Management Summit](<https://lwn.net/Articles/lsfmm2019/>), Vlastimil Babka [led a session](<https://lwn.net/Articles/787740/>) about the unexpected alignment problems developers face when using this function. After a few months he has come back with [the second version of a patch set](<https://lwn.net/ml/linux-kernel/20190826111627.7505-1-vbabka@suse.cz/>) implementing a natural alignment guarantee for `kmalloc()`. From the strong opposition it faced initially, it seemed that the change would not get accepted. However, it ended up in Linus Torvalds's tree. Let's explore what happened.

The issue Babka wanted to fix is the fact that `kmalloc()` sometimes returns objects that are not naturally aligned (that is, aligned to the object size if that size is a power of two). Most of the time, though, `kmalloc()` does return naturally aligned objects and some drivers and subsystems have come to depend on that property. The exceptions are when SLUB debugging is enabled or when the SLOB allocator is used. `kmalloc()` is essentially a shell around the SLAB, SLUB or SLOB allocator, depending on the kernel configuration; interested readers may wish to read an article on [the reasons SLUB was introduced](<https://lwn.net/Articles/229984/>) and look at a LinuxCon 2014 [slide set [PDF]](<https://events.static.linuxfound.org/sites/events/files/slides/slaballocators.pdf>) on the three allocators. Unexpectedly returning an unaligned object can cause data corruption and other errors. In response to that problem, Babka proposed to guarantee natural alignment for allocated objects with power-of-two size, so that all alignment expectations are fulfilled.

#### For and against `kmalloc()` alignment

In the patch set discussion, Christopher Lameter (the creator of the SLUB allocator) [disagreed](<https://lwn.net/ml/linux-kernel/0100016cd98bb2c1-a2af7539-706f-47ba-a68e-5f6a91f2f495-000000@email.amazonses.com/>) with the idea of adding natural alignment and noted that `kmalloc()` has its own alignment limit (`KMALLOC_MINALIGN`) for a reason: to allow optimized memory layout without wasting memory. The SLOB allocator is an example; it is designed for small embedded systems and to incur minimal overhead. The patch from Babka would change that expected behavior. Also, any future allocators would have to take those new constraints into account and that would prevent them from implementing certain optimizations in their memory layout.

Matthew Wilcox [was in favor of Babka's proposal](<https://lwn.net/ml/linux-kernel/20190828194607.GB6590@bombadil.infradead.org/>), as there are many subsystems that already depend on the implied alignment behavior. He mentioned examples like the persistent-memory (pmem) and RAM-disk drivers. The XFS filesystem, without an alignment guarantee, would need slab caches for each object size between 512 bytes and `PAGE_SIZE`, and it may need even more of them depending on what `kmalloc()` does guarantee.

Dave Chinner [agreed](<https://lwn.net/ml/linux-kernel/20190828222422.GL1119@dread.disaster.area/>) with providing alignment for small objects and spoke for further alignment of large objects (bigger than a page) to page boundaries. This need was seen when using pmem with [KASAN](<https://lwn.net/Articles/612153/>). He suggested, though, using a `GFP` flag to tell the allocator to return a naturally aligned object, and to fail if it cannot. That would avoid the need for higher-level subsystems to create additional caches. Babka and other developers preferred to deal with the issue without a separate flag.

A heated debate followed about the severity of the issue. Lameter [disagreed](<https://lwn.net/ml/linux-kernel/0100016cf8c3033d-bbcc9ba3-2d59-4654-a7c2-8ba094f8a7de-000000@email.amazonses.com/>) that the misalignment cases are frequent, or even seen in practice, as the drivers affected are enabled in distribution test systems that use debug options. The cases of bad alignment should have been seen in that testing, according to him. Christoph Hellwig [noted](<https://lwn.net/ml/linux-kernel/20190904051933.GA10218@lst.de/>) that the breakage often happens under special conditions, like buffers that cross a page boundary.

#### From a private NAK to the mainline

Following the debate, Babka [asked](<https://lwn.net/ml/linux-kernel/df8d1cf4-ff8f-1ee1-12fb-cfec39131b32@suse.cz/>) for formal approval or disapproval of the patch set: 

So if anyone thinks this is a good idea, please express it (preferably in a formal way such as Acked-by), otherwise it seems the patch will be dropped (due to a private NACK, apparently). 

David Sterba [commented](<https://lwn.net/ml/linux-kernel/20190923171710.GN2751@twin.jikos.cz/>) that he has had to apply workarounds for misalignment cases and would be happy to remove them when the generic code is fixed. Darrick J. Wong [seconded](<https://lwn.net/ml/linux-kernel/20190923175146.GT2229799@magnolia/>) Sterba's opinion and expressed his strong preference for open discussion: 

Oh, I didn't realize ^^^^^^^^^^^^ that *some* of us are allowed the privilege of gutting a patch via private NAK without any of that open development discussion inconvenience. <grumble>

Lameter [followed up](<https://lwn.net/ml/linux-kernel/alpine.DEB.2.21.1909242045250.17661@www.lameter.com/>) stating that the options to detect misalignment have been available for years and are ready to use. Wilcox [disagreed](<https://lwn.net/ml/linux-kernel/20190924205133.GK1855@bombadil.infradead.org/>), as the issues show up when debugging options are enabled and this is particularly the case when all of the other features should work fine: 

People who are enabling a debugging option to debug their issues, should not have to first debug all the other issues that enabling that debugging option uncovers! 

Andrew Morton moved the discussion back to the technical subject and [asked for verification](<https://lwn.net/ml/linux-kernel/20190924165425.a79a2dafbaf37828a931df2b@linux-foundation.org/>) of the patch's correctness. Lameter [confirmed](<https://lwn.net/ml/linux-kernel/alpine.DEB.2.21.1909260005060.1508@www.lameter.com/>) that it is technically fine, while still disagreeing with the intent. That was followed by a number of acknowledgments (`Acked-by:`) from kernel developers showing their support for Babka's solution.

That series of approvals ended the public discussion; Babka did not resend the patch set or submit a third version. The situation seemed blocked as the patch set had support of multiple developers, but not from the maintainer of the SLUB allocator, which is heavily affected by the patch set. However, [the patch was included](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=eda57a0e42998d1d403187844faa86c9a3ab2fd0>) in Morton's tree and was [merged to the mainline](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=59bb47985c1db229ccff8c5deebecd54fc77d2a9>) on October 7th.

#### Summary

This discussion shows an example of the kernel community working on a change that affects a behavior that has been present for a long time. It is not a surprise that not all developers agreed with the solution -- however, in this case, the one disagreeing was the maintainer of one of the modified subsystems. The final result shows that such changes can be accepted into the mainline since there was wide support from `kmalloc()` users and other memory-management developers.
