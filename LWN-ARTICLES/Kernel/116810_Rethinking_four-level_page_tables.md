---
title: "Rethinking four-level page tables"
url: https://lwn.net/Articles/116810/
date: "December 22, 2004"
category: "Large-memory systems; Memory management-Four-level page tables"
author: 
---

Andi Kleen's four-level page table patch has been in the -mm tree for some time; it is widely understood to be one of the first things in the queue to be merged once 2.6.10 is out. For those who are not familiar with this patch, why it matters, and how it works, a look at [this LWN Kernel Page article from last October](<http://lwn.net/Articles/106177/>) might be helpful in understanding the following discussion. 

The three levels of page table currently implemented by the kernel are, from top to bottom, the PGD, PMD, and PTE. Andi's patch extends the hierarchy by adding a new top-level directory called PML4 (from the x86-64 specification). A system which currently has a single PGD (per virtual address space) will have, instead, a single PML4 directory which may contain pointers to many PGD directories. In the current implementation, the PMD vanishes transparently on systems which only have two-level page tables; as a result, the kernel can treat all systems as if they had three-level page tables. Andi's four-level patch works in a similar way, causing the new PML4 level to be optimized out on hardware which does not support it. 

Nick Piggin has recently posted [a new, alternative four-level patch](<https://lwn.net/Articles/116541/>). Nick is not hugely upset by Andi's patch set, but he thinks he has a better way. Essentially, Nick thinks that it would be better to keep the PGD as the top-level page directory, and to insert the new level in the middle, next to the PMD. With this organization, all architectures would have an active PGD at the top of the hierarchy, and active PTEs at the bottom, but the PMD and the PUD (Nick's name for the new level) would be optimized out on systems which do not use them. 

Andi [would prefer to stick with the current](<https://lwn.net/Articles/116815/>) patches; he sees Nick's approach as being mainly an exercise in renaming which could delay the merging of the four-level capability. The current patches have been shaken down well in the -mm tree and seem to work; thrashing them up now would require a new round of testing before they had the same level of confidence. Andi has other work which is waiting for the four-level patch to be merged, so he would rather not see the whole process slowed down. 

Others are in less of a hurry, however, and see merit in Nick's patches. In particular, Linus [prefers placing the new level below the PGD](<https://lwn.net/Articles/116817/>) as the least intrusive way of extending the page table hierarchy. 

Basically, by doing the new folded table in the middle, it _only_ affects code that actually walks the page tables. Basically, what I wanted in the original 2->3 level expansion was that people who don't use the new level should be able to conceptually totally ignore it. I think that is even more true in the 3->4 level expansion. 

Andi has not yet given in, but there seems to be a strong wind blowing in favor of Nick's page table arrangement. So four-level page tables might not be the first thing to go into 2.6.11 after all.
