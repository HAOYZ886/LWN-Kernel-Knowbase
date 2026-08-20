---
title: "Hole-punching races against page-cache filling"
url: https://lwn.net/Articles/864363/
date: "July 29, 2021"
category: "Filesystems-Virtual filesystem layer"
author: "By Jonathan Corbet July 29, 2021"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
July 29, 2021

Filesystem developers tend to disagree with each other about many things, but they are nearly unanimous in their dislike for the [`truncate()`](<https://man7.org/linux/man-pages/man2/truncate.2.html>) system call, which chops data off the end of a file. Implementing `truncate()` tends to be full of traps for the unwary — the kind of traps that can lead to lost data. But it turns out that a similar operation, called "hole punching", may be worse. This operation has been subject to difficult-to-hit but real race conditions in many filesystems for years; [this patch set from Jan Kara](<https://lwn.net/ml/linux-mm/20210715133202.5975-1-jack@suse.cz/>) may finally be at a point where it can fill the hole in hole punching. 

Hole punching, as its name suggests, is the act of creating a hole in the middle of a file; it is performed using the `FALLOC_FL_PUNCH_HOLE` option to the [`fallocate()`](<https://man7.org/linux/man-pages/man2/fallocate.2.html>) system call. The caller provides an offset and a length; the kernel then erases the given number of bytes in the file, starting at the provided offset. The associated blocks on the underlying storage device are freed for other uses. The length of the file does not change, though; this operation creates a hole that, if read, will return zeroes. It is, essentially, an efficient way of writing zeroes to the specified range within the file. 

Note that neither the offset nor the length must be page-aligned. The kernel will write zeroes to the partial pages at the beginning and end of the hole, should they exist; this edge work is essentially just a couple of `write()` calls. The efficiency gains of hole punching, though, come from its ability to simply drop entire pages from the file without writing anything; that, naturally, is also where the challenges lie. 

To implement (the full-page part of) hole punching, a filesystem must do (at least) two things: remove the associated pages from the page cache, and free the blocks on the storage device. A failure to do either could leave the old data visible in the file, which is something that user space has just made an explicit request to prevent. But, even if both tasks are properly carried out, there is another way in which things can go wrong. The problem is that, in current kernels, there is nothing that ties those two operations together into an atomic change, meaning that something else can happen between one and the other. 

Specifically, a filesystem can clear the relevant pages out of the page cache in the usual way, but then race against another task that is trying to access the same file. Should that other task access one or more of the hole-punched pages in the file, they can be reinstated in the page cache before the filesystem has done the work of cleaning up the blocks on disk, leaving stale information in the page cache that may get written back out at some future time. That could lead to any of a number of unpleasant things, including the old data persisting, exposure of unrelated data, or corruption of the filesystem. That can punch a hole in the user's trust in the system overall. 

This race is clearly difficult to hit, or there would have been a stream of corruption reports since hole punching was added to 2.6.38 just over ten years ago. But it is a real race that will surely bite somebody sooner or later; it needs to be fixed. Doing that properly has required ten versions of Kara's patch set (at last count) since early 2021. 

The solution is conceptually simple: filesystems must take a lock that prevents hole punching and the instantiation of page-cache entries from happening at the same time. But the words "simple" and "locking" are rarely found together in the filesystem realm. In this case, the locks normally used to serialize operations on page-cache pages cannot be used, since the point is that the pages should be absent. Other existing locks run into locking-order issues. So Kara had to add a new lock (a reader-writer semaphore, specifically) to the [`address_space` structure](<https://elixir.bootlin.com/linux/v5.13.5/source/include/linux/fs.h#L435>) that describes a mapping between the page cache and a file. This lock, called `invalidate_lock`, prevents operations that instantiate page-cache pages (readers, in the sense of this lock) from racing with those that invalidate pages and underlying storage (writers). 

The way filesystems use this lock varies a bit, depending on their internal architecture, but the end result is the same: the race is closed in almost all of the filesystems that support hole punching in the first place. There are a couple of exceptions, specifically the GFS and OCFS2 cluster filesystems, where everything is more complex and the maintainers need to be involved; fixes for those filesystems are still under development. 

This work was deemed ready to go and was pushed to Linus Torvalds for the 5.14 merge window, but Torvalds [was not impressed](<https://lwn.net/ml/linux-fsdevel/CAHk-=whuUxfoYj=dRnzRybg_sOdFPMDx_t06Lz936Pgnh6QCTQ@mail.gmail.com/>): ""There is no way I'll merge something this broken"" was his response. He was unhappy with the use of the new lock, which was being acquired even in situations where the page(s) in question already exist in the page cache and do not need to be instantiated. Finding pages in the page cache is one of the most performance-critical functions in the kernel, so adding unnecessary overhead there is highly unwelcome. Fixing that required another iteration of the patch set — and another development cycle waiting for the merge window to open again. 

By all appearances, this work _is_ now ready to go for 5.15; once that happens, this particular obscure race will have been closed. Even though the problem is evidently hard to hit, it would not be surprising to see this work backported to older kernels once a sufficient level of confidence in its stability has been reached. That will help to ensure that hole-punched files remain whole.
