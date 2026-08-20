---
title: The search for available page flags
url: https://lwn.net/Articles/787338/
date: "May 4, 2019"
category: "Memory management-struct page"
author: "By Jonathan Corbet May 4, 2019 LSFMM"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
May 4, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

Among the many other things crammed into the `page` structure that is used to represent a page of memory in the kernel is a set of flags to track the state of the page. These flags have been in short supply for some time; LWN [looked at the problem](<https://lwn.net/Articles/335768/>) nearly ten years ago. Jérôme Glisse ran a session during the memory-management track of the 2019 Linux Storage, Filesystem, and Memory-Management Summit to explore ways of making some flags available for new uses. While there may be some easily available bits in the field that holds the page flags, obtaining a significant number of them may be tricky. 

Glisse is looking for a way to add a new, generic page-protection bit in order to implement a feature like kernel same-page merging (KSM) for file-backed pages. There is a flag for KSM now, but it's already overloaded with the `PageMovable` flag, which is relevant for file-backed pages. So, he asked, where might he be able to locate another flag that could be used for this purpose? 

His attention was drawn to the general area of memory reclaim, which uses many of the available flags. The `PageActive`, `PageIsolated`, `PageLRU`, `PageMovable`, `PageReclaim`, `PageReferenced`, `PageUnevictable`, and `PageWorkingset` flags are all tied to reclaim in one way or [![Jérôme Glisse](https://static.lwn.net/images/conf/2019/lsfmm/JérômeGlisse-sm.jpg)](<https://lwn.net/Articles/787387/>) another. Often, when that many flags are associated with an activity, one can discern patterns of flags that are either always set together or which never appear together; that can allow the replacement of flags with a more efficient representation. In this case, though, he found that almost all combinations of those flags are possible and valid, so there is no real opportunity to merge any of them. To further complicate matters, there is no single lock that protects access to all of those flags, so combining them would require difficult (or impossible) locking changes. 

Having failed to reclaim any of the reclaim-oriented flags, Glisse turned his attention to the node ID and zone number, both of which are stored in the same word as the page flags. The node ID is frequently used in hot paths, so trying to move it out of the `page` structure is likely to create performance issues. The zone number is not quite as hot and might be a candidate to be relocated. Indeed, in some configurations this information already gets pushed out of the `page` structure, so there might be some promise here. 

Some low-hanging fruit might also be found in the width of the node ID field; it is sized to hold the maximum number of NUMA nodes that the kernel is configured to support — typically 1024. But there are not all that many 1024-node systems out there; that sizing was described as a holdover from the days of SGI's huge systems. So it may be possible to gain a bit or two there, though probably not more than that. It is also possible to calculate the node ID from the zone number, eliminating that field altogether, but that would add overhead to some of the hottest page-allocator paths, which would be unwelcome. 

There was some talk of getting rid of either `PageIsolated` or `PageMovable`, both of which are optimizations for information that can be had elsewhere. `PageIsolated` is there to keep two threads running compaction from interfering with each other, a situation that, according to Hugh Dickins, cannot happen anyway. 

Looking at some of the other bits, Matthew Wilcox observed that the `PageError`, `PageSlab`, and `PageHWPoison` flags are incompatible with each other and could perhaps be unified in some way. Indeed, it seems that there is no need for a flag for `PageHWPoison` (which marks a page that has been taken out of service due to hardware errors) at all. To the extent that the kernel needs to reference that state, it does not need to happen quickly, but most of the time such a page should be out of the kernel's view entirely. 

Dave Hansen suggested creating a concept of "fast" and "slow" page flags, where the slow ones are stored outside of the `page` structure. A single fast bit could be set when any of the slow ones are, hopefully eliminating the need to actually check the slow bits most of the time. Mel Gorman, instead, suggested that some kernel features could be made dependent on whether sufficient page flags are available; if the kernel is configured to support a large number of NUMA nodes, perhaps it would be unable to support KSM. But, as Hansen reminded the group, there are not many bits to be had via that path. 

Another option is to push more data (flags or something else) out to the [`page_ext` structure](<https://elixir.bootlin.com/linux/v5.0.11/source/include/linux/page_ext.h#L28>), which already exists to hold information that won't fit into `struct page`. Hansen worried, though, that this would make it too easy to bloat the amount of per-page data stored, which already occupies a significant fraction of the system's memory. Without a hard constraint, that data could easily get out of hand. 

The final suggestion aired in this session was to create an [XArray](<https://lwn.net/Articles/745073/>) to hold pages in relatively rare states (such as `PageIsolated`). They could be stored using their page-frame number and searched for when the need arises. Whether any of these suggestions will be implemented remains to be seen, though.
