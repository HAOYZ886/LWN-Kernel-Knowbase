---
title: Finding the proper scope of a file collapse operation
url: https://lwn.net/Articles/589260/
date: "March 5, 2014"
category: fallocate
author: "By Jonathan Corbet March 5, 2014"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
March 5, 2014

System call design is never easy; there are often surprising edge cases that developers fail to consider as they settle on an interface. System calls involving filesystems seem to be especially prone to this kind of problem, since the complexity and variety of filesystem implementations means that there may be any number of surprises waiting for a developer who wants to create a new file-oriented operation. Some of these surprises can be seen in the discussion of a proposed addition to the `fallocate()` system call. 

[`fallocate()`](<http://man7.org/linux/man-pages/man2/fallocate.2.html>) is concerned with the allocation of space within a file; its initial purpose was to allow an application to allocate blocks to a file prior to writing them. This type of preallocation ensures that the needed space is available before trying to write the data that goes there; it can also help filesystem implementations lay out the allocated space more efficiently on disk. Later on, the `FALLOC_FL_PUNCH_HOLE` operation was added to deallocate blocks within a file, leaving a hole in the file. 

In February, Namjae Jeon proposed [a new `fallocate()` operation](<https://lwn.net/Articles/589262/>) called `FALLOC_FL_COLLAPSE_RANGE`; this proposal included implementations for the ext4 and xfs filesystems. Like the hole-punching operation, it removes data from a file, but there is a difference: rather than leaving a hole in the file, this operation moves all data beyond the affected range to the beginning of that range, shortening the file as a whole. The immediate user for this operation would appear to be video editing applications, which could use it to quickly and efficiently remove a segment of a video file. If the removed range is block-aligned (which would be a requirement, at least for some filesystems), the removal could be effected by changing the file's extent maps, with no actual copying of data required. Given that files containing video data can be large, it is not hard to understand why an efficient "cut" operation would be attractive. 

So what kinds of questions arise with an operation like this? One could start with the interaction with the `mmap()` system call, which maps a file into a process's address space. The proposed implementation works by removing all pages from the affected range to the end of the file from the page cache; dirty pages are written back to disk first. That will prevent the immediate loss of data that may have been written via a mapping, and will get rid of any memory pages that will be after the end of the file once the operation is complete. But it could be a surprise for a process that does not expect the contents of a file to shift around underneath its mapping. That is not expected to be a huge problem; as Dave Chinner [pointed out](<https://lwn.net/Articles/589267/>), the types of applications that would use the collapse operation do not generally access their files via `mmap()`. Beyond that, applications that are surprised by a collapsed file may well be unable to deal with other modifications even in the absence of a collapse operation. 

But, as Hugh Dickins [noted](<https://lwn.net/Articles/589268/>), there is a related problem: in the tmpfs filesystem, all files live in the page cache and look a lot like a memory mapping. Since the page cache _is_ the backing store, removing file pages from the page cache is unlikely to lead to a happy ending. So, before tmpfs could support the collapse operation, a lot more effort would have to go into making things play well with the page cache. Hugh was not sure that there would ever be a need for this operation in tmpfs, but, he said, solving the page cache issues for tmpfs would likely lead to a more robust implementation for other filesystems as well. 

Hugh also wondered whether the uni-directional collapse operation should, instead, be designed to work in both directions: 

I'm a little sad at the name COLLAPSE, but probably seven months too late to object. It surprises me that you're doing all this work to deflate a part of the file, without the obvious complementary work to inflate it - presumably all those advertisers whose ads you're cutting out, will come back to us soon to ask for inflation, so that they have somewhere to reinsert them. 

Andrew Morton went a little further, [suggesting](<https://lwn.net/Articles/589303/>) that a simple ""move these blocks from here to there"" system call might be the best idea. But Dave [took a dim view](<https://lwn.net/Articles/589304/>) of that suggestion, worrying that it would introduce a great deal of complexity and difficult corner cases: 

IOWs, collapse range is a simple operation, "move arbitrary blocks from here to there" is a nightmare both from the specification and the implementation points of view. 

Andrew [disagreed](<https://lwn.net/Articles/589503/>), claiming that a more general interface was preferable and that the problems could be overcome, but nobody else supported him on this point. So, chances are, the operation will remain confined to collapsing chunks out of files; a separate "insert" operation may be added in the future, should an interesting use case for it be found. 

Meanwhile, there is one other behavioral question to answer; what happens if the region to be removed from the file reaches to the end of the file? The current patch set returns `EINVAL` in that situation, with the idea that a call to `truncate()` should be used instead. Ted Ts'o [asked](<https://lwn.net/Articles/589308/>) whether such operations should just be turned directly into `truncate()` calls, but Dave is [set against](<https://lwn.net/Articles/589309/>) that idea. A collapse operation that includes the end of the file, he said, is almost certainly buggy; it is better to return an error in that case. 

There are also, evidently, some interesting security issues that could come up if a collapse operation were allowed to include the end of the file. Filesystems can allocate blocks beyond the end of the file; indeed, `fallocate()` can be used to explicitly request that behavior. Those blocks are typically not zeroed out by the filesystem; instead, they are kept inaccessible so that whatever stale data is contained there cannot be read. Without a great deal of care, a collapse implementation that allowed the range to go beyond the end of the file could end up exposing that data, especially if the operation were to be interrupted (by a system crash, perhaps) in the middle. Rather than set that sort of trap for filesystem developers, Dave would prefer to disallow the risky operations from the beginning, especially since there does not appear to be any real need to support them. 

So the end result of all this discussion is that the `FALLOC_FL_COLLAPSE_RANGE` operation is likely to go into the kernel essentially unchanged. It will not have all the capabilities that some developers would have liked to see, but it will support one useful feature that should help to accelerate a useful class of applications. Whether this will be enough for the long term remains to be seen; system call API design is hard. But, should additional features be needed in the future, new `FALLOC_FL` commands can be created to make them available in a compatible way.
