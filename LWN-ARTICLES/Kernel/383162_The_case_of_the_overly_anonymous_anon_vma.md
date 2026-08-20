---
title: The case of the overly anonymous anon_vma
url: https://lwn.net/Articles/383162/
date: "April 13, 2010"
category: "Memory management-Object-based reverse mapping; anon vma"
author: "By Jonathan Corbet April 13, 2010"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
April 13, 2010

During the stabilization phase of the kernel development cycle, the -rc releases typically happen about once every week. [2.6.34-rc4](<http://lwn.net/Articles/383198/>) is a clear exception to that rule, coming nearly two weeks after the preceding -rc3 release. The holdup in this case was a nasty regression which occupied a number of kernel developers nearly full time for days. The hunt for this bug is a classic story of what can happen when the code gets too complex. 

Sending email to linux-kernel can be an intimidating prospect for a number of reasons, one of which being that one never knows when a massive thread - involving hundreds of messages copied back to the original sender - might result. Borislav Petkov's [2.6.34-rc3 bug report](<https://lwn.net/Articles/383163/>) was one such posting. In this case, though, the ensuing thread was in no way inflammatory; it represents, instead, some of the most intensive head-scratching which has been seen on the list for a while. 

The bug, as reported by Borislav, was a null pointer dereference which would happen reasonably reliably after hibernating (and restarting) the system. It was quickly recognized as being the same as [another bug report](<https://bugzilla.kernel.org/show_bug.cgi?id=15680>) filed the same day by Steinar H. Gunderson, though this one did not involve hibernation. The common thread was null pointer dereferences provoked by memory pressure. The offending patch was [identified by Linus](<https://lwn.net/Articles/383165/>) almost immediately; it's worth taking a look at what that patch did. 

Way back in 2004, LWN [covered the addition of the anon_vma code](<http://lwn.net/Articles/75198/>); this patch was controversial at the time because the upcoming 2.6.7 kernel was still expected to be an old-style "stable, no new features" release. This patch, a 40-part series which fundamentally reworked the virtual memory subsystem, was not seen as stable material, despite Linus's [attempt](<http://lwn.net/Articles/86718/>) to characterize it as an "implementation detail." Still, over time, this code has proved solid and has not been changed significantly since - until now. 

The problem solved by anon_vma was that of locating all `vm_area_struct` (VMA) structures which reference a given anonymous (heap or stack memory) page. Anonymous pages are not normally shared between processes, but every call to `fork()` will cause all such pages to be shared between the parent and the new child; that sharing will only be broken when one of the processes writes to the page, causing a copy-on-write (COW) operation to take place. Many pages are never written, so the kernel must be able to locate multiple VMAs which reference a given anonymous page. Otherwise, it would not be able to unmap the page, meaning that the page could not be swapped out. 

The reverse mapping solution originally used in 2.6 proved to be far too expensive, necessitating a rewrite. This rewrite introduced the `anon_vma` structure, which heads up a linked list of all VMAs which might reference a given page. So a `fork()` also causes every VMA in the child process which contains anonymous pages to be added to a the list maintained in the parent's `anon_vma` structure. The `mapping` pointer in `struct page` points to the `anon_vma` structure, allowing the kernel to traverse the list and find all of the relevant VMA structures. 

This diagram, from the 2004 article, shows how this data structure looks: 

> ![\[anonvma\]](https://static.lwn.net/images/ns/anonvma2.png)

This solution scaled far better than its predecessor, but eventually the world caught up. So Rik van Riel set out to make things faster, writing [this patch](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commitdiff;h=5beb49305251e5669852ed541e8e2f2f7696c53e>), which was merged for 2.6.34. Rik describes the problem this way: 

In a workload with 1000 child processes and a VMA with 1000 anonymous pages per process that get COWed, this leads to a system with a million anonymous pages in the same anon_vma, each of which is mapped in just one of the 1000 processes. However, the current rmap code needs to walk them all, leading to O(N) scanning complexity for each page. 

Essentially, by organizing all anonymous pages which originated in the same parent under the same `anon_vma` structure, the kernel created a monster data structure which it had to traverse every time it needed to reverse-map a page. That led to the kernel scanning large numbers of VMAs which could not possibly reference the page, all while holding locks. The result, says Rik, was "catastrophic failure" when running the AIM benchmark. 

Rik's solution was to create an `anon_vma` structure for each process and to link those together instead of the VMA structures. This linking is done with a new structure called `anon_vma_chain`: 

```
struct anon_vma_chain {
    	struct vm_area_struct *vma;
    	struct anon_vma *anon_vma;
    	struct list_head same_vma;
    	struct list_head same_anon_vma;
        };
```

Each `anon_vma_chain` entry (AVC) maintains two lists: all `anon_vma` structures relevant to a given vma (`same_vma`), and all VMAs which fall within the area covered by a given `anon_vma` structure (`same_anon_vma`). It gets complicated, so some diagrams might help. Initially, we have a single process with one anonymous VMA: 

> ![\[AV
Chain\]](https://static.lwn.net/images/ns/kernel/avchain1.png)

Here, "AV" is the `anon_vma` structure, and "AVC" is the `anon_vma_chain` structure seen above. The AVC links to both the `anon_vma` and VMA structures through direct pointers. The (blue) linked list pointer is the `same_anon_vma` list, while the (red) pointer is the `same_vma` list. So far, so simple. 

Imagine now that this process forks, causing the VMA to be copied in the child; initially we have a lonely new VMA like this: 

> ![\[AV
Chain\]](https://static.lwn.net/images/ns/kernel/avchain2.png)

The kernel needs to link this VMA to the parent's `anon_vma` structure; that requires the addition of a new `anon_vma_chain`: 

> ![\[AV
Chain\]](https://static.lwn.net/images/ns/kernel/avchain3.png)

Note that the new AVC has been added to the blue list of all VMAs referencing a given `anon_vma` structure. The new VMA also needs its own `anon_vma`, though: 

> ![\[AV
Chain\]](https://static.lwn.net/images/ns/kernel/avchain4.png)

Now there's yet another `anon_vma_chain` structure linking in the new `anon_vma`. The new red list has been expanded to contain all of the AVCs which reference relevant `anon_vma` structures. As your editor said, it gets complicated; the diagram for the 1000-child scenario which motivated this patch will be left as an exercise for the reader. 

When the `fork()` happens, all of the anonymous pages in the area point back to the parent's `anon_vma` structure. Whenever the child writes to a page and causes a copy-on-write, though, the new page will map back to the child's `anon_vma` structure instead. Now, reverse-mapping that page can be done immediately, with no need to scan through any other processes in the hierarchy. That makes the lock contention go away, making benchmarkers happy. 

The only problem is that embarrassing oops issue. Linus, Rik, Borislav, and others chased after it, trying no end of changes. For a while, it seemed that a bug causing excessive reuse of `anon_vma` structures when VMAs were merged could be the problem, but fixing the bug did not fix this oops. Sometimes, changing VMA boundaries with `mprotect()` could cause the wrong `anon_vma` to be used, but fixing that one didn't help either. The reordering of chains when they were copied was also noted as a problem...but it wasn't _the_ problem. 

Linus was clearly beginning to [wonder](<https://lwn.net/Articles/383170/>) when it might all end: ""Three independent bugs found and fixed, and still no joy?"" He repeatedly considered just reverting the change outright, but he was reluctant to do so; the solution seemed so tantalizingly close. Eventually he [developed another hypothesis](<https://lwn.net/Articles/383171/>) which seemed plausible. An anonymous page shared between parent and child would initially point to the parent's `anon_vma`: 

> ![\[AV
Chain\]](https://static.lwn.net/images/ns/kernel/avchain5.png)

But, if both processes were to unmap the page (as could happen during system hibernation, for example), then the child referenced it first, it could end up pointing to the child's `anon_vma` instead: 

> ![\[AV
Chain\]](https://static.lwn.net/images/ns/kernel/avchain6.png)

If the parent mapped the page later, then the child unmapped it (by exiting, perhaps), the parent would be left with an anonymous page pointing to the child's `anon_vma` \- which no longer exists: 

> ![\[AV
Chain\]](https://static.lwn.net/images/ns/kernel/avchain7.png)

Needless to say, that is a situation which is unlikely to lead to anything good in the near future. 

The fix is straightforward; when linking an existing page to an `anon_vma` structure, the kernel needs to pick the one which is highest in the process hierarchy; that guarantees that the `anon_vma` will not go away prematurely. [Early testing](<https://lwn.net/Articles/383172/>) suggests that the problem has indeed been fixed. In the process, three other problems have been fixed and Linus has come to understand a tricky bit of code which, if he has his way, will soon gain some improved documentation. In other words, it would appear to be an outcome worth waiting for.
