---
title: A proposal for a major memory management rework
url: https://lwn.net/Articles/125488/
date: "March 1, 2005"
category: "Memory management-Internal API"
author: 
---

As has been described in [previous Kernel Page articles](<https://lwn.net/Articles/106177/>), the Linux kernel works with a four-level, hierarchical page table mechanism. A virtual address is translated to a physical address by walking down the table until the relevant page table entry is found. When running on hardware which does not implement a four-level tree, the kernel transparently "folds" the missing layers out of existence. So the same high-level memory management code runs on all hardware, regardless of the depth of page table tree that hardware implements. 

There is one interesting issue with this scheme: not all hardware uses this sort of hierarchical page table mechanism. It matches the i386 hardware well \- to the point that the processor works directly from the same page tables that the generic kernel memory management code manipulates. Other processors have different ways of handling address translation, however. The ia-64 architecture uses a linear page table which is, itself, mapped in virtual memory; there is a "virtual hashed page table walker" hardware function which can quickly resolve page faults in many situations. The hierarchical page tables carefully maintained by the core kernel are never used directly by the hardware; instead, the architecture-specific code takes care of moving information between the core kernel tables and the hardware versions. This impedance matching requires extra code and work; it also makes it harder to take advantage of any high-level features that the hardware may offer. 

(See [this chapter from _ia-64 Linux Kernel_](<http://www.phptr.com/articles/article.asp?p=29961&seqNum=4>) for a detailed description of how the ia-64 architecture handles page tables). 

Christoph Lameter would like to get rid of the disconnect between in-kernel and hardware page tables; to that end, he has proposed [a new abstraction layer](<https://lwn.net/Articles/124961/>) which would handle access to the processor's memory management unit (MMU). With the new layer in place, there would be no more hierarchical page tables in the core kernel. If the hardware uses hierarchical tables, the architecture-specific code would still work with them, but they would be hidden from the core. The proposed replacement interface is somewhat vague at this stage, but some features have been sketched out: 

  * A new type, `mmu_entry_t` would represent a translation from a virtual address to the corresponding physical address. It thus functions like a page table entry, but it could contain information not necessarily found in page table entries now, such as "large page" information and, possibly, statistics information. 

  * A translation set (`mmu_translation_set_t`) represents the address space for a process; it is a collection of `mmu_entry_t` values and required housekeeping information. 

  * The new interface would also implement transactions (`mmu_transaction_t`), so that complex changes to page tables could be performed in an atomic manner. The transaction abstraction hides the page table locking within the architecture-specific code, since that locking may be done in very different ways. 

Initially, the new interface would be implemented on top of the existing hierarchical page tables. The transition could thus be made a little smoother, and architectures which actually use the hierarchical tables could continue to function as always. Eventually, however, direct access to those tables from the core kernel code would be removed, and architectures with different ideas of how page tables should be managed would be able to drop the hierarchical tables. 

Once the transition has been made, other things would become possible as well. The current memory management system is really only comfortable when pages are all the same size. The support for huge pages has been bolted on to the side, and it does not really hide the fact that different processors handle large pages in very different ways. The new scheme would present a simple `mksize()` function to change the size of a page, and would hide from the kernel the details of how that size change is actually done. In addition, the new scheme would allow for global pages which appear in every process's address space, and for keeping statistics of the various types of pages in the system. 

Discussion of the proposal has been muted. Actually, it has been almost nonexistent. Unfortunately, things often happen that way when abstract proposals are posted to the kernel lists. Kernel developers respect actual code far more than design ideas; they will often wait until an implementation is posted for review, then talk about how it _should_ have been done. So the new memory management interface may have to make some more progress before the discussion can truly begin.
