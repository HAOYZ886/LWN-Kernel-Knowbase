---
title: "Not-so-anonymous virtual memory areas"
url: https://lwn.net/Articles/867818/
date: "September 3, 2021"
category: Android; Memory management
author: "By Jonathan Corbet September 3, 2021"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
September 3, 2021

Computing terminology can be counterintuitive at times, but even a longtime participant in the industry may have to look twice at the notion of named anonymous memory. That, however, is just the concept that [this patch set](<https://lwn.net/ml/linux-kernel/20210827191858.2037087-1-surenb@google.com/>) posted by Suren Baghdasaryan proposes to add. There are, it seems, developers who find the idea useful enough to not only overcome the initial cognitive dissonance that comes with it, but also to resurrect an eight-year-old patch to get it into the kernel. 

Memory used by user space is divided into two broad categories: file-backed and anonymous. A file-backed page of memory has a direct correspondence to a page in a file in persistent storage; when the page is clean, its contents are identical to what is found on disk. An anonymous page, instead, is not associated with a file in the filesystem; these pages hold a process's data areas, stacks, and so on. If an anonymous page must be written to persistent storage (to reclaim the page for another user, usually), space must be allocated in the swap area to hold its contents. 

Whether a given process's memory use is dominated by file-backed or anonymous pages varies from one workload to the next. In many cases, the bulk of a process's pages will be anonymous; this, it seems, is more likely in workloads with a lot of cloud-computing clients, which tend not to use many local files. Android devices are one place where this sort of behavior can be found. If one is trying to optimize the memory usage of such a workload, anonymous pages can pose a challenge; since the pages are anonymous, with no information about how they were created, it is difficult to know what any given anonymous page is being used for. 

That situation can be improved by making anonymous pages just a bit less anonymous. If it were possible to know which user-space subsystem or library created a given page, it would become easier to figure out who the biggest users are. Information on, say, how many anonymous pages in the system were created by the [jemalloc library](<https://github.com/jemalloc/jemalloc>), for example, could help determine whether jemalloc users are the best target for optimization efforts. Linux systems, however, do not make it easy (or even possible) to get that sort of information. 

Making things better requires obtaining some cooperation from user space, since the kernel cannot know which subsystem is allocating any given page. To that end, at the core of the patch set is [this patch from Colin Cross](<https://lwn.net/ml/linux-kernel/20210827191858.2037087-3-surenb@google.com/>), which was [originally posted in 2013](<https://lore.kernel.org/linux-mm/1383170047-21074-2-git-send-email-ccross@android.com/>). It adds a new [`prctl()`](<https://man7.org/linux/man-pages/man2/prctl.2.html>) operation: 

```
prctl(PR_SET_VMA, PR_SET_VMA_ANON_NAME, start, len, name);
```

This operation will cause the given `name` to be associated with the `len` anonymous pages beginning at `start`. In truth, the name is associated with the virtual memory area (VMA) structure describing a range of memory. Thus, what actually happens is that all pages that are part of the VMAs in the given range will have the name assigned to them, even if the pages themselves are not within that range. Each [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>) call usually creates a VMA (though there are complications), so all pages associated with any given VMA will normally have been created in the same way. 

The `maps` and `smaps` files in each process's `/proc` directory already contain a lot of information about that process's VMAs. With this patch set applied, those files will also contain the name that has been associated with the anonymous VMAs, if any; the name is duly checked for printability before being accepted. Using that information, system tools can associate pages with those names and, from there, with the subsystems that created them. 

Assigning a name to a VMA does not seem like a difficult endeavor, but it has proved to be the trickiest part of this patch. A system can have a lot of processes, each of which can have a lot of VMAs, so the management of these names needs to scale reasonably well. Previous versions of the patch set have tried just pointing to the provided names in user space; this avoids the need to allocate memory in the kernel but, as Kees Cook [pointed out](<https://lwn.net/ml/linux-mm/202009031031.D32EF57ED@keescook/>), it presents some interesting security problems as well. At the time, Cook suggested simply copying the strings into kernel space. 

While copying the strings works, there is still a little problem: when a process forks, its VMAs are copied for the new child. Now all of those name strings must be copied too. Baghdasaryan ran a worst-case test, with a process creating 64,000 VMAs, assigning a long name to each, then calling `fork()`, the result was a nearly 40% performance regression. Even if such numbers will not be seen in real-world workloads, a slowdown of that magnitude is sure to raise eyebrows. 

As a way of avoiding excessive eyebrow elevation, Baghdasaryan [added a mechanism](<https://lwn.net/ml/linux-kernel/20210827191858.2037087-4-surenb@google.com/>) to use shared, reference-counted names. A `fork()` call now need only increase the reference counts rather than allocate memory and copy a string. With this added machinery in place, the performance cost is ""reduced 3-4x"" in the worst case, and is said to not be measurable for more reasonable test cases. 

This functionality is evidently useful; Android has been using it for years, having kept the original patch going for all of that time. Thus far, the review comments have focused on relatively minor issues — which characters should be allowed in names, for example. So there would not appear to be a lot of obstacles to overcome before this work can be merged. For this feature, it seems, eight years of waiting on the sidelines should be enough, and anonymous pages may soon lose a bit of their anonymity.
