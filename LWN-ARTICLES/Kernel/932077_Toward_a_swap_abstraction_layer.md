---
title: Toward a swap abstraction layer
url: https://lwn.net/Articles/932077/
date: "May 22, 2023"
category: "Memory management-Swapping"
author: "By Jonathan Corbet May 22, 2023 LSFMM+BPF"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
May 22, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

The kernel's swapping code tends to not get much love. Users try to avoid it, and developers often find better things to do with their time than trying to improve it. At the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>), though, Yosry Ahmed dedicated a memory-management-track session to the problem of the swap layer and what might be done to make it better. 

Ahmed started with the subject of [zswap](<https://www.kernel.org/doc/html/latest/admin-guide/mm/zswap.html>), which "swaps" pages by compressing them and storing them more efficiently in RAM. This action happens just before actual swapping would occur, and the rest of the memory-management system doesn't know anything about it. One of the results of that is that a system using zswap must provide a swap file, even if no pages are ever stored there. Every page kept in zswap must also have a slot reserved in the swap file, which wastes space in that file and takes extra CPU time to manage. The reclaim code is unaware of zswap and has no idea where the pages it evicts will go. 

So zswap is not perfectly integrated with the rest of the system, but it is still heavily used, including on Google's production systems. He suspects that Chrome OS may be using it, and that a number of other companies running large fleets also have a place for it. So it would be good if zswap were to work more efficiently. Google has hacked together a short-term fix in the form of a simple indirection layer that indexes both the swap file and the zswap area. The rest of the memory-management subsystem just sees a single swap index and doesn't know whether it corresponds to an actual swap entry or a zswap entry. He is "not fond of it", but it works. 

A better solution for the medium-term, he said, could be a proper indirection layer. There would be an operations vector (a structure full of function pointers) for each swap implementation, and an [XArray](<https://www.kernel.org/doc/html/latest/core-api/xarray.html>) to index the whole thing. As well as addressing the duplication problem, this implementation would make it possible to optimize the [`swapoff()`](<https://man7.org/linux/man-pages/man2/swapon.2.html>) operation. 

An even better, long-term idea, is something he described as "swap descriptors" that would track swapping information independently from the backend used to store any given swapped page. This would be a much cleaner abstraction, he said, though it would add a bit of memory overhead for the additional tracking. There would also have to be another index added to support cluster readahead (which reads pages in larger chunks for performance) properly. This would be a bigger job overall. 

Matthew Wilcox said that cluster readahead is an optimization that was necessary for spinning media, but that it has no value for solid-state storage. Rather than worrying about cluster readahead, the memory-management subsystem should just be swapping entire folios and keeping the data together that way. Michal Hocko agreed, saying that users with slow disks have learned to avoid swap at all costs; cluster readahead is an optimization for a non-existent set of users. 

Ahmed said that he was worried that he would be told that keeping cluster readahead would be a requirement, but nobody in the room seemed to feel that way. His next question was whether a memory overhead of 0.6-0.8% for swap descriptors would be too much; Wilcox responded that it was affordable, but said that care had to be taken to avoid allocating memory in the reclaim path. 

Chris Li briefly presented an alternative, which he called a "virtual filesystem interface for swap files", that would abstract out all of the operations for each swap type. Each swap file would have its own swap count and slot management. Swapping, when implemented this way, could be organized in layers without the need for an additional indirection layer; that would avoid the need to increase its memory usage. The Android system, he said, aims to keep about 20% of its pages in swap; its developers might not be pleased with the extra overhead. 

David Hildenbrand, at the end of the session, said that people have been complaining about the swap layer for years. He would like to see that code rewritten to work more like the virtual filesystem layer does, so he has a strong preference for Li's alternative if it takes the kernel in that direction. Dan Williams advised Li that "if you come to LSFMM and show people a pony, they will want the pony". Hildenbrand agreed and, with a laugh, said that Li's proposal was accepted. 

Swapping in Linux may thus finally be on a path toward improvement. It's worth keeping in mind, though, that the code implementing this idea does not exist yet. So the kernel will probably not be swapping out its swap interface in the near future.
