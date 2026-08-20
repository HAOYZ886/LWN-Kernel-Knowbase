---
title: A new swap abstraction layer for the kernel
url: https://lwn.net/Articles/974587/
date: "May 23, 2024"
category: "Memory management-Swapping"
author: "By Jonathan Corbet May 23, 2024 LSFMM+BPF"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
May 23, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

Swapping may be a memory-management technique at its core, but its implementation also involves the kernel's filesystem and storage layers. So it is not surprising that a session on the kernel's swap abstraction layer, led by Chris Li at the [2024 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), was held jointly by all three of those tracks. Li has some ambitious ideas for an improved subsystem, but getting to a workable implementation may not be easy. 

Li started by looking at the current swap state maintained by the kernel to get a sense for what needs to be kept in a new implementation. The key datum is the swap offset — the location in the swap file where any further information about a swapped-out page can be found. Any other information is optional within the kernel. This scattering of information is flexible, but can also be a source of pain, he said. 

[![\[Chris Li\]](https://static.lwn.net/images/conf/2024/lsfmm/ChrisLi-sm.png)](<https://lwn.net/Articles/974591/>) The current swap design is memory-efficient, but complex. It could be improved at the cost of using more memory — getting worse in order to get better. David Hildenbrand said that all of the resources needed by the swap layer are preallocated, since trying to allocate memory when the system needs to swap is failure-prone. That preallocation is why minimizing the overhead is so important; if a way could be found to do less preallocation, overhead would be less of a concern. It would be nice to consume less memory when swap is not being used, but it is not good to have to allocate memory when swapping is necessary. 

Li agreed that systems often do not swap; any preallocated memory is simply wasted in that case. On the other hand, high memory consumption by the swap layer also hurts when a lot of swapping is happening. 

He proposed — initially — to add one byte to each [swap entry](<https://elixir.bootlin.com/linux/latest/source/include/linux/mm_types.h#L265>); that would be used to hold some flags. The full swap map (used to track the usage of space in the swap device) would not be preallocated, but would be grown as needed. The problem with adding a single byte, though, is that it would turn a four-byte entry into five bytes, which will create alignment problems. So, instead, the entry should grow by four bytes, which would allow the addition of pointers. But, then, if eight bytes are added, more things become possible, including dynamic allocation of the swap-entry structure. Its size could vary, as has been [proposed](<https://lwn.net/Articles/973565/>) for memory descriptors. Compound swap entries could share this descriptor, which would, in the end, more than pay back the cost of those extra eight bytes. 

Support for directly swapping multi-size transparent huge pages (mTHPs) has been added to the mm-unstable tree, he said. Swapping 64KB mTHPs to [zram](<https://docs.kernel.org/admin-guide/blockdev/zram.html>) devices significantly improves the compression ratio and saves nearly two-thirds of the CPU time needed when swapping single pages. But, as usual, there is a cost, in the form of increasing fragmentation in the back end. As time passes, the ability to allocate mTHP-sized chunks degrades, to the point that it becomes unusable after five hours, even with less than half of swap space in use. 

The problem lies in how swap clusters are handled, he said. The cluster size is set equal to the full THP size (typically 2MB). Any single-page allocation will be taken from the first cluster on the per-CPU list, leading to a partially empty cluster that cannot be used to swap even mTHP-sized chunks, which are smaller than the full THP size; he is not sure why. Clearly there is a need for a better allocator. In the short term, his plan is to make note of the half-empty swap clusters and allocate mTHP-size chunks from there. The longer-term plan is to create a buddy allocator for swap entries. 

But, he said, a better allocator is not enough. Since the swap layer does not control the lifecycle of swap entries, fragmentation can still happen. A malicious user could selectively free memory, leading to a situation where a lot of swap space is available, but none of it can be allocated. The solution to this problem is non-contiguous swap entries, managed by way of a compound swap structure. The head entry would contain the order of the structure, which would suffice for the simple case. The more complex case would be handled by dropping the alignment requirement for swap space, and allowing it to not be contiguous. 

Li noted that this would be an invasive change. Matthew Wilcox agreed, warning Li that he was setting himself up for "a world of pain". This plan is, Wilcox said, a reinvention of the filesystem, and the tragic results of a memory-management developer trying to design filesystems are well known. He suggested that Li find a filesystem developer to work with if is truly necessary to follow this path. 

Jan Kara said that existing filesystem designs are not suited to this task, since they are not written with the goal of minimizing memory overhead. But, he said, managing that kind of complexity will have its cost. He suggested that an easier solution might be to set a minimum size for swapped-out data as a way of reducing fragmentation. Large ranges of anonymous memory tend not to be used, he said, so it should be possible to swap it out in bigger chunks, reducing both overhead and fragmentation. 

At the end of the session, Hildenbrand said that this plan was introducing too much complexity. Instead, he said, the swap-in and swap-out granularity should be decoupled from each other. If swap-space fragmentation is an issue, folios should just be split prior to swapping out. Folios could be reassembled at swap-in time. Li answered that his current design allows for partial inward swapping; it is not necessary to bring in an entire folio. 

The next step, as always, will be to wait for patches to show up implementing some of these ideas.
