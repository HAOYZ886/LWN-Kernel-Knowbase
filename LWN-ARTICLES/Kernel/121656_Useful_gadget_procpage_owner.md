---
title: "Useful gadget: /proc/page_owner"
url: https://lwn.net/Articles/121656/
date: "February 1, 2005"
category: Debugging; Memory management; procpage owner
author: 
---

If you look far enough into the [2.6.11-rc2-mm2 announcement](<https://lwn.net/Articles/121271/>), you'll find a mention of a "page owner tracking leak detector" patch. The addition of this patch was almost certainly motivated by the series of memory leak problems which have afflicted the 2.6.11 prepatches. It is a heavy-handed tool, but, for some situations, it might make the problem of finding memory leaks far easier. 

Essentially, this patch causes the kernel to keep track of the call chain that leads to the allocation of every page. This information is made available via `/proc/page_owner`; it looks something like this: 

> 

```
>     Page allocated via order 0
>     [0xc0146f01] kmem_getpages+49
>     [0xc014846d] cache_grow+173
>     [0xc0148aac] cache_alloc_refill+460
>     [0xc0118a8f] copy_files+431
>     [0xc0148ff5] kmem_cache_alloc+149
>     [0xc011986b] copy_process+3051
>     [0xc01199d1] fork_idle+65
>     [0xc041824a] do_boot_cpu+42
>
```

Your editor's 256MB sacrificial kernel box has, after a short period of run time, over 13,000 such entries. So plowing through the raw data is probably not what most people want to do. To help out, a small program ([page_owner.c](<https://lwn.net/Articles/121659/>)) has been put into the `Documentation` directory (though one might argue that it should be in `scripts` instead). This program boils down the contents of `/proc/page_owner` to something which looks like this: 

> 

```
>     856 times:
>     Page allocated via order 0
>     [0xc0146572] __do_page_cache_readahead+290
>     [0xc0146a70] max_sane_readahead+48
>     [0xc0140166] filemap_nopage+790
>     [0xc013fe50] filemap_nopage+0
>     [0xc0150861] do_no_page+193
>     [0xc0150cc6] handle_mm_fault+246
>     [0xc01126cc] do_page_fault+492
>     [0xc0151b3c] remove_vm_struct+140
>     
>     839 times:
>     Page allocated via order 0
>     [0xc0146572] __do_page_cache_readahead+290
>     [0xc0146a70] max_sane_readahead+48
>     [0xc0140166] filemap_nopage+790
>     [0xc013fe50] filemap_nopage+0
>     [0xc0150861] do_no_page+193
>     [0xc0150cc6] handle_mm_fault+246
>     [0xc01126cc] do_page_fault+492
>     [0xc013c207] ltt_log_event+71
>
```

With this output, finding the source of a major memory leak should be relatively straightforward. It's worth noting that this program fails if told to read directly from `/proc/page_owner` (it does a `stat()` to determine the size of its input), so you must copy the data to a regular file first. This patch is also a major memory consumer in its own right, since it must store the call chain information for every allocated page. It's thus not something most people would put onto a production system - or even on most development systems. But it can be a useful thing to have around when a page-level memory leak bites.
