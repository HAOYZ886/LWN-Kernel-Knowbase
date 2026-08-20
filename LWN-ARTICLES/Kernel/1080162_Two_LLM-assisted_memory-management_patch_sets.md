---
title: "Two LLM-assisted memory-management patch sets"
url: https://lwn.net/Articles/1080162/
date: "July 2, 2026"
category: "Development tools-Large language models; Memory management-Large allocations; Memory management-Virtualization"
author: "By Jonathan Corbet July 2, 2026"
---

> **For humans, by humans**
> 
> Every article on LWN.net is written for humans, by humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the slop at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
July 2, 2026

The kernel community (like many other free-software projects) has recently seen a large influx of patches developed with the assistance of large language models (LLMs). Those patches tend to come from developers who were previously unknown to the community. At the moment, though, the memory-management developers are evaluating two large patch sets, developed with LLM assistance, that were submitted by established and well-respected developers. The rather different reception accorded to that work may give insights into how LLM-generated contributions will be handled going forward. 

#### Reliable 1GB allocations

As a Linux system runs, it tends to fragment its memory, making the allocation of large, physically contiguous chunks of memory difficult. Much work has been done over the years to improve this situation; the kernel tries hard to avoid fragmentation, and to actively defragment memory as needed. Larger allocations are more reliable than they once were. Still, allocation challenges can, for example, make it harder for the system to provide 2MB PMD-level huge pages than would be ideal. 

