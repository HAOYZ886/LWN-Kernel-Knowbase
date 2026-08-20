---
title: Toward merging reiser4
url: https://lwn.net/Articles/151204/
date: "September 11, 2005"
category: "Filesystems-Reiser4"
author: 
---

The reiser4 filesystem has been the subject of a long, ongoing conversation for many months; look under "reiser4" in [the LWN Kernel Page Index](<http://lwn.net/Kernel/Index/>) for previous coverage on this page. The reiser4 developers have been working hard to get their new filesystem merged into the mainline kernel, and they believe that the time has come. To that end, Hans Reiser has posted [a list of concerns](<https://lwn.net/Articles/151206/>) raised by others. His hope is to get definitive answers on what has to be done to get reiser4 in, hopefully for 2.6.14. 

One of the big issues since the beginning has been the reiser4 metafiles feature, where every file can, itself, be treated as a directory with the file's attributes accessible as files in their own right. This feature raised many eyebrows just by looking weird and non-Unix-like, but the real issue was one of locking. The Linux virtual filesystem code is simply not set up to handle files as directories, so it is easy for a user to deadlock the system. Even Hans Reiser, a strong defender of the metafile feature, sees these deadlocks as an undesirable thing. 

So, while reiser4 has been in -mm for quite some time, the metafile feature has been disabled. There is no talk of turning it back on for a mainline merge; the real issue, instead, is whether the code should be allowed to remain at all. The consensus on the kernel side would appear to be that unused code does not belong in the kernel, so the metafile implementation is likely to be removed altogether. Someday, if the locking issues are resolved, it might yet return. 

Reiser4 has long had trouble working with 4K kernel stacks (see [last week's Kernel Page](<http://lwn.net/Articles/149977/>)). It would appear that this issue has now been resolved. Another complaint which has been raised has to do with a large number of debugging tests in the code itself; some developers see it as clutter and would like it to be removed. Here, however, Andrew Morton has sided with the reiser4 hackers and told them to leave the tests in. 

Reiser4 implements a couple of its own types for condition variables and linked lists. In both cases, it is thought that the in-kernel primitives could be used, rather than introducing new, redundant types. Those will probably have to be fixed before this code can be merged. 

The end result is that quite a bit of work remains to be done, meaning that it is unlikely to be ready before 2.6.14 closes to new features. Andrew has [hinted](<https://lwn.net/Articles/151208/>) that reiser4 might just slip in after the deadline, though: 

But something like a brand new filesystem can go in pretty much any time, as long as it compiles. Because it can't break anyone's current setup. 

The one issue which, interestingly, has not come up in the recent discussion has been the plugin architecture used by reiser4. To a number of developers, that sort of feature does not belong at the individual filesystem level; it should, instead, be made part of the VFS layer and made available to all filesystems. It would appear that a more moderate viewpoint, allowing the feature to be merged now with the idea of shifting it up into the VFS over time, has won out.
