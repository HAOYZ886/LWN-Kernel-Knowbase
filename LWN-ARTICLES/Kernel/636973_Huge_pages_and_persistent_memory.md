---
title: Huge pages and persistent memory
url: https://lwn.net/Articles/636973/
date: "March 17, 2015"
category: "Memory management-Nonvolatile memory"
author: "By Jonathan Corbet March 17, 2015 LSFMM 2015"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
March 17, 2015

* * *

[LSFMM 2015](<https://lwn.net/Articles/lsfmm2015/>)

One of the final sessions in the memory-management track of LSFMM 2015 had to do with the intersection of persistent memory and huge pages. Since persistent memory looks set to come in huge sizes, using huge pages to deal with it looks like a performance win on a number of levels. But support for huge pages on these devices is not yet in the kernel. 

Matthew Wilcox started off by saying that he has posted [a patch set](<https://lwn.net/Articles/615323/>) adding huge-page support to the DAX subsystem. But, he said, only one other developer reviewed the code. The biggest complaint was the introduction of the `pmd_special()` function, which tracks a "special" bit at the page middle directory (PMD) level in the page table hierarchy, which is where huge pages are managed. 

Some background: the kernel allows architecture-level code to mark specific pages as being "special" by providing a `pte_special()` function. These pages have some characteristic that causes them to behave differently than ordinary memory. In cases where the architecture has enough bits available in its page table entries, `pte_special()` just checks a bit there; otherwise things get more complicated. The core memory-management code treats so-marked pages, well, specially; for example, virtual memory areas containing "special" pages should also have a `find_special_page()` method to get the associated `struct page`. 

Back to the discussion: adding `pmd_special()` requires that the "specialness" of the huge page be tracked at the PMD level. It is not clear that every architecture has a free bit available in the PMD to track that state. In theory, free bits should abound there since as many as 20 bits in the lower part of the entry are not needed to map to a page frame [![\[Matthew Wilcox\]](https://static.lwn.net/images/conf/2015/lsfmm/MatthewWilcox-sm.jpg)](<https://lwn.net/Articles/636975/>) number, but some quick searching by developers in the room revealed that, on x86 at least, the "extra" bits must be set to zero. For now, though, Matthew is using the same bit that `pte_special()` uses, so his code should work on every architecture that supports `pte_special()`. 

In the case of huge pages backed by persistent memory, the `pmd_special()` bit indicates to the memory-management code that there is no associated `page` structure. Andrea Arcangeli asked why a special bit was needed to mark that condition; Matthew responded that it's because he doesn't really understand the memory-management subsystem, so he implemented something he knew he could make work. 

This code may eventually be pushed in a direction where `pmd_special()` is no longer needed. But there are some other issues that come up. Matthew raised one: what happens when an application creates a `MAP_PRIVATE` mapping of a file into memory, then writes to a page in that mapping? The write will cause the memory-management code to allocate anonymous memory to replace the 2MB huge page being written to; the question is: should it allocate and copy a full 2MB page, or just copy the 4KB page that was actually written? Andy Lutomirski suggested that the answer had to be to copy 4KB; copying the full 2MB for each single-page change would be too expensive. But Kirill Shutemov replied that copy-on-write for huge pages does a 2MB copy now; the behavior with persistent memory, he said, should be consistent. 

Matthew moved on to the topic of in-kernel uses for persistent memory. There will be some interesting ones, he thought, but how it should all work has yet to be worked out. HP, for example, is using `ioremap()` to map persistent memory into the kernel as if it were device memory; Matthew said that seems like the wrong approach to him. We should, he said, be using logical interfaces to persistent memory rather than direct physical interfaces like `ioremap()`. So he would like to see the creation of some sort of mapping interface implemented within the virtual filesystem layer that would allow persistent memory to be mapped into the kernel's address space. 

Andy said that the [pstore mechanism](<https://lwn.net/Articles/434821/>) could benefit from directly-mapped persistent memory. There was also talk of maybe being able to load kernel modules from persistent memory without the need to copy them into "regular" memory. It might be possible to even map the entire kernel, but there is one little catch: the kernel patches its own code for a number of reasons, including use of optimal instructions for the specific hardware in use and turning tracepoints on and off. If the kernel were mapped from persistent memory, that patching would change the version stored in the device as well — probably not the desired result. 

Finally, Matthew said, there have been requests for the ability to use extra-huge, 1GB pages as well as 2MB pages. He is looking at adding that functionality, but he has been struck by the amount of code duplication that exists at each of the four page table levels. He has some thoughts about creating a level-independent "virtual page table entry" abstraction that could be used to get rid of much of that duplication. The reaction from the assembled memory-management developers was cautiously positive; Matthew was encouraged to implement this abstraction within the DAX code. If it works out well there, it can then spread into the rest of the memory-management code.
