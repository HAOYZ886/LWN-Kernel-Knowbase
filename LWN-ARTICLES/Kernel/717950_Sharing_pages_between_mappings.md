---
title: Sharing pages between mappings
url: https://lwn.net/Articles/717950/
date: "March 26, 2017"
category: "Filesystems-Union"
author: "By Jonathan Corbet March 26, 2017 LSFMM 2017"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
March 26, 2017

* * *

[LSFMM 2017](<https://lwn.net/Articles/lsfmm2017/>)

In the memory-management subsystem, the term "mapping" refers to the connection between pages in memory and their backing store — the file that represents them on disk. One of the fundamental assumptions in the kernel is that a given page in the page cache belongs to exactly one mapping. But, as Miklos Szeredi explained in a plenary session at the 2017 Linux Storage, Filesystem, and Memory-Management Summit, there are situations where it would be desirable to associate the same page with multiple mappings. Achieving this goal may not be easy, though. 

Szeredi is working with the [overlayfs filesystem](<https://lwn.net/Articles/403012/>), which works by stacking a virtual filesystem on top of another filesystem to provide a modified view of that lower filesystem. When pages from the real file in the lower filesystem are read, they show up in the page cache. When the upper filesystem is accessed, the virtual file at that level is a separate mapping, so the same pages show up a second time in the page cache. The same sort of problem can arise in a single copy-on-write (COW) filesystem like Btrfs; different files can share the same data on disk, but that data is duplicated in the page cache. At best, the result of this duplication is wasted memory. 

Kirill Shutemov noted that anonymous memory (program data that does not have a file behind it) has similar semantics; a page can appear in many [![\[Miklos Szeredi\]](https://static.lwn.net/images/conf/2017/lsfmm/MiklosSzeredi-sm.jpg)](<https://lwn.net/Articles/717952/>) different address spaces. For anonymous pages, the [`anon_vma`](<https://lwn.net/Articles/75198/>) mechanism allows the kernel to keep track of everything and provides proper COW semantics. Perhaps something similar could be done with file-backed pages. 

James Bottomley said that the important questions were how much it would cost to maintain these complicated mappings, and how coherence would be maintained. He pointed out that pages could be shared, go out of sharing for a while, then become shared again. Perhaps, he said, the [KSM mechanism](<https://lwn.net/Articles/330589/>) could be used to keep things in order. Szeredi said he hadn't really thought about all of those issues yet. 

On the question of cost, Josef Bacik said that his group had tried to implement this sort of feature and found it to be "insane". There are a huge number of places in the code that would need to be audited for correct behavior. There would be a lot of real-world benefits, he said, but he decided that it simply wasn't worth it. 

Matthew Wilcox suggested a scheme where there would be a master inode on each filesystem with other inodes sharing pages linked off of it. But Al Viro responded that this approach has its own challenges, since the inodes involved do not all have to be on the same filesystem. Given that, he asked, where would this master inode be? Bacik agreed, saying that he had limited his exploration to single-filesystem sharing; things get "even more bonkers" if multiple filesystems are involved. If this is going to be done at all, he said, it should be done on a single filesystem first. 

Bottomley said that the problems come from attempting to manage the sharing at the page level. If it were done at the inode level instead, things would be easier. Viro said that inodes can actually share data now, but it's an all-or-nothing deal; there is no way to share only a subset of pages. At that level, this functionality has worked for the last 15 years. But, since the entire file must be shared, Szeredi pointed out, the scheme falls down if the sharing must be broken at some point — if the file is written, for example. Viro suggested trying to steal all of the pages when that happens, but Szeredi said that memory mappings would still point to the shared pages. 

Bottomley then suggested stepping back and considering the use cases for this feature. Users with lots of containers, he said, want to transparently share a lot of the same files between those containers; this sort of feature would be useful in such settings. Bacik added that doing this sharing at the inode level would lose a lot of flexibility, but it might be enough for the container case which, he said, might be the most important case. Jan Kara suggested simply breaking the sharing when a file is opened for write, or even requiring that users explicitly request sharing, but Bottomley responded that container users would not do that. 

The conclusion from the discussion is that per-inode sharing of pages between mapping is probably possible if somebody were sufficiently motivated to try to implement it. Per-page sharing, instead, was widely agreed to be insane.
