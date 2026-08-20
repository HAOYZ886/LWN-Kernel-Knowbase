---
title: Toward the unification of hugetlbfs
url: https://lwn.net/Articles/974491/
date: "May 22, 2024"
category: "Huge pages; Memory management-Huge pages; hugetlbfs"
author: "By Jonathan Corbet May 22, 2024 LSFMM+BPF"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
May 22, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

The kernel's [hugetlbfs subsystem](<https://docs.kernel.org/admin-guide/mm/hugetlbpage.html>) was the first mechanism by which the kernel made huge pages available to user space; it was added to the 2.5.46 development kernel in 2002\. While hugetlbfs remains useful, it is also viewed as a sort of second memory-management subsystem that would be best unified with the rest of the kernel. At the [2024 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), Peter Xu raised the question of what that unification would involve and what the first steps might be. 

In theory, the kernel's transparent huge page mechanism makes hugetlbfs unnecessary. There are, though, reasons for the longevity of hugetlbfs. It allows huge pages to be reserved, so that they will remain available even if system memory as a whole is fragmented. It also implements page-table sharing across multiple processes, which is not otherwise available in Linux (a later LSFMM+BPF session talked about [`mshare()`](<https://lwn.net/Articles/895217/>), which is meant to fill that gap). And, of course, software has been written using the hugetlbfs ABI, so it must continue to be supported. 

#### Consolidation

Xu began by saying that his objective was not to remove hugetlbfs, but to consolidate it into the rest of the memory-management subsystem. There are 11 different code paths that are specific to hugetlbfs; he thinks that can be reduced to two or three. Making hugetlbfs into an ordinary filesystem is not a goal; doing so would likely increase complexity for little benefit. 

