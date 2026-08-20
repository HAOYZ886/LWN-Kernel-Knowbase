---
title: The ongoing trouble with get_user_pages()
url: https://lwn.net/Articles/930667/
date: "May 4, 2023"
category: "Memory management-get user pages"
author: "By Jonathan Corbet May 4, 2023"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 4, 2023

The 2018 Linux Storage, Filesystem, and Memory-Management (LSFMM) conference included [a session on `get_user_pages()`](<https://lwn.net/Articles/753027/>), an internal kernel interface that can, in some situations, be used in ways that will lead to data corruption or kernel crashes. As the [2023 LSFMM+BPF event](<https://events.linuxfoundation.org/lsfmm/>) approaches, this problem remains unsolved and is still the topic of ongoing discussion. [This patch series](<https://lwn.net/ml/linux-kernel/cover.1683044162.git.lstoakes@gmail.com/>) from Lorenzo Stoakes, which is another attempt at a partial solution, is the latest focus point. 

#### The problem

The `get_user_pages()` API comes in a number of variants; this API family is often referred to as "GUP". Its purpose is to provide the kernel with direct access to user-space memory; that involves ensuring that the relevant pages are resident in RAM and cannot be evicted for as long as that access is needed. The root of the problem with `get_user_pages()` is that it creates a situation where there are two separate paths to the memory in question. 

User-space processes access their memory by way of virtual addresses mapped in their page tables. Those addresses are only valid within the process owning the memory. The page tables provide a convenient handle when the kernel needs to control access to specific ranges of user-space memory for a while. A common example is writing dirty file pages back to persistent store. A filesystem will mark those pages (in the relevant page table) as read-only, preventing further modification while the writeback is underway. If the owning process attempts to write to those pages, it will be blocked until writeback completes; thereafter, the read-only protection will cause a page fault, allowing the filesystem to be notified that the page has been dirtied again. 

A call to `get_user_pages()` will return pointers to the kernel's `page` structures representing the physical pages holding the user-space memory, which can be used to obtain kernel-virtual addresses for those pages. Those addresses are in the kernel's address space, usually in the kernel's direct map that covers all of physical memory (on 64-bit systems). They are not the same as the user-space addresses, and are not subject to the same control. Direct-mapped memory that does not hold executable text is (almost) always writable by the kernel. 

User space can use [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>) to map a file into its address space, creating a range of file-backed pages. Those pages will be initially marked read-only, even if mapped for write access, so that the filesystem can be notified when one of them is changed. If the kernel uses `get_user_pages()` to obtain write access to file-backed pages, the underlying filesystem will be duly notified that the pages have been made writable. At some future time, that filesystem will write the pages back to persistent storage, making them read-only. That protection change, though, applies only to the _user-space_ addresses for those pages. The mapping in the kernel's address space remains writable. 

That is where the problem arises: if the kernel writes to its mapping after the filesystem has written the pages back, the filesystem will be unaware that those pages have changed. Kernel code can mark the pages dirty, possibly leading to filesystem-level surprises when a seemingly read-only page has been written to. There are also a few scenarios in which the pages may never get marked as dirty, despite having been written to, in which case the written data may never find its way out to disk. Either way, the consequences are unfortunate. 

This problem has been the subject of a long series of LSFMM discussions and an equally interminable [set of LWN articles](<https://lwn.net/Kernel/Index/#Memory_management-get_user_pages>), but it is not an easy one to solve. There are times when the kernel simply needs access to user-space memory, often for performance purposes. A frequently repeated example is using RDMA to read data directly into file-backed pages. Allowing a DMA-capable device to write data directly into a user-space page requires pinning that page, perhaps for a long time. Finding a reliable way to enable this kind of back channel into user-space has proved difficult. 

#### A partial solution?

In late April, Stoakes decided to face part of the problem head-on, posting [a patch](<https://lwn.net/ml/linux-kernel/f86dc089b460c80805e321747b0898fd1efe93d7.1682168199.git.lstoakes@gmail.com/>) that would simply disallow `get_user_pages()` calls that request write access to file-backed pages. Recognizing, though, that there are some cases that require exactly this kind of mapping, he also included a new flag, `FOLL_ALLOW_BROKEN_FILE_MAPPING`, to override the prohibition; some InfiniBand controllers were updated to use that flag. Making this change, Stoakes said, ""is an important step towards a more reliable GUP, and explicitly indicates which callers might encounter issues moving forward"". 

Over the following week or so, the series went through several revisions. The most significant, perhaps, was to drop the `FOLL_ALLOW_BROKEN_FILE_MAPPING` flag and, instead, only prohibit `get_user_pages()` calls that provide the `FOLL_LONGTERM` flag (and which request write access to file-backed pages), indicating that the mapping is likely to persist for a long time. Shorter-term mappings are not immune to the problem but, by virtue of being short-lived, they are much less likely to trigger it. This change was an acknowledgment that it is still not possible to fully solve — or even block — the problem. 

This proposal has provoked a fair amount of discussion. Christoph Hellwig [worried](<https://lwn.net/ml/linux-kernel/ZEZPXHN4OXIYhP+V@infradead.org/>) that it would break users who are using direct I/O to write into file-backed mappings, but Jason Gunthorpe [questioned](<https://lwn.net/ml/linux-kernel/ZEZ117OMCi0dFXqY@nvidia.com/>) whether any such users exist, saying that people who tried it ""didn't get very far before testing uncovered data corruption and kernel crashes"". David Hildenbrand, instead, [suggested](<https://lwn.net/ml/linux-kernel/afcc124e-7a9b-879c-dfdf-200426b84e24@redhat.com/>) that some virtualization setups could be broken by the change; once again, Gunthorpe [doubted](<https://lwn.net/ml/linux-kernel/ZEvZtIb2EDb%2FWudP@nvidia.com/>) that any such use cases could be working successfully now. 

Hildenbrand [had some other concerns](<https://lwn.net/ml/linux-kernel/094d2074-5b69-5d61-07f7-9f962014fa68@redhat.com/>) about the patch, including the fact that it does not solve the full problem: ""If we want to sell this as a security thing, we have to block it *completely* and then CC stable. Everything else sounds like band-aids to me."". He complained that it does not address the "GUP-fast" subset of `get_user_pages()` APIs — an omission that Stoakes later fixed. He suggested that bringing the topic to this year's LSFMM+BPF conference (which starts on May 8) would be a logical next step. 

Ted Ts'o [described](<https://lwn.net/ml/linux-kernel/ZEybNZ7Rev+XM4GU@mit.edu/>) an ext4 bug that had resulted from this problem; the filesystem was not prepared for pages to be marked dirty at unexpected times and could be made to crash. [A fix](<https://git.kernel.org/linus/cc5095747edf>) was merged into 5.18 to prevent the crash but, Ts'o said, that might not have been the right thing to do, since it ""has apparently removed some of the motivation of really fixing the problem instead of papering over it"". He stated that, in the view of the filesystem developers, writing to file-backed pages via `get_user_pages()` is a bug and ""you get to keep both pieces"". 

Gunthorpe [took Ts'o's words](<https://lwn.net/ml/linux-kernel/ZE2ht9AGx321j0+s@nvidia.com/>) as yet another reason to block write access to file-backed pages: 

> This alone is enough reason to block it. I'm tired of this round and round and I think we should just say enough, the mm will work to enforce this view point. Files can only be written through PTEs. If this upsets people they can work on fixing it, but at least we don't have these kernel problems and inconsistencies to deal with. 

There is still not a complete agreement, though, that even the partial block that is on the table should be merged. The worries that it could end up breaking user-space applications, or that merging the relatively easy fix could delay the implementation of a complete solution, are not going to just vanish. So it seems that yet another LSFMM+BPF discussion is inevitable; indeed, Stoakes [seems to be looking forward](<https://lwn.net/ml/linux-kernel/b11d8e94-1324-41b3-91ba-78dbef0b1fc0@lucifer.local/>) to it: ""I think discussion at LSF/MM is also a sensible idea, also you know, if beers were bought too it could all work out nicely :]"". So this long-term discussion is, it seems, not over yet.
