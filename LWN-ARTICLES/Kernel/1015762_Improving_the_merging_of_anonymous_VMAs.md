---
title: Improving the merging of anonymous VMAs
url: https://lwn.net/Articles/1015762/
date: "March 31, 2025"
category: "Memory management-Object-based reverse mapping"
author: "By Jonathan Corbet March 31, 2025 LSFMM+BPF"
---

By **Jonathan Corbet**  
March 31, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

The virtual memory area (VMA), represented by [`struct vm_area_struct`](<https://elixir.bootlin.com/linux/v6.13.7/source/include/linux/mm_types.h#L670>), is one of the core abstractions of the kernel's memory-management subsystem; a VMA represents a portion of a process's address space with the same characteristics. A memory-mapped file will be represented by (at least) one VMA, as will the process's stack or a region of anonymous memory. Efficiently managing VMAs and the logic around them is crucial for good performance overall. Lorenzo Stoakes focused on one specific problem area: the merging of anonymous VMAs, during the memory-management track at the 2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit. 

Each VMA structure occupies non-movable kernel memory and increases the amount of memory-management overhead, so there is value in keeping them to a minimum. So, for example, if a process calls [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>) to allocate a range of memory, the kernel will check whether the new range adjoins an existing range with a compatible VMA; if so, the kernel will extend the existing VMA to cover the new address range rather than allocating a new VMA. There are also scenarios, as we will see, where the kernel can merge existing VMAs to reduce the overall number. 

#### Some background

Before getting into Stoakes's topic, though, there are a couple of other bits of context that will, hopefully, make the discussion more understandable. Specifically, the topic at hand deals heavily with the [`anon_vma`](<https://elixir.bootlin.com/linux/v6.13.7/source/include/linux/rmap.h#L17>) and [`anon_vma_chain`](<https://elixir.bootlin.com/linux/v6.13.7/source/include/linux/rmap.h#L69>) structures. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

[![\[Lorenzo Stoakes\]](https://static.lwn.net/images/conf/2025/lsfmm/LorenzoStoakes-sm.png)](<https://lwn.net/Articles/1015766/>) Given a folio in memory, the kernel often needs to know which VMAs contain mappings to it. If a folio is about to be swapped out or migrated, for example, the kernel has to find all of the mappings so that the associated page-table entries can be updated. Many years ago, this required scanning all of the page tables in the system, which turned out, surprisingly, to be rather inefficient. So, in 2004, [the `anon_vma` structure was introduced](<https://lwn.net/Articles/75198/>) by Andrea Arcangeli. In short, each anonymous folio (or page) contains a pointer to this structure in its `mapping` field; the `anon_vma` structure, in turn, keeps track of all of the VMAs that map the folio. 

At least, that is how it worked initially. The existence of copy-on-write memory complicates life, in that the same VMA in parent and child processes can refer to a combination of folios, some of which are shared and some of which have been written to, and thus copied. Trying to track all of this with a single `anon_vma` structure became unwieldy and scaled poorly so, in 2010, Rik van Riel created the `anon_vma_chain` which, as its name would suggest, tracks a list of `anon_vma` structures, each of which may point to VMAs that map any given folio. 

It is worth mentioning that any given anonymous VMA may not, on creation, have an associated `anon_vma` structure. The `anon_vma` is only created when the memory is faulted into that VMA. In the discussion below, a "faulted" VMA is one where this has happened — at least one folio has been faulted into the address space, and an `anon_vma` structure exists. An "unfaulted" VMA has no `anon_vma` structure. 

Finally, there are two relevant structure fields that are needed to know just where any given folio falls within its VMA. The folio itself has a field called `index`, which gives that folio's location within the VMA. That index, though, is adjusted by the `pgoff` field stored in the VMA itself; `pgoff` can be thought of as the origin for the `index` field; that origin might be outside of the VMA itself. Subtracting `pgoff` from `index` will give the true offset of the folio from the beginning of the VMA. 

#### Anonymous-VMA merging

Stoakes started with one of the core rules that applies when the kernel is considering merging two adjacent VMAs. Beyond the usual checks (are the VMAs of the same type, with the same permissions?), the kernel checks the `anon_vma` pointers. If both of the VMAs being considered for merging are faulted (thus having non-null `anon_vma` pointers), the two pointers must be the same. If one of the `anon_vma` pointers is null, then the merge can still happen, as long as the `pgoff` field of the upper VMA (the one at the higher virtual address) is equal to the sum of the `pgoff` and length of the lower VMA. 

Things get trickier, though, in the presence of the [`mremap()`](<https://man7.org/linux/man-pages/man2/mremap.2.html>) system call. This call can move a VMA within the address space, that can cause `pgoff` to fail to line up as described above, preventing a VMA merge that could otherwise take place. In the absence of an `anon_vma` pointer, a VMA's `pgoff` can simply be changed, but that will not work if the VMA is faulted. In that case, the `index` field of the faulted-in folios indicate where that folio fits within the VMA; it is relative to `pgoff`, so changing `pgoff` would corrupt the address space. The end result is that merging can almost never happen when `mremap()` is used to move a faulted VMA. 

Matthew Wilcox asked how common this type of moving is; once upon a time, `mremap()` did not even allow it. But it is allowed now; David Hildenbrand pointed out that this kind of movement can be used in attempts to defragment memory in user space. 

Beyond better efficiency, Stoakes continued, there are other reasons to improve this situation. An `mremap()` call cannot cross a VMA boundary, so unmerged VMAs can prevent operations that would otherwise be allowed. In the worst case, he said, this failure to merge could be seen as breaking the kernel's user-space ABI. 

Stoakes was there to present a ""crazy idea"" that he hoped could address this problem; it has been implemented in [this patch set](<https://lwn.net/ml/all/cover.1742478846.git.lorenzo.stoakes@oracle.com>) posted just before the conference. It adds a new flag to `mremap()` called `MREMAP_RELOCATE_ANON`; if this flag is present, `mremap()` will walk the page tables to update the VMA and folio offsets, with the intent that the moved VMA could then be merged into another one adjacent to its new location. It would be a best-effort attempt, which could fail if the needed resources are not available. 

User space could use this flag when it hopes that merging can happen, and when it is willing to pay the cost of the page-table walk. He pointed out that, sometimes, user space will move several VMAs around with multiple `mremap()` calls. Often, in this case, only the last call would need the new flag and have to pay that extra cost. This feature would be especially appreciated, he said, for ""a coffee-oriented language"" that does these kinds of moves. 

Hildenbrand expressed concern that this patch series adds a lot of complexity for a single, specific use case. He suggested that perhaps an [`madvise()`](<https://man7.org/linux/man-pages/man2/madvise.2.html>) operation could be provided instead as a way of separating this complexity from `mremap()`. Wilcox, instead, suggested simply deferring all of this work until the kernel sees an `mremap()` call that crosses a VMA boundary; at that point, the kernel could try adjusting the mappings before failing the call altogether. There would be an additional advantage that the call would make it clear which VMAs need to be merged at that time. 

The first implementation, Stoakes said, is cautious and does not attempt to handle cases with longer `anon_vma_chain` lists. It relocates the VMA `pgoff` and the associated folio `index` fields regardless of whether it is able to merge the VMAs or not. Transparent huge pages are correctly handled (and improve performance overall) — but this series needs a lot of testing, he said. Overall, the performance overhead is small, especially when huge pages are in use, so perhaps this remapping could be done opportunistically even when user space does not request it. 

Hildenbrand asked whether there might be a way to avoid walking the page tables and adjusting the `index` in every folio found there. Van Riel said that the problem comes about when a folio is mapped into both a parent and a child process, and the child remaps it. At that point, the folio exists at two different addresses in the two processes, but it only has one `index`. Thus, both the `pgoff` and `index` fields are needed to properly place a folio within a VMA. 

Vlastimil Babka said that he was advising a student some years ago who was trying to implement a similar solution; that work got as far as an initial patch posting. But the workload it was improving was proprietary, and nobody was able to find an equivalent open-source workload. There is little desire to merge kernel features just to serve proprietary workloads; among other things, that use could end in a few years and nobody would know that the feature is no longer needed. He asked whether Stoakes's use case was more open; Stoakes admitted that there may be a similar problem this time around. 

Kalesh Singh said that he might have a suitable use case. The Android system uses [`userfaultfd()`](<https://man7.org/linux/man-pages/man2/userfaultfd.2.html>) to manage app address spaces, and it goes out of its way to try to enable merging of VMAs when possible. That is non-proprietary code, and could make a good test case for this new feature. Michal Hocko agreed, and suggested that, given the time, this would be a good place to end this session.
