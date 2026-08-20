---
title: "Willy's memory-management to-do list"
url: https://lwn.net/Articles/753078/
date: "April 30, 2018"
category: "Memory management-struct page"
author: "By Jonathan Corbet April 30, 2018 LSFMM"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
April 30, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

Matthew "Willy" Wilcox has been doing a fair amount of work in the memory-management area recently. He showed up at the 2018 Linux Storage, Filesystem, and Memory-Management Summit with a list of discussion topics related to that work; it was enough to fill a plenary session with some spillover into the memory-management track the next day. Some of his topics were fairly straightforward; others look to be somewhat more involved. 

He started the plenary session by noting that the "`vm_fault_t` controversy" turned out to be rather more involved than he had expected; he seemed to be referring to a disconnected series of patches ([example](<https://lwn.net/ml/linux-kernel/20180421171442.GA17919@jordon-HP-15-Notebook-PC/>)) creating a new `vm_fault_t` typedef for page-fault handlers. He has been busy trying to run the resulting changes through the relevant maintainers, but it has been some work; he didn't realize, he said, that the filesystem developers would be so "[belligerent](<https://lwn.net/Articles/753081/>)" about wanting to see the full series — which doesn't exist yet. In any case, he said, this is a boring topic; the room seemed to agree, so he moved on. 

He then put up an example of code performing a memory allocation, and pointed out that it contained several bugs, including a missing overflow check and a lack of type checking. Bugs like this are fairly common in the kernel. He proposes to handle that use case with a new helper called `kvmalloc_struct()`, and is looking for feedback. The room didn't seem to find this topic to be worth arguing about either; Ted Ts'o finally suggested that Wilcox should "paint it blue". 

He then called for the addition of `malloc()` and `free()` to the kernel API. A call to `malloc()` would turn into a [![\[Matthew Wilcox\]](https://static.lwn.net/images/conf/2018/lsfmm/MatthewWilcox-sm.jpg)](<https://lwn.net/Articles/753086/>) `kvmalloc()` call with the `GFP_KERNEL` flags. His purpose is to make it easier for new developers to write drivers by providing something that is more similar to the normal C API. There did not seem to be a lot of support for this idea from the group, though. 

If an application uses `mmap()` to map the same page four-billion times, the map count in the kernel will overflow, with all of the [undesirable effects](<https://lwn.net/Articles/693038/>) that come from a reference-count overflow. Getting to this point is not easy; one needs a machine with 30GB of RAM to be able to do it. He has posted a fix for the problem; it simply kills any process that has tried to map the same page more than 5,000 times. Andrew Morton suggested that the alternative is to just leak the page. 

There are two ways to get huge pages in user space (hugetlbfs and transparent huge pages), and they use the page cache differently; Wilcox would like to unify them. Hugetlbfs indexes pages in multiples of 2MB, while transparent huge pages use a normal 4KB offset. He would like to make hugetlbfs less special by using 4KB offsets there too. The only problem is a big performance hit, since there are many more entries in the radix tree; that makes this approach unworkable. So a solution he intends to pursue instead is to change the transparent huge pages implementation to use the multi-size features of his [XArray](<https://lwn.net/Articles/745073/>) mechanism, making it more closely match hugetlbfs. 

Then, he would like to enhance the page cache to allow the use of block sizes that are bigger than the system page size. He thinks it can be done without requiring higher-order allocations, which has been a sticking point in the past. In short, the memory-management subsystem would inform the filesystem when a page fault has occurred and ask the filesystem to take care of populating the page cache with the needed pages. The filesystem can do that with normal 4KB pages; better performance will be had if it attempts a larger allocation first. 

Dave Chinner pointed out that there were working patches for larger block sizes in 2007; they used compound pages, and were not accepted due to the high-order allocation issues. We have been here before, he said, and know how it works. Have high-order allocations been fixed in the meantime? Wilcox answered that the difference this time around is the fallback path that is implemented within the filesystems. Chinner worried that this idea didn't sound reliable; in particular, there could be problems (as usual) with `truncate()`. Wilcox answered that much of the work could be done once in the virtual filesystem layer and, hopefully, made to work reliably. 

He also briefly mentioned the idea of teaching the page cache about holes (ranges with no blocks allocated) in files. Currently those are represented by zero-filled pages in the cache if need be. Replacing those with "zero entries" could save a significant amount of memory; an actual page would only need to be allocated in the event of a write operation. 

There was also a brief discussion of "PFN entries" in the page cache. Currently, page-cache entries include a pointer to the `page` structure representing the page in memory. That structure will point to a specific mapping (such as the file containing the page). If you want to share pages in memory that are shared on disk (in a couple reflinked files, for example), the same page will have different mappings depending on where it came from. In that case, putting that pointer in the cache is going to lead to trouble, so he proposes using the page-frame number instead. There would still be `page` structures backing the whole thing up, but there would be an extra level of indirection to access them. 

Finally, he said that he would like to get rid of the `GFP_NOFS` allocation flag, which tells the system that it cannot call into filesystem code to free memory. Instead, "scoped allocation", which simply tracks when filesystem code is holding a lock (and thus cannot be called back into) should be used. XFS is the closest to having implemented scoped allocation, he said, but there are still places where `GFP_NOFS` is used. This work is not currently making good progress. 

Ts'o said that some good documentation would help; he has been trying to push this work forward, but has run into some questions. Chinner warned that this work has to be done carefully; `GFP_NOFS` is often used to silence the lockdep checker rather than out of a real need to avoid filesystem calls. He suggested adding a `GFP_NOLOCKDEP` flag for that purpose. Meanwhile, these call sites are hard to identify, since they are almost never documented as such. 

The plenary session came to an end at this point, but Wilcox had not yet run out of ideas to run by the development community. 

#### Cleaning up struct page

The [`page` structure](<https://lwn.net/Articles/565097/>) is one of the most complicated in the kernel; the curious are encouraged to have a look at [its definition](<https://elixir.bootlin.com/linux/latest/source/include/linux/mm_types.h#L29>). Each `page` structure tracks one page of physical memory; it is used differently depending on how the page itself is used. As a result of varying needs and of the need to keep the structure small [![\[Matthew Wilcox\]](https://static.lwn.net/images/conf/2018/lsfmm/MatthewWilcoxPage-sm.jpg)](<https://lwn.net/Articles/753087/>) (current systems have millions of them), `struct page` has become a difficult-to-follow mixture of structures, unions, and `#ifdefs`. Few developers dare to try to make changes there. 

Wilcox posted [a set of diagrams](<https://docs.google.com/spreadsheets/d/1tvCszs_7FXrjei9_mtFiKV6nW1FLnYyvPvW-qNZhdog/edit#gid=1377749841>) showing how the various fields of `struct page` are used now. When a kernel subsystem allocates a page, he said, it also gets access to the `page` structure to keep track of it. But that access is not exclusive. The `refcount` field can go up and down at any time, even if the allocating subsystem thinks it has exclusive access to the page. If the page is mapped into user space, the `mapping` field will be use for reverse-mapping. Various flags have special meanings, and so on. 

In the end, a lot of users simply don't bother trying to store information in `struct page` even though there is space available there; it simply looks too complicated, and it is not at all clear which fields are safe to modify. That is, he said, "a shame". 

To make kernel development less shameful, he is proposing a reorganization of the `page` structure to make it more comprehensible. The fields that are safe to touch have been moved together, resulting in five contiguous words of available memory. The complex arrangement of structs and unions has been replaced with a single set of unions, each containing a set of structs or simple types. 

There was some discussion about the details of specific fields, and it was established that drivers could safely use the `mem_cgroup` field. In general, though, everybody seemed to feel that the proposal was a major improvement that made `struct page` much easier to understand. Wilcox promised that a patch set making these changes would be forthcoming soon.
