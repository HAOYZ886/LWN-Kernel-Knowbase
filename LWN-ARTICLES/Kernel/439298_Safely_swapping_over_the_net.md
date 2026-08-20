---
title: Safely swapping over the net
url: https://lwn.net/Articles/439298/
date: "April 19, 2011"
category: "Memory management-Swapping"
author: "By Jonathan Corbet April 19, 2011"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
April 19, 2011

Swapping, like page writeback, operates under some severe constraints. The ability to write dirty pages to backing store is critical for memory management; it is the only way those pages can be freed for other uses. So swapping must work well in situations where the system has almost no memory to spare. But writing pages to backing store can, itself, require memory. This problem has been well solved (with mempools) for locally-attached devices, but network-attached devices add some extra challenges which have never been addressed in an entirely satisfactory way. 

This is not a new problem, of course; LWN [ran an article about swapping over network block devices](<https://lwn.net/Articles/129703/>) (NBD) almost exactly six years ago. Various approaches were suggested then, but none were merged; it remains to be seen whether [the latest attempt](<https://lwn.net/Articles/438407/>) (posted by Mel Gorman based on a lot of work by Peter Zijlstra) will be more successful. 

The kernel's page allocator makes a point of only giving out its last pages to processes which are thought to be working to make more memory free. In particular, a process must have either the `PF_MEMALLOC` or `TIF_MEMDIE` flag set; `PF_MEMALLOC` indicates that the process is currently performing [memory compaction](<https://lwn.net/Articles/368869/>) or direct reclaim, while `TIF_MEMDIE` means the process has run afoul of the out-of-memory killer and is trying to exit. This rule should serve to keep some memory around for times when it is needed to make more memory free, but one aspect of this mechanism does not work entirely well: its interaction with slab allocators. 

The slab allocators grab whole pages and hand them out in smaller chunks. If a process marked with `PF_MEMALLOC` or `TIF_MEMDIE` requests an object from the slab allocator, that allocator can use a reserved page to satisfy the request. The problem is that the remainder of the page is then made available to any other process which may make a request; it could, thus, be depleted by processes which are making the memory situation worse, not better. 

So one of the first things Mel's patch series does is to adapt a patch by Peter that adds more awareness to the slab allocators. A new boolean value (`pfmemalloc`) is added to `struct page` to indicate that the corresponding page was allocated from the reserves; the recipient of the page is then expected to treat it with due care. Both slab and SLUB have been modified to recognize this flag and reserve the rest of the page for suitably-marked processes. That change should help to ensure that memory is available where it's needed, but at the cost of possibly failing other memory allocations even though there are objects available. 

The next step is to add a `__GFP_MEMALLOC` GFP flag to mark allocation requests which can dip into the reserves. This flag separates the marking of urgent allocation requests from the process state - a change will be useful later in the series, where there may be no convenient process state available. It will be interesting to see how long it takes for some developer to attempt to abuse this flag elsewhere in the kernel. 

The big problem with network-based swap is that extra memory is required for the network protocol processing. So, if network-based swap is to work reliably, the networking layer must be able to access the memory reserves. Quite a bit of network processing is done in software interrupt handlers which run independently of any given process. The `__GFP_MEMALLOC` flag allows those handlers to access reserved memory, once a few other tweaks have been added as well. 

It is not desirable to allow _any_ network operation to access the reserves, though; bittorrent and web browsers should not be allowed to consume that memory when it is urgently needed elsewhere. A new function, `sk_set_memalloc()`, is added to mark sockets which are involved with memory reclaim. Allocations for those sockets will use the `__GFP_MEMALLOC` flag, while all other sockets have to get by with ordinary allocation priority. It is assumed that only sockets managed within the kernel will be so marked; any socket which ends up in user space should not be able to access the reserves. So swapping onto a FUSE filesystem is still not something which can be expected to work. 

There is one other problem, though: incoming packets do not have a special "needed for memory reclaim" flag on them. So the networking layer must be able to allocate memory to hold _all_ incoming packets for at least as long as it takes to identify the important ones. To that end, any network allocation for incoming data is allowed to dip into the reserves if need be. Once a packet has been identified and associated with a socket, that socket's flags can be checked; if the packet was allocated from the reserves and the destination socket is _not_ marked as being used for memory reclaim, the packet will be dropped immediately. That change should allow important packets to get into the system without consuming too much memory for unimportant traffic. 

The result should be a system where it is safe to swap over a network block device. At least, it should be safe if the low watermark - which controls how much memory is reserved - is high enough. Systems which are swapping over the net may be expected to make relatively heavy use of the reserves, so administrators may want to raise the watermark (found in `/proc/sys/vm/min_free_kbytes`) accordingly. The final patch in the series keeps an eye on the reserves and start throttling processes performing direct reclaim if they get too low; the idea here is to ensure that enough memory remains for a smaller number of reclaimers to actually get something done. Adjusting the size of the reserves dynamically might be the better solution in the long run, but that feature has been omitted for now in the interest of keeping the patch series from getting too large.
