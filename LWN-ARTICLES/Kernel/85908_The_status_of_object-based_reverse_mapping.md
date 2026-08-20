---
title: "The status of object-based reverse mapping"
url: https://lwn.net/Articles/85908/
date: "May 19, 2004"
category: "Memory management-Object-based reverse mapping; Object-based reverse mapping; anon vma; anonmm"
author: 
---

The discussion has been quiet in recent times, but work on replacing the low-level reverse-mapping virtual memory code in the 2.6 kernel continues. When we last looked at the new, object-based reverse mapping ("objrmap") approach, there were two competing implementations: 

  * Andrea Arcangeli's [anon-vma](<https://lwn.net/Articles/75198/>), which adds a data structure creating a connection between each physical page and the virtual memory area (VMA) structures which reference it. 

  * Hugh Dickins's [anonmm](<https://lwn.net/Articles/77106/>), which associates pages with the top-level memory management ("mm") structure instead. 

The two approaches are conceptually similar, but each has its strong and weak points. Their performance is essentially equivalent. Thus far, there has not been any sort of spirited debate over which should be included; most kernel developers, if they have a preference, have kept it to themselves. 

Hugh has been busy over the last few weeks, however, creating a series of 40 patches aimed at slowly moving the reverse mapping code over to the object-based approach. The first five of those patches, which are restricted to cleanup and preparatory work, have been merged into the 2.6 mainline. "rmap-10" added anonmm; it was promptly merged into the -mm tree. This action did not imply that anonmm had been chosen over anon-vma, however; it was simply the first step in the testing process which would lead to a final decision. 

Hugh's final series of patches (rmap-34 to rmap-40) completes the process by replacing anonmm with anon-vma; these patches are present in 2.6.6-mm4. Hugh [introduces](<https://lwn.net/Articles/85715/>) the patch set by saying: 

Judge for yourselves which you prefer. I do think I was wrong to call anon_vma more complex than anonmm (its lists are easier to understand than my refcounting), and I'm happy with its vma merging after the last patch. It just comes down to whether we can spare the extra 24 bytes (maximum, on 32-bit) per vma for its advantages in swapout and mremap. 

As Hugh notes, anon-vma should have better swapping performance, since its structures make it easier to find the VMA for a given page. Additionally, the anonmm code works best when shared anonymous pages have the same virtual address in each address space that uses them; if a process moves pages with `mremap()`, some relatively complicated work must be performed to make things work. The anon-vma solution does not have that particular problem. 

On the other hand, expanding the VMA structure is not something which should be done lightly; some loads can use huge numbers of VMAs, and they must all be located in low memory. That said, either reverse mapping scheme should free far more low memory than it consumes; that is, after all, one of the main points behind this entire exercise. 

There still has been no public word on which scheme will be chosen, or when it might be merged. The current state of affairs suggests, however, that anon-vma will be the one that goes in unless some sort of major problem turns up. As for timing: enough major work has already gone into 2.6.7 that it's hard to imagine throwing major VM surgery into the mix. So 2.6.8 is the earliest such a merge could possibly happen. A couple of 2.6 releases after that, the forking of the 2.7 tree might just become a possibility.
