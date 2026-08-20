---
title: "zcache: a compressed page cache"
url: https://lwn.net/Articles/397574/
date: "July 27, 2010"
category: "Memory management-Swapping; Transcendent memory"
author: "By Jonathan Corbet July 27, 2010"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
July 27, 2010

Last year, Nitin Gupta was pushing the [compcache](<http://lwn.net/Articles/334649/>) patch, which implemented a sort of swap device which stored pages in main memory, compressing them on the way. Over time, compcache became "ramzswap" and found its way into the staging tree. It's not clear that ramzswap can ever graduate to the mainline kernel, so Nitin is trying again with a development called [zcache](<http://lwn.net/Articles/396467/>). But zcache, too, currently lacks a clear path into the mainline. 

Like its predecessors, zcache lives to store compressed copies of pages in memory. It no longer looks like a swap device, though; instead, it is set up as a backing store provider for the [Cleancache](<http://lwn.net/Articles/386090/>) framework. Cleancache uses a set of hooks into the page cache and filesystem code; when a page is evicted from the cache, it is passed to Cleancache, which might (or might not) save a copy somewhere. When pages are needed again, Cleancache gets a chance to restore them before the kernel reads them from disk. If Cleancache (and its backing store) is able to quickly save and restore pages, the potential exists for a real improvement in system performance. 

Zcache uses LZO to compress pages passed to it by Cleancache; only pages which compress to less than half their original size are stored. There is also a special test for pages containing only zeros; those compress exceptionally well, requiring no storage space at all. There is not, at this point, any other attempt at the unification of pages with duplicated contents (as is done by KSM), though. 

There are a couple of obvious tradeoffs to using a mechanism like zcache: memory usage and CPU time. With regard to memory, Nitin says: 

While compression reduces disk I/O, it also reduces the space available for normal (uncompressed) page cache. This can result in more frequent page cache reclaim and thus higher CPU overhead. Thus, it's important to maintain good hit rate for compressed cache or increased CPU overhead can nullify any other benefits. This requires adaptive (compressed) cache resizing and page replacement policies that can maintain optimal cache size and quickly reclaim unused compressed chunks. This work is yet to be done. 

The current patch does allow the system administrator to manually adjust the size of the zcache area, which is a start. It will be a rare admin, though, who wants to watch cache hit rates and tweak low-level memory management parameters in an attempt to sustain optimal behavior over time. So zcache will almost certainly have to grow some sort of adaptive self-tweaking before it can make it into the mainline. 

The other tradeoff is CPU time: it takes processor time to compress and decompress pages of memory. The cost is made worse by any pages which fail to compress down to less than 50% of their original size - the time spent compressing them is a total waste. But, as Nitin points out: ""with multi-cores becoming common, benefits of reduced disk I/O should easily outweigh the problem of increased CPU usage"". People have often wondered what we are going to do with the increasing number of cores on contemporary processors; perhaps zcache is part of the answer. 

One other issue remains to be resolved, though: zcache depends on Cleancache, which is not currently in the mainline. There is some [opposition](<https://lwn.net/Articles/397582/>) to merging Cleancache, mostly because that patch, which makes changes to individual filesystems, is seen as being overly intrusive. It's also not clear that everybody is, yet, sold on the value of Cleancache, despite the fact that SUSE has been shipping it for a little while now. Until the fate of Cleancache is resolved, add-on patches like zcache will be stuck outside of the mainline.
