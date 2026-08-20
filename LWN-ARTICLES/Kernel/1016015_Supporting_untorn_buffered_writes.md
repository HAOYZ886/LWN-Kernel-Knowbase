---
title: Supporting untorn buffered writes
url: https://lwn.net/Articles/1016015/
date: "April 4, 2025"
category: Atomic IO operations
author: "By Jake Edge April 4, 2025 LSFMM+BPF"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
April 4, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

At last year's Linux Storage, Filesystem, Memory-Management, and BPF Summit (LSFMM+BPF), there was a [discussion about atomic writes](<https://lwn.net/Articles/974578/>) that was accompanied by patches to support the feature in the block layer, and for direct I/O on XFS. That work was merged, but another piece of that discussion concerned adding the feature for buffered I/O, in part because the PostgreSQL database currently has to jump through hoops to ensure that its writes are not "torn" (partially written) when there is an error or crash. Luis Chamberlain led a combined storage and filesystem track at this year's summit to revisit the idea of providing atomic (or untorn) writes for buffered I/O. 

[ ![\[Luis Chamberlain\]](https://static.lwn.net/images/2025/lsfmb-chamberlain-sm.png) ](<https://lwn.net/Articles/1016409/>)

Chamberlain suggested that there was a belief that it did not make sense to work on buffered atomic I/O simply to work around a missing feature in PostgreSQL; some think that the database should just support direct I/O. It turns out that the default storage engine for MongoDB supports both buffered and direct I/O, he said, but MongoDB recommends using buffered. The reason is that MongoDB compresses data on disk by default and keeps the data uncompressed in its cache. The data can be accessed via [`mmap()`](<https://www.man7.org/linux/man-pages/man2/mmap.2.html>), which is not compatible with direct I/O. 

He thinks that the database developers should be able to decide on the architecture that works best for their needs. Providing untorn buffered writes allows the databases to eliminate the double-buffering they are doing now as a workaround. There are configuration options to turn off the double-buffering for MySQL and PostgreSQL, which can be used to test the impacts of the change. 

The atomic-write API could eventually be used by databases to provide the torn-write prevention, but a prototype can be run without it to verify that the databases are writing with the correct sizes and alignment needed by that API. In his [slides](<https://docs.google.com/presentation/d/e/2PACX-1vRdyZz362oQj9ma0acha6K-n8nmx-jxmy0sxsXdLSIQzh7reALDpH6nG_WXVrkJZeOLW7QCYpKokGit/pub>), Chamberlain showed graphs of running MySQL and PostgreSQL with and without the double-buffering options. Both showed higher average transactions per second, with much less variability, without double-buffering. To reproduce these results, Chamberlain recommended using [blkalgn](<https://github.com/dkruces/bcc/blob/lbs/tools/blkalgn.py>), which he called ""the bees knees for I/O atomic-alignment verification and visualization"". The tests are [integrated into his kdevops](<https://github.com/linux-kdevops/kdevops/blob/main/docs/sysbench/sysbench.md>) kernel development and testing tool. 

Chamberlain wondered what the next steps might be. John Garry said that he thought the testing needed to validate the idea should also be run with more threads because, at least for MySQL, his testing showed some contention when multiple threads were writing. Chamberlain agreed with that, noting that the tests can easily be run in kdevops, so doing so with various numbers of threads needs to happen; the tests run for a long time (12 hours), so he has not yet gotten to further testing. He said that various members of the community will need to do their own homework to decide whether it makes sense to support the feature; if so, then there is the question of what the API should be. 

As Chamberlain had mentioned earlier, Ted Ts'o noted that various large cloud vendors (hyperscalers) already have hosted MySQL solutions that are taking advantage of untorn writes. They are doing so ""without any upstream patches, just auditing code paths, and it mostly works as long as you are really, really careful"". So he agrees that the community needs to do its homework, but vendors have made it clear that they see advantages, at least for MySQL and direct I/O. 

His main concern regarding atomic buffered I/O is the semantics of the `RWF_ATOMIC` flag. The database people only need untorn writes for 8KB, 16KB, and, maybe someday, 32KB sizes, but there is a contingent in some parts of the filesystem-development community that believes a 1MB write with the atomic flag should be fully supported. That would be painful to do for direct I/O, but it is exceedingly difficult for buffered I/O. The kernel needs to track different atomic-write sizes as they make their way through the page cache and onto the storage medium. There may be ""fancy ways to do that [with] large folios and making darn sure you don't break apart a large folio when that happens"", but he thinks it makes more sense to restrict the size of untorn writes. 

One additional concern that Ts'o has is with writeback when a page is locked because, for example, a page fault is in progress. Currently, the writeback thread simply skips any locked pages, which could result in a torn write. He thinks the XFS implementation for atomic writes takes care of that problem, though he has not looked closely, but a more general solution is probably required. 

David Howells asked about the interaction between atomic writes and `mmap()`. The alignment and length of the mapped part of the file need to be the same as that required by the atomic writes, Ts'o said, which is presumably what was done for XFS. Another problem comes when buffered and direct I/O are both being done to the same file, Howells said. Ts'o said that the filesystem community has always recommended against combining buffered and direct I/O, but, since it is known that MySQL and its backup program already do so, the community ""made it safe in some circumstances""; it all just works 99.9% of the time, he said. 

Chamberlain said that because the `RWF_ATOMIC` flag is marking the I/O, filesystems can prevent problematic combinations. Amir Goldstein suggested that files be opened for either atomic direct I/O or atomic buffered I/O, which is similar to the restriction added for [FUSE passthrough mode](<https://lwn.net/Articles/932060/>); there is a flag on the inode of the file while it is open that indicates its mode. Jeff Layton pointed out that `RWF_ATOMIC` is a flag on the I/O operation, not the open, but it could be used to simply return `EINVAL` for operations that would violate the combination rules. 

Ts'o thought it made sense to switch to an open flag, and suggested that `O_UNTORN` would be the right name; the granularity for the untorn writes could be placed in the inode. One of the problems he sees is that the developers have been using the term "atomic" because that is term used by SCSI and NVMe, but then people wanted to make 1MB atomic writes work, which is not at all what the database developers care about. Switching to untorn and being clear about the granularity supported will help simplify the API and lead to the feature landing much sooner, he thinks. 

Chamberlain asked Jens Axboe about the [`RWF_UNCACHED` flag for uncached, buffered I/O](<https://lwn.net/Articles/998783/>) and whether it would be suitable to use for untorn writes. Axboe did not really have an opinion, as it is largely a filesystem, rather than a block layer, concern, but could see that some of the effects of the flag might be useful—immediately kicking off writeback, for example. Ts'o cautioned that the PostgreSQL developers actually want the page cache to manage the caching for the database, as he understands it. One of the reasons that they have not switched to using direct I/O is that they would need to do their own user-space cache management; he suggested talking with the PostgreSQL developers to assess their needs. 

Chris Mason said that the need for cached, untorn writes does not mean that uncached, untorn writes should not be supported as well. Chamberlain and others agreed with that. Christian Brauner noted that adding an open flag and marking the inode would mean that other users of the file might be precluded; that implies that privileges of some sort should be required. The "deadly combination" is a file that is open for untorn and then gets opened for direct I/O, Ts'o said; that is a rare enough combination that the second open could just fail. 

The session had run out of time at that point, but Chamberlain said that it would seem that developers are interested in supporting untorn buffered writes, but only with some restrictive rules that had not been determined yet. Goldstein suggested starting slowly with some kind of opt-in, perhaps via a mount option or filesystem-creation flag, then possibly growing the feature from there.
