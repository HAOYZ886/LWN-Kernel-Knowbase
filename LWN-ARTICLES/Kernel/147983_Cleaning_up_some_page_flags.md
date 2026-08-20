---
title: Cleaning up some page flags
url: https://lwn.net/Articles/147983/
date: "August 17, 2005"
category: "Memory management-Internal API"
author: 
---

`struct page` is at the core of the memory management subsystem; one of these structures exists for every physical page of memory on the system (and for a few places which are not memory). Since a typical system will contain large numbers of `page` structures, there is a great deal of pressure to keep that structure small. But there are a lot of things that the kernel needs to know about pages. The result is that `struct page` contains a densely-packed `flags` field, and that the developers continually worry about running out of space for flags \- even though a fair number of them are currently unused. Some of these flags also carry a fair amount of historical baggage which would be nice to clean up. 

Consider, for example, a flag called `PG_checked`. Its definition in `include/linux/page-flags.h` (2.6.13-rc6) reads as follows: 

```
#define PG_checked		 8	/* kill me in 2.5.<early>. */
```

Somebody clearly missed a deadline. In fact, there is a certain amount of confusion over just what this flag does. A bit of research revealed that it is used in several filesystems, and that it is unlikely to go away anytime soon. ext3 uses this flag to mark pages to be written to disk at a future time. AFS uses it to indicate valid directory pages. Reiserfs uses this flag for journaling purposes. And the (out-of-tree) cachefs implementation uses it to mark pages currently being written to local backing store. 

So this flag clearly is not going away anytime soon, much less by 2.5.early. In an effort to clarify the situation, Daniel Phillips has posted [a patch](<https://lwn.net/Articles/147988/>) which renames the flag as follows: 

```
#define PG_fs_misc		 8	/* don't let me spread */
```

There is some disagreement over naming, but the core of the patch is uncontroversial. This flag will officially be dedicated to filesystem use. 

Another flag with significant history is `PG_reserved`. In this case, too, the meaning of the flag has been somewhat obscured over time, though it can be summarized as "this page is special and the VM subsystem should leave it alone." It marks parts of the physical address space which have `page` structures, but which are not real memory - the legacy ISA hole in the i386 space, for example. The memory dedicated to the kernel text is also marked reserved. The kernel function which maps physical address spaces into a process's virtual space (`remap_pfn_range()`) will refuse to remap unreserved memory, leading to a long history of device drivers setting that flag to remap internal buffers. 

The consensus seems to be that the "reserved" flag can go. So Nick Piggin has been working on [a patch](<http://lwn.net/Articles/146579/>) which takes it out - mostly. In many cases, code which was testing that flag was really trying to decide if it was looking at a valid RAM page; there are other, better ways of making that test. In other cases, the higher-level VMA structure (which has its own `VM_RESERVED` flag) contains all of the needed information. In the `remap_pfn_range()` case, the test is simply removed, allowing all memory to be remapped. This change will modify the behavior of `/dev/mem`, which, previously, could not be used to `mmap()` regular RAM. 

All that is left, after Nick's patch, is a set of tests in the software suspend code. Once that has been taken care of, `PG_reserved` can go.