[![\[Peter Xu\]](https://static.lwn.net/images/conf/2024/lsfmm/PeterXu-sm.png)](<https://lwn.net/Articles/974509/>) Hugetlbfs, thus, will remain a "special", RAM-based filesystem. It is, he said, ancient stuff, much of which is aimed at use cases that may not even exist anymore. But developers are afraid to touch it. Hugetlbfs is a maintenance nightmare, inflicting its special code paths on the rest of the kernel; users have requested new features, but they have been rejected out of fear of increasing the complexity of the system. So, he said, there is no time like the present to deal with this problem. Fortunately, the large-folio work is making it easier to coalesce at least some of the hugetlbfs code into the rest of the kernel. 

Xu wondered whether this work should be done by creating a new, better version of hugetlbfs, or by working to unify the existing code. His feeling, though, is that a new version would not be justified; there is no need for any sort of ABI break, which would be the biggest reason to start over. Unifying hugetlbfs means working with an ugly ABI implemented by ugly code, but starting over would bring an entirely different kind of pain. 

David Hildenbrand agreed that the hugetlbfs ABI is ugly; for him, though, the biggest problem is all of the "`if (hugetlbfs)`" calls sprinkled through the rest of the memory-management subsystem. Many of these tests are driven by alignment requirements. Creating a new version of hugetlbfs would be too much, he said, but there would be value in being able to set a flag to remove some of the hugetlbfs restrictions; that would make it possible to, for example, free half of a hugetlb folio. Xu agreed with that view. 

Hildenbrand mentioned [high-granularity mapping](<https://lwn.net/Articles/931773/>) as a proposed hugetlbfs enhancement that ended up being rejected out of fear of adding more hugetlbfs-related complexity to the memory-management subsystem. Rather than add special-case exceptions like that, he said, it would be better to just drop the hugetlbfs restrictions everywhere. Michal Hocko, though, asked the group to take a step back and summarize the features that are actually needed. Hugetlbfs came about in a time when transparent huge pages didn't exist; perhaps it would be better to make more use of transparent huge pages than to add more hugetlbfs features. 

Xu answered that the use of transparent huge pages has its own performance impact; the realtime configuration disables it, for example. There are also use cases that insist on 1GB huge pages, and hugetlbfs is the only way get them in current kernels. He would, he said, be happy to see a proposal based on transparent huge pages that addresses those concerns. 

#### The 1GB page reservation

John Hubbard said that there are a lot of artificial-intelligence applications out there that can benefit from huge pages; some of those applications need huge pages badly, and so they use hugetlbfs. Others can just take advantage of the kernel's improving transparent huge page support and get faster with no additional effort. There are, he suspects, some applications out there that have been well tuned and benefit from not having to wait for the kernel to collapse their memory into transparent huge pages. Some applications will always need huge pages that are always available. 

A remote participant said that hugetlbfs is often most useful to allocate memory for virtual machines; this use case really wants the 1GB guarantee that hugetlbfs can provide. In this case, the 1GB aspect is the only thing that matters. Another remote attendee said that the high-granularity-mapping code was an attempt to add transparent huge page features to hugetlbfs, but that it would be better to support 1GB huge pages in the core memory-management subsystem than to add more hugetlbfs features. 

Jason Gunthorpe said that he would really like to see the hugetlbfs code taken out of the core; after that, he doesn't care about any "craziness" hidden within it. Matthew Wilcox said that the biggest problem is the hugetlbfs page-table walker, which has a lot of special cases and needs to be gotten rid of, somehow. 

Xu tried to reach a sort of conclusion by saying that there is still sense in having a separate allocator that can provide the guarantees that some applications need. But, he said, if he cannot implement high-granularity mappings on top of that allocator, he will lose a lot of his motivation to do this work. Hildenbrand said that, if this work is done right, high-granularity mappings should just come naturally. 

Xu continued, saying that anybody who wants partial mappings in hugetlbfs should go ahead and post a patch; it will be interesting to see how that works with the 1GB-page allocator. There is still a need for a better interface to consume hugetlbfs pages, though. Gunthorpe said that [memfd](<https://lwn.net/Articles/593918/>) is that interface; it just needs to be taught how to reach into hugetlbfs, which could provide a single reservation for all users needing 1GB pages. Hildenbrand said that plans for [`guest_memfd()`](<https://lwn.net/Articles/949277/>) need a number of the proposed features, including partial mappings and high-granularity mapping. Gunthorpe added that there is merit in separating the various hugetlbfs components; the 1GB page pool is generally useful and should be a separate feature. In general, users want the reservation feature, but would rather do without a "screwy ABI". Accessing the reservation with an `mmap()` flag would be nice, he said. 

Dan Williams read a suggestion from the online chat: hugetlbfs should be removed and reimplemented as an [`fallocate()`](<https://man7.org/linux/man-pages/man2/fallocate.2.html>) option on the tmpfs filesystem. Xu said that, in that case, the challenge would be getting users to move over; a deprecation process would be needed. Another participant said that adding hugetlbfs features to tmpfs would require unifying the page-table walker. 

Gunthorpe said that, once features become available in the core memory-management subsystem, everything else just falls into place. A new ABI could then be simply implemented as a memfd `ioctl()` call providing access to the 1GB-page reservation. Hocko, though, said that pushing users away from hugetlbfs would take 15 to 20 years; it is better to just leave it in place, clean up its internals, and make them usable elsewhere. 

For 1GB pages, Xu said, the mechanism is already in place; all that is needed is to expose a better ABI for it. Hildenbrand suggested, again, simply dropping the restrictions on hugetlbfs pages, allowing 1GB huge pages to be mapped as needed. Xu continued that existing users do not see the hugetlbfs ABI as ugly; they are happily using it. The memory-management developers, instead, are not happy with it; is that a sufficient reason to introduce a new ABI? 

As this (two-slot) session ran out of time, Hildenbrand mentioned the strange semantics that hugetlbfs imposes on `MAP_PRIVATE` mappings. Among other things, that makes it impossible to insert a uprobe or a breakpoint in a hugetlbfs 1GB page. He said that it was clear that Xu would have to clean up the page-table walker, but that the kernel would have to continue to provide hugetlbfs as it is, since there are users out there. 

#### The next steps

The discussion was not done, though; another slot was scheduled later in the day. Xu got more deeply into the details, saying that, in his first attempt, he was trying to clean up the `get_user_pages()` code path (which is the way that the kernel maps user-space pages). After some work, that project was mostly successful; [patches](<https://lwn.net/ml/linux-kernel/20240321220802.679544-1-peterx@redhat.com/>) have been posted and since merged for the 6.10 release. 

There are numerous challenges remaining, though. One of those is the "hugepd" mechanism used by the PowerPC architecture to handle huge pages. Hugepd is imposed by that architecture's special page-table requirements, but it can evidently be gotten out of the way for huge pages, simplifying the unification of the code. Christophe Leroy has posted [a patch set](<https://lwn.net/ml/linux-kernel/cover.1715971869.git.christophe.leroy@csgroup.eu/>) doing that work; Xu would like some help reviewing it. 

Huge pages can be represented in three ways in the kernel, he said. They can be a huge mapping as defined by the architecture (a PMD-level mapping, for example), the "cont-pte" format (where the huge page is mapped as base pages, but with a special flag set to tell the CPU that a group of physically contiguous pages exists — see [this article](<https://lwn.net/Articles/937239/>)), and the PowerPC hugepd format. The page-table-walker ABI supports only the first two of them. Unification requires adding generic support for hugepd, or just removing it; the latter approach is the direction taken by Leroy's patch set, but it needs to be extended to remove hugepd completely. 

A generic page-table walker that handles all cases would be an elegant solution, he said, if it could be achieved. Wilcox said that work needs to be done to make page-table walkers easier to write, starting with figuring out what all the needs are. Gunthorpe agreed, noting that the kernel is full of duplicated page-table-walking code. It would be good to abstract out the details to create a generic ABI; Wilcox said he was tempted to just try it. 

Xu asked the group if there was a need to support P4D huge pages; these are mapped one page-table-level higher than 1GB pages, and are 512GB in size. Wilcox said that 512GB pages would be ridiculous, with no practical use; the consensus in the room was that there was no need to support that size anytime soon. 

As time (once again) ran low, Xu said that it may never be possible to unify all of the hugetlbfs paths in the kernel; he may have to just give up on some of them. Page-fault handling and PMD-level page-table sharing may be cases in point. There are some hugetlbfs quirks to work around. For example, a read on a `MAP_PRIVATE` page does not result in a page-cache entry; instead, it creates a read-only anonymous page. It makes no sense to port features like this to generic code, he said. 

Wilcox agreed that there was no problem with not unifying quirks like that; they don't affect other users of the system. The PMD-sharing problem is better solved with `mshare()`. Perhaps the page-table sharing supported by hugetlbfs could eventually be dropped, he said. Xu concluded by listing a set of paths that he intends to address in the near future. These included page-table walking, handling [`userfaultfd()`](<https://man7.org/linux/man-pages/man2/userfaultfd.2.html>) faults, [`mprotect()`](<https://man7.org/linux/man-pages/man2/mprotect.2.html>), [`mremap()`](<https://man7.org/linux/man-pages/man2/mremap.2.html>), [`fork()`](<https://man7.org/linux/man-pages/man2/fork.2.html>), and more. Some of those, he noted, would be difficult. The session ended with Wilcox expressing his thanks to Xu for addressing this "long overdue" problem.
