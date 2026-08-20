---
title: Tracking actual memory utilization
url: https://lwn.net/Articles/642202/
date: "April 29, 2015"
category: Memory management
author: "By Jonathan Corbet April 29, 2015"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
April 29, 2015

One might be tempted to think that an operating-system kernel should be able to answer a simple question: how much memory is a given process actually using? But, despite all the effort that has gone into providing visibility for this type of data, simple answers can be hard to come by. So the effort to provide better information continues, as can be seen by a recent [patch set](<https://lwn.net/Articles/642135/>) from Vladimir Davydov adding another way to calculate memory utilization. 

A process's resident set size (RSS) is relatively easily calculated; that is the number of pages of physical memory currently owned by that process. Interested parties can get this information now from `/proc` or the `ps` command. In theory, the kernel is handling page reclaim in such a way that each process is actually _using_ every page in its resident set, but, in the real world, things don't always work out that way. 

It can be worth knowing if there is a significant difference between a process's RSS and the amount of memory actually in use; this information can be helpful when partitioning the system between containers or setting control-group limits. As it happens, the kernel contains a mechanism designed to allow an observer to determine how much of a process's resident set has actually been referenced. That information is found in a virtual file called `smaps` in the process's `/proc` directory. For example, the following fragment comes from the `smaps` file corresponding the the X.org server on your editor's desktop: 

```
016bc000-04af4000 rw-p 00000000 00:00 0                      [heap]
        Size:              53472 kB
        Rss:               51936 kB
        Pss:               51936 kB
        Shared_Clean:          0 kB
        Shared_Dirty:          0 kB
        Private_Clean:         0 kB
        Private_Dirty:     51936 kB
        **Referenced:        45384 kB**
        Anonymous:         51936 kB
        AnonHugePages:     38912 kB
        Swap:                  0 kB
        KernelPageSize:        4 kB
        MMUPageSize:           4 kB
        Locked:                0 kB
        VmFlags: rd wr mr mw me ac
```

This entry describes an anonymous memory area that occupies 53,472KB of memory; 51,936KB of that area is currently resident (the `Rss` field), and 45,384KB have been referenced (the line in bold) since tracking was last reset. Since nothing is monitoring memory use on this system, that number has never been reset and thus counts every page referenced since the X.org server started. 

If one wants to track usage over a specific period, it is necessary to reset the "referenced" count at the beginning, let the process run for a bit, then look in `smaps` to see how much memory was actually touched. That reset is done by writing a value of `1` to the `clear_refs` file in the same `/proc` directory. 

At a first look, this mechanism seems like it should be able to answer the question of how much memory a process is actually using. But it turned out to not meet Vladimir's needs for a couple of reasons. One of those is that, while the `smaps` entry tracks references to memory mapped into the process's address space, it does not track page-cache memory used when files are accessed with system calls like `read()` or `write()`. That memory, too, is used by the process, so there would be value in knowing how much of it there is. Perhaps more importantly, the "referenced" state of each page is used by the memory-management subsystem itself to make decisions on which pages to evict. Resetting every page to the "not referenced" state will thus perturb page reclaim, and probably not for the better. If these measurements are to be made often, it would be good to have a less invasive way to make them. 

Vladimir's patch adds a new file called `/proc/kpageidle`; since it's in the top-level `/proc` directory, it's a single file that describes an aspect of the the global state of the system. The file can be read like a long array of 64-bit integer values; each value corresponds to one physical page in the system, indexed by page-frame number. If a program wants to know whether physical page _N_ has been referenced, it can seek to the appropriate location in `/proc/kpageidle` and read the value there; if the lowest bit is set, the page is idle. (Note that [this file may change to a bitmap format](<https://lwn.net/Articles/642358/>) in a future version of the patch set). 

Once again, one needs to be able to reset that state to make observations over a given time period; in this case, setting a page to the "idle" state is done by writing `1` to the appropriate location in `/proc/kpageidle`. That action will make the page inaccessible (much like the normal kernel usage tracking does) so that a fault will result whenever a process tries to read or write that page. At that point, the "idle" state can be reset and the page made accessible again. The idle state will also be reset if the page is accessed via the file-related system calls, so it will track the state of pages in the page cache as well. 

To track the idle state, the patch set adds a new "idle" page flag that is set whenever a page is marked idle. That flag is then passed back to user space whenever a given page's entry in `/proc/kpageidle` is read. As it turns out, there is a need for a second page flag as well, though. As mentioned above, making a page inaccessible is a technique already used within the memory-management subsystem; when a write to `/proc/kpageidle` causes that to happen, it makes the page appear to have _never_ been accessed. To avoid that, Vladimir adds a second flag called "young"; whenever a write to `/proc/kpageidle` makes a page inaccessible, the "young" bit will be set as well. When the memory-management code asks whether a page has been referenced, the "young" bit is taken into account. In the end, that means that using `/proc/kpageidle` will not change how page reclaim is done. 

There is one little problem with this approach: page flags are in short supply on 32-bit systems. To get around this problem, the code uses the "`struct page` extension" mechanism in the 32-bit case. This mechanism was originally created to support memory control groups (memcgs), which need to store more information about each page than can fit in the `page` structure. Using extensions can use quite a bit of memory in its own right, but there's little alternative on systems where shoehorning even one more bit into `struct page` is not an option. 

Readers who have gotten this far may be wondering about one final piece of the puzzle: knowing which physical pages in the system are in use does not say much about what any specific processes are using. There are two ways of connecting the two pieces, one of which exists now and one which is part of Vladimir's patch. In current systems, the `pagemap` file in any process's `/proc` directory can be used to see which physical pages are mapped into that process's address space. That information is [only available to privileged processes](<https://lwn.net/Articles/642069/>) as of the 4.0 release, but `/proc/kpageidle` is a privileged interface too. 

If the task at hand is partitioning a system's resources, though, then memcgs are likely already in use to set limits on groups of processes. In that case, it is more interesting to know how much memory each memcg is using than to track this information on a per-process basis. To that end, the patch set adds yet another file (`/proc/kpagecgroup`) which, when read, yields the control group that owns each page. By using that file together with `/proc/kpageidle`, a monitoring process can determine how many pages each memcg is using — and how many it owns but is not making use of. 

The end result is an interface that can be used to determine how well a control group's memory limits fit its actual needs. As service providers of all types seek to run more clients on each physical system, they will likely be pleased to have this extra information available. That, of course, depends on this patch set being merged into the mainline. Given the lack of significant opposition, that seems likely to happen sooner or later — though, with memory-management patches, it's always hard to say just when that might happen.