Given that difficulty, it might seem that reliable allocation of PUD-level (1GB) huge pages is an impossible goal. But, as Rik van Riel said in the [cover letter](<https://lwn.net/ml/all/20260520150018.2491267-1-riel@surriel.com>) to a 40-part patch series, there are workloads that can gain significant performance benefit from the use of 1GB huge pages, if they can succeed in obtaining them. The only way to reliably allocate regions of that size in current kernels is to use the [hugetlbfs](<https://docs.kernel.org/admin-guide/mm/hugetlbpage.html>) subsystem, which reserves memory for this purpose at boot time. Hugetlbfs is inflexible, though; its reserved pages cannot be used for other purposes, and it cannot reserve additional pages if the workload needs them. The system administrator can only hope that the reservation set up at boot time is suitable for all workloads the system may subsequently run. 

Van Riel's patch set attempts to make 1GB allocations more reliable without the need for the hugetlbfs reservation. It takes a number of approaches to reach that goal, but the core idea is the management of memory in units known as "super page blocks". The kernel breaks memory into page blocks now, and it manages them in ways designed to combat fragmentation. One of the key techniques is to segregate allocations that can be moved from those that cannot. For example, most user-space memory is movable, it is just a matter of changing all of the relevant page-table entries to match. On the other hand, allocations for the kernel's own use generally cannot be moved. By keeping the two types separate, the kernel maximizes its chances of creating entire free page blocks by moving the pages within them elsewhere. 

Page blocks work reasonably well when it comes to helping the page allocator provide PMD-level huge pages. But they are much smaller than the 1GB target that Van Riel is trying to hit; on the system used to write this article, page blocks are configured to be 4MB in size. In a typical system, some page blocks will be used for movable allocations, while others will hold unmovable allocations. The movable blocks can, at need, be emptied out to create more PMD-level huge pages. But it only takes a single unmovable page block to render its entire containing 1GB huge page unmovable and, as a result, permanently fragmented. 

The addition of super page blocks gives the page allocator another level of visibility into the use of memory. If a request comes in to allocate an unmovable page, the allocator will attempt to allocate from an unmovable page block as before. Should there be no such page blocks with free memory available, though, the allocator will need to pick a new page block to allocate from. Van Riel's patch set will cause the allocator to attempt to select that page block from a super page block that already holds other unmovable allocations. In other words, the segregation of movable and unmovable allocations is now done at the 1GB scale, increasing the chances that 1GB huge pages can be created and allocated at need. 

There are a number of other techniques that are employed to help this overall policy succeed. Consider, for example, a virtually mapped kernel-space allocation (as might be obtained with `kvmalloc()`) requiring several pages. This allocation is not movable, and thus should be put into an unmovable super page block. If there are no such blocks with that many contiguous pages available, the allocator would naturally pick a new super page block, thus "tainting" it and making it unmovable. If, however, the allocation can be satisfied from existing unmovable super page blocks by splitting it into smaller chunks, the allocator will do that. 

The patches in this series carry an Assisted-by tag indicating that the Claude Opus LLM was used in their creation. As noted above, though, van Riel is not a new developer needing LLM assistance to be able to put together a kernel patch. His [first mention in LWN](<https://lwn.net/1998/0716/#:~:text=Rik%20van%20Riel%27s>) was with regard to a discussion on memory fragmentation — in 1998; he [proposed](<https://web.archive.org/web/20000521171334/http://www.surriel.com/zone-alloc.html>) the addition of a zoned memory allocator, which subsequently came to be. He is the sort of developer who is likely to get the benefit of the doubt when it comes to choices of tools. 

This particular patch series has not been entirely well received, though, despite the fact that its goal — reliable allocation of 1GB huge pages — is widely supported. The most pointed criticism was surely [this lengthy message](<https://lwn.net/ml/all/aj9yrlB0TrlYCLlf@lucifer>) from Lorenzo Stoakes, who took issue with the organization of the series and the code within it. ""The series is completely unmergeable as it stands. Not even close"." Other parts of the series were described as ""code war crimes against __rmqueue_smallest()"" or ""something you expect to see on a 1990's PHP website, not in core mm code"". He concluded with: 

> OK with all that said - to be absolutely clear - I respect you a great deal, and I KNOW you're (much, much) better than this. 
> 
> And, to repeat, this idea is very exciting and I _want_ to see this land. 
> 
> But I feel you've rather let the LLM run amok and it's selling you (very, very) short, given just how smart and capable you are. 

Van Riel [answered](<https://lwn.net/ml/all/528e3a5fbc27c9dc7a098121c32b7679b4c9962a.camel@surriel.com>) that he never expected the code to be merged in its current form, and that he had really been hoping for feedback on the overall design before reimplementing it in a form that could be considered for merging. The massive dump of LLM-generated code has gotten in the way of that process, though. Developers want to look at the implementation of a design to understand how it works in practice, and that has proved difficult for them to do in this case. This work will have to be redone, with more human attention paid, before it can be seriously considered, even at the design level. 

#### Working-set tracking for virtual-machine guest memory

Virtual machines (VMs) are subject to two levels of memory management. They handle their own memory internally, but there is also host-level management that tries to ensure that the system's physical memory is used efficiently by all of the running VMs. The host-level task is easier if the VM manager (hypervisor) has visibility into which pages a given VM is actually using; that information can be used to reclaim the colder pages (or to move them to slower memory). But that information tends to be locked away inside the VM itself. 

[This patch series](<https://lwn.net/ml/all/20260629120749.566063-1-kirill@shutemov.name>) from Kiryl Shutsemau is meant to make that usage information more readily available to VM managers. It adds a new registration mode to the [`userfaultfd()`](<https://man7.org/linux/man-pages/man2/userfaultfd.2.html>) system call that will cause the protections (at the host level) on the indicated range of pages to be marked as "no access". At registration time, it is possible to choose between two alternatives for what happens when the virtual machine does access one of those pages. If the synchronous mode has been selected, the VM manager will receive a message from the kernel and, after having noted the fault, can resolve it with another request back to the kernel. In the asynchronous mode, the kernel resets the permissions without notification to the VM manager. At a later time, the manager can scan the page range to see which pages have been accessed. [This documentation patch](<https://lwn.net/ml/all/20260629120749.566063-16-kirill@shutemov.name>) describes the feature in more detail. 

These patches, too, include an Assisted-by tag naming Claude Opus. Shutsemau does not have Van Riel's longevity in the kernel community; having made [his first contribution](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e621e69137b2>) to the 2.6.25 kernel in 2008, he is a relative newcomer. Still, his track record is long enough to make it clear that, once again, he is entirely capable of understanding the work that he is submitting to the community. 

This series has been through seven revisions and significant changes resulting from the reviews that were received. In contrast to how Van Riel's work was received, none of those reviews was the role of the LLM in the creation of the work raised as a concern. In response to the second revision, though, Andrew Morton [did ask](<https://lwn.net/ml/all/20260508103220.aa46427b6f4c5d0247d2afb0@linux-foundation.org>) for information about how the LLM was used; Shutsemau [responded in detail](<https://lwn.net/ml/all/af5eALk9yO8pPcHv@thinkstation>). Much of the tool's role was in validating ideas: 

> For this particular project there was quite a bit of path-finding. I had a phase where I bounced ideas off Claude. It helped me understand the problem space better and formulate possible solutions. Rubber ducking on steroids. 
> 
> Once it's clear _what_ to do, we formulate a plan on _how_. It also involves back and forth. 
> 
> Once the plan was done, I gave the go-ahead on executing it. 

The most time-consuming part of the process, he said, was reviewing the resulting code; that involved restarting the process from the beginning more than once. It took ""maybe between 8 and 10"" rounds to obtain code that he was happy with — before then feeding the whole series, along with a different set of prompts, to an LLM for review. 

This workflow has seemingly produced a viable patch set that makes significant changes to the core memory-management code. The key to success, relative to the work previously described above, would appear to be the investment of substantial amounts of human time to get the tool to do the job properly, and in dealing with the remaining problems afterward. The care seemingly taken to ensure that the patches were up to the level of quality that the community expects by the time they hit the mailing lists appears to have made all the difference. 

Shutsemau did not say whether that whole process was more efficient than simply doing the job without LLM assistance, but he does seem inclined to use that process again in the future. 

The end of Shutsemau's message was a request for other developers to share their process for working with LLMs, but there were no responses. In truth, when it comes to significant work on the core kernel, he would appear to be one of the pioneers. The path toward success that he described, though, does not seem to be an easy one; anybody who is hoping to use an LLM as a quick way to become a kernel developer would do well to take note of what is likely to become the biggest success story (for kernel code) so far.
