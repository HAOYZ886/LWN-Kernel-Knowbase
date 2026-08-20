---
title: Some ado about zero
url: https://lwn.net/Articles/340370/
date: "July 7, 2009"
category: Memory management; Zero page
author: "By Jonathan Corbet July 7, 2009"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
July 7, 2009

Computers use a lot of zeroes. Early in your editor's programming career, he worked on a machine that provided a special hardware register containing zero; programmers on this system knew they could use all the zeroes they needed with no fear of running out. Meanwhile, in this century, the Linux kernel sets aside a page full of zeros. It's called `empty_zero_page` on the x86 architecture, and it's even exported to modules. Interestingly, this special page is not used as heavily as it was prior to the 2.6.24 kernel, but that may be about to change. 

In the good old days, the kernel would use the zero page in situations where it knew it needed a page full of zeroes. So, for example, if a process incurred a read fault on a page it had never used, the kernel would simply map the zero page into that address. A copy-on-write mapping would be used, of course; if the process subsequently modified the page, it would end up with its own copy. But deferring the creation of a new, zero-filled page helped to conserve zeroes, keeping the kernel from running out. Incidentally, it also saved memory, reduced cache pressure, and eliminated the need to clear the new page. 

Memory management changes made back in 2007 had the effect of adding reference counting to the zero page. And that turned out to be a problem on multiprocessor machines. Since all processors shared the same zero page (per-CPU differences being unlikely), they also all manipulated the same reference count. That led to serious problems with cache line bouncing, with a measurable performance impact. In response, Nick Piggin evaluated a number of possible fixes, including special hacks to avoid reference-counting the zero page or adding per-CPU zero pages. The [patch that got merged](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commit;h=557ed1fa2620dc119adb86b34c614e152a629a80>), though, simply eliminated most use of the zero page altogether. The change was justified this way: 

Inserting a ZERO_PAGE for anonymous read faults appears to be a false optimisation: if an application is performance critical, it would not be doing many read faults of new memory, or at least it could be expected to write to that memory soon afterwards. If cache or memory use is critical, it should not be working with a significant number of ZERO_PAGEs anyway (a more compact representation of zeroes should be used). 

There was some nervousness about the patch at the time; Linus grumbled about the changes which created the problem in the first place, and [worried](<http://lkml.org/lkml/2007/10/10/16>): 

The kernel has *always* (since pretty much day 1) done that ZERO_PAGE thing. This means that I would not be at all surprised if some application basically depends on it. I've written test-programs that depends on it - maybe people have written other code that basically has been written for and tested with a kernel that has basically always made read-only zero pages extra cheap. 

Despite his misgivings, Linus merged the patch for 2.6.24 to see what sort of problems might come to the surface. For the next 18 months, it appeared that such problems were scarce indeed; most people forgot about the zero page altogether. In early June, though, Julian Phillips [reported](<https://lwn.net/Articles/340377/>) a problem he had observed: 

I have a program which creates a reasonably large private anonymous map. The program then writes into a few places in the map, but ends up reading from all of them. 

When I run this program on a system running 2.6.20.7 the process only ever seems to use enough memory to hold the data that has actually been written (well - in units of PAGE_SIZE). When I run the program on a system running 2.6.24.5 then as it reads the map the amount of memory used continues to increase until the complete map has actually been allocated (and since the total size is greater than the physically available RAM causes swapping). Basically I seem to be seeing copy-on-read instead of copy-on-write type behaviour. 

What Julian was seeing, of course, was the effects from the removal of the zero page. On older kernels, all of the unwritten pages in the data structure would be mapped to the zero page, using no additional physical memory at all. As of 2.6.24, each of those pages gets an actual physical page - containing nothing but zeroes - assigned to it, increasing memory use significantly. 

Hiroyuki Kamezawa reports that he has seen zero-page-dependent workloads at other sites. Many of those sites, he says, are running enterprise Linux distributions which have not, yet, shipped kernels new enough to lack zero page support. He worries that these users will encounter the same sort of unpleasant surprise Julian found when they upgrade to newer kernels. In response, he has [posted a patch](<http://lwn.net/Articles/340323/>) which restores zero page support to the kernel. 

Hiroyuki's zero page support isn't quite the same as what came before, though. It avoids reference counting for the zero page, a change which should eliminate the worst of the performance problems. It does add some interesting special cases, though, where virtual memory code has to be careful to test for zero pages; the bulk of those cases are handled with the addition of a `get_user_pages_nonzero()` function which removes any zero pages from the indicated range. Linus [dislikes](<https://lwn.net/Articles/340382/>) the special cases, thinking that they are unnecessary. Instead, he has proposed an alternative implementation using the relatively new `PTE_SPECIAL` flag to mark zero pages. As of this writing, a updated version of the patch using this approach has not yet been posted. 

Nick Piggin, who wrote the patch removing zero page support in the first place, [would rather not see it return](<https://lwn.net/Articles/340383/>). With regard to the affected users, he asks: 

Can we just try to wean them off it? Using zero page for huge sparse matricies is probably not ideal anyway because it needs to still be faulted in and it occupies TLB space. They might see better performance by using a better algorithm. 

Linus, however, [would like to see this feature restored](<https://lwn.net/Articles/340384/>) if it can be done in a clean way. So the return of zero page support seems fairly likely, assuming the patch can be worked into sufficiently good shape. Whether that will bring comfort to enterprise kernel users remains to be seen, though; the next generation of enterprise Linux releases look set to use kernels around 2.6.27. Unless distributors backport the zero page patch, enterprise Linux users will still be stuck with the current, zero-wasting behavior.
