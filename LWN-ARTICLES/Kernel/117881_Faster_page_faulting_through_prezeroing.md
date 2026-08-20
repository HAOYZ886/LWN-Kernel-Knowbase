---
title: Faster page faulting through prezeroing
url: https://lwn.net/Articles/117881/
date: "January 5, 2005"
category: Memory management
author: 
---

In early December, this page [covered](<http://lwn.net/Articles/114596/>) Christoph Lameter's efforts to speed up the page fault mechanism by reducing lock contention. That work speeds things significantly on multiprocessor systems, but is of little help to uniprocessor users. That is not true of Christoph's other page fault work, which can benefit users on all systems. 

Christoph [notes](<http://lwn.net/Articles/117794/>) that, once the locking issues are taken care of, the most expensive part of the page fault handler is the code which zeroes anonymous pages before handing them to the faulting process. He has concluded that, in some situations, performance can be significantly improved by clearing those pages ahead of time and having them ready when the page fault happens. Just zeroing pages ahead of time is not particularly helpful - it is mostly an exercise in moving work around to different places in the system. But, if (1) the zeroing of pages can be made more efficient, and (2) the workload is of the right type, things can be made quite a bit faster. 

What is the right kind of workload? For the purposes of this patch set, the best workload is one which allocates whole pages, but then only touches parts of them. If those pages are already cleared, there is no need to load an entire page into the processor cache when it is faulted in. The improved cache behavior, along with the speedup in fault handling itself, can yield big improvements. Some figures posted by Christoph show an almost 4x improvement in the page fault rate in the right conditions. As it turns out, many applications fit this profile, so "the right conditions" should not be all that rare. 

There are four parts to the prezeroing patch set. [The first patch](<http://lwn.net/Articles/117795/>) extends the page allocation mechanism to make it explicitly handle requests for zeroed memory. There is a new `__GFP_ZERO` allocation flag which tells `alloc_pages()` (and thus functions like `__get_free_page()` and `kmalloc()`) to return zeroed memory. Many places in the kernel which clear their own pages have been fixed to request zeroed memory instead. With only this patch applied, the kernel's code is cleaned up a bit, but no performance improvements result - the `__GFP_ZERO` flag just causes a call to `clear_page()` in the page allocator. 

[The second patch](<http://lwn.net/Articles/117796/>) changes the prototype of the `clear_page()` function to: 

```
void clear_page(void *page, int order);
```

With the new interface, `clear_page()` can zero higher-order pages. This change is an important part of the patch set: pages are most efficiently zeroed if they can be done in larger groups. Often, the setup cost is a big part of the total; the value of prezeroing pages is much reduced if it can only be done one page at a time. 

[The kscrubd patch](<http://lwn.net/Articles/117797/>) is where things start to get interesting. This patch expands the `zone` structure so that it can keep track of pages which are known to be clear. Requests for zeroed pages are satisfied from this list when possible. A new kernel thread (actually, a set of per-node threads) wakes up occasionally and clears pages for future allocation. This thread does not normally scrub zero-order (single) pages, but can be configured to do so (via `/proc`) if desired. 

The kscrubd patch also implements a linked list of "zero drivers," being functions which can be called upon to zero pages efficiently. There are no such drivers in this patch, so all pages are zeroed with a call to `clear_page()`, which, as a comment in the code notes, can be hard on the processor's cache. It would be nicer if pages could be zeroed without the cache impacts. [The fourth patch](<http://lwn.net/Articles/117798/>) shows how this can be done - at least, on Altix systems. It adds a driver for the Altix block transfer engine which can zero memory directly without the processor's involvement - at least, when relatively large chunks of memory are involved. Drivers for other hardware have not yet been posted, but it would not be surprising to see them begin to appear after the prezeroing code has been merged. 

And that could happen soon: Linus, having been convinced by Christoph's results, has [requested](<https://lwn.net/Articles/117888/>) that this set of patches be merged soon. So prezeroing could even find its way into the kernel prior to the 2.6.11 release. (**Update** : the `__GFP_ZERO` patch was merged just as LWN was being published).
