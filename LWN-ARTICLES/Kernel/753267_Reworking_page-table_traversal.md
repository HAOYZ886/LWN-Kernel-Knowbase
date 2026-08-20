---
title: "Reworking page-table traversal"
url: https://lwn.net/Articles/753267/
date: "May 4, 2018"
category: "Memory management-Internal API"
author: "By Jonathan Corbet May 4, 2018 LSFMM"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 4, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

A system's page tables are organized into a tree that is as many as five levels deep. In many ways those levels are all similar, but the kernel treats them all as being different, with the result that page-table manipulations include a fair amount of repetitive code. During the memory-management track of the 2018 Linux Storage, Filesystem, and Memory-Management Summit, Kirill Shutemov proposed reworking how page tables are maintained. The idea was popular, but the implementation is likely to be tricky. 

On a system with five-level page tables (which few of us have at this point, since Shutemov just added the fifth level), a traversal of the tree starts at the page global directory (PGD). From there, it proceeds to the P4D, the page upper directory (PUD), the page middle directory (PMD), and finally to the PTE level that contains information about individual 4KB pages. If the kernel wants to unmap a range of page-table entries, it may have to make changes at multiple levels. In the code, that means that a call to [`unmap_page_range()`](<https://elixir.bootlin.com/linux/latest/source/mm/memory.c#L1496>) will start in the PGD, then call [`zap_p4d_range()`](<https://elixir.bootlin.com/linux/latest/source/mm/memory.c#L1477>) [![\[Kirill
Shutemov\]](https://static.lwn.net/images/conf/2018/lsfmm/KirillShutemov2-sm.jpg)](<https://lwn.net/Articles/753268/>) to do the work at the P4D level. The calls trickle down through [`zap_pud_range()`](<https://elixir.bootlin.com/linux/latest/source/mm/memory.c#L1448>) and [`zap_pmd_range()`](<https://elixir.bootlin.com/linux/latest/source/mm/memory.c#L1411>) before ending up in [`zap_pte_range()`](<https://elixir.bootlin.com/linux/latest/source/mm/memory.c#L1281>). All of the levels in this traversal (except the final one) look quite similar, but each is coded separately. There is a similar cascade of functions for most common page-table operations. Some clever coding ensures that the unneeded layers are compiled out when the kernel is built for a system with shallower page tables. 

Shutemov would like to replace this boilerplate with something a bit more compact. He is proposing representing a pointer into the page tables (at any level) with a structure like: 

```
struct pt_ptr {
            unsigned long *ptr;
    	int lvl;
        };
```

Using this structure, page-table manipulations would be handled by a single function that would call itself recursively to work down the levels. Recursion is generally frowned upon in the kernel because it can eat up stack space, but in this case it is strictly bounded by the depth of the page tables. That one function would replace the five that exist now, but it would naturally become somewhat more complex. 

He asked: would this change be worth it? Michal Hocko asked just how many years of work would be required to get this change done. Among other things, it would have to touch every architecture in the system. If it proves impossible to create some sort of a compatibility layer that would let architectures opt into the new scheme, an all-architecture flag day would be required. Given that, Hocko said that he wasn't sure it would be worth the trouble. 

Laura Abbott asked what problems would be solved by the new mechanism. One is that it would deal more gracefully with pages of different sizes. Some architectures (POWER, for example) can support multiple page sizes simultaneously; this scheme would make that feature easier to use and manage. Current code has to deal with a number of special cases involving the top-level table; those would mostly go away in the new scheme. And, presumably, the resulting code would be cleaner. 

It was also said in jest that this mechanism would simplify the work when processors using six-level page tables show up. The subsequent discussion suggested that this is no joking matter; it seems that such designs are already under consideration. When such hardware does appear, Shutemov said, there will be no time to radically rework page-table manipulations to support it, so there will be no alternative to adding a sixth layer of functions instead. In an effort to avoid that, he is going to try to push this work forward on the x86 architecture and see how it goes.
