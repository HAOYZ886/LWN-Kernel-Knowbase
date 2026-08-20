---
title: Supporting Intel/AMD memory encryption
url: https://lwn.net/Articles/752683/
date: "April 25, 2018"
category: "Memory management-Memory encryption; Security"
author: "By Jonathan Corbet April 25, 2018 LSFMM"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
April 25, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

Once a niche feature, memory encryption is becoming mainstream with support in both Intel and AMD processors, Kirill Shutemov said at the beginning of his session during the memory-management track of the 2018 Linux Storage, Filesystem, and Memory-Management Summit. Memory encryption can harden the system against attack, but it also presents some interesting challenges for the kernel. 

Both Intel and AMD use the upper bits of the physical address to mark encrypted pages, he said. AMD processors currently only support a single encryption key, and so use a single bit to indicate that encryption is in use. Intel, instead, encrypts all of memory and uses up to six upper bits to indicate which encryption key is used for each page. The default key is marked by all zeroes in that field. 

One interesting challenge is that the CPU's memory caches are based on the physical address — including the encryption bit(s). Encryption is handled by the memory controller, and the same page with two different keys will look like different addresses to the CPU. Unless due care is taken, the same page can thus appear multiple times in the cache under different encryption keys. Writing multiple competing cache lines to the same page will likely corrupt the data there, an outcome that is likely to increase the overall user [![\[Kirill Shutemov\]](https://static.lwn.net/images/conf/2018/lsfmm/KirillShutemov-sm.jpg)](<https://lwn.net/Articles/752714/>) disgruntlement level. Avoiding it requires carefully flushing the caches whenever the encryption status of a page changes to ensure that no entries remain under the old key. 

Doing that flush turns out to be expensive. In an effort to minimize that cost, Shutemov is looking at adding encryption-key awareness to the page allocator. The key that was last used with each page would be remembered; if the page is allocated to a new use with the same key, there will be no need to flush any old cache entries. This can be implemented by putting a wrapper around the current page allocator. It is worth the effort, he said; otherwise allocation of encrypted pages can be up to three times as expensive. Since the intent is that all of memory will be encrypted, this extra cost could hurt overall performance significantly. 

One question that arises, though, is: where should the key ID be stored? It needs to be associated with the `page` structure somehow, and it must be kept around after the page has been freed. Ross Zwisler suggested that perhaps pages could be kept in separate pools, one for each key ID. Shutemov agreed that could be done, but it would involve more significant surgery to the page allocator. There was a period of somewhat rambling exploration of ideas for solutions with no real conclusion reached. 

Hugh Dickins asked how key IDs interact with the buddy allocator, which will want to join pages with different IDs into larger groupings. The buddy allocator ignores the IDs, Shutemov said. Cache flushing is done with a single-page granularity, though, so things work as expected. 

For the time being, the key ID is being stored in the `anon_vma` structure; that means that memory encryption only works for anonymous pages. It also is not yet tracking the key ID after pages are freed. Dave Hansen said that the search for a permanent home for the key ID is likely to lead to a challenge all memory-management developers have faced at one time or another: poring over `struct page` in search of a few available bits that can be used for this purpose. For now, though, Shutemov is looking at storing it in the `page_ext` structure that is used for information that doesn't fit in `struct page`. 

Michal Hocko worried that adding complexity to the page allocator may be a mistake, especially if memory encryption works better in future CPUs and this level of tracking is no longer needed. He also worried that encryption will add a degree of nondeterminism to the page allocator; the time required to satisfy a request will vary considerably depending on the previous encryption status of the allocated pages. The networking developers, who have been working to reduce allocation times, will complain. It would be better, he said, to ensure that the cost of using encrypted memory falls on those who choose to use it. That suggests just flushing caches when the memory is freed. 

Shutemov raised another issue: the direct mapping (the portion of kernel space that, on 64-bit systems, maps directly to all of physical memory) uses the default key. But the kernel will often find itself having to access pages that are encrypted with other keys. One way to handle that would be to bring back something like the `kmap()` interface to create a temporary mapping to a specific page using the appropriate key. That would be somewhat less efficient than going through the direct mapping, though, and will hurt performance. 

The alternative is to partition the direct mapping, creating one section for each possible key ID (of which there are up to 64 in current hardware). The promise of this approach is better, he said, but it's not working yet. The potential problem is that replicating the direct mapping in this way will use a lot of virtual address space, reducing the amount of physical memory that the machine can handle in the process. But, by grabbing a set of physical-address bits for the key ID, memory encryption already reduces the possible amount of physical memory anyway. The other possible disadvantage is that kernel address-space layout randomization would have fewer bits to play with. 

The proper API for key management still needs to worked out. With 64 keys available, they can't just be handed out to any process that wants one — at least, not without complicating context switches in unpleasant ways. The user-space API is likely to be based on the existing kernel key-management facilities. A new `mprotect()` call would be used to apply a key to a range of memory; doing so will destroy the current contents of the indicated range. It would also be nice to have a control-group API at some point, he said. 

The final challenge discussed was DMA, which also has to work with memory encryption. On systems with an I/O memory-management unit, encryption should just work. For other systems, it depends on whether the DMA mask for any specific device can handle the full range of physical addresses; encryption will just work if that is the case. Otherwise, bounce buffers will be needed so that the kernel can handle the encryption; that is easy to implement but slow to run.
