---
title: Handling writeback errors
url: https://lwn.net/Articles/718734/
date: "April 4, 2017"
category: "Block layer-Writeback"
author: "By Jake Edge April 4, 2017 LSFMM 2017"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
April 4, 2017

* * *

[LSFMM 2017](<https://lwn.net/Articles/lsfmm2017/>)

Error handling during writeback is something of a mess in Linux these days, Jeff Layton said in his plenary session to open the second day of the 2017 Linux Storage, Filesystem, and Memory Management Summit. He has investigated the situation and wanted to discuss it with attendees. He also presented a proposal for a way to make things better. Writeback is the process of asynchronously writing dirty pages from the page cache back to the underlying filesystem. 

The process started when he was looking into `ENOSPC` (out-of-space) errors for the Ceph filesystem. He found that the `PG_error` page flag, which is set when a problem occurs during writeback, would override any other error status and cause `EIO` (I/O error) to be returned. That inspired him to start looking at other filesystems to see what they did for this case and he found that there was no consistency among them. Dmitry Monakhov thought that writeback time was too late to get an `ENOSPC` error, but Layton said there are other ways to get it at that time. 

[ ![\[Jeff Layton\]](https://static.lwn.net/images/2017/lsfmm-layton-sm.jpg) ](<https://lwn.net/Articles/718792/>)

If there is an error during the writeback process, it should be returned when user space calls `close()` or `fsync()`, Layton said. The errors are tracked using the `AS_EIO` and `AS_ENOSPC` in the flags field of `struct address_space`. The errors are also tracked at the page level, sometimes, using `PG_error` in the page flags. 

A stray sync operation can clear the error flag without the error getting reported to user space, Layton said. `PG_error` is also used to track and report read errors, so a mixed read-write pattern can cause the flag to be lost before getting reported. There is also a question of what to do with the page after there is a writeback error for it; right now, the page is left in the page cache marked clean and up-to-date, so user space does not know there is a problem. 

So, Layton asked, "how do we fix this mess?" James Bottomley asked what granularity the filesystems want for error tracking. Layton said that `address_space` was the right level to track the information. Jan Kara pointed out that `PG_error` was meant to be used for read errors, but Layton said that some writeback implementations use it. 

Layton suggested cleaning up the tree to remove the use of `PG_error` for writeback errors. That would entail taking a pass through the tree to clean that up and to see if the errors on writeback are propagating out to user space or whether they may be getting cleared incorrectly. Ted Ts'o said there may be a need for a way to do writeback without getting any errors because they cannot be returned to user space. 

Bottomley said that he would rather not mark pages clean if they have not been written to disk. The error information would need to be tracked by sector, he said, so that the block layer can tell the filesystem where in a BIO (the structure used to describe block I/O requests) the error happened. Chris Mason suggested that anyone working on "redoing the radix tree" (i.e. [Matthew Wilcox](<https://lwn.net/Articles/684864/>)) might want to provide a way to store an error for a specific spot in the file. That way, the error could be reported then cleared once that reporting is done. 

Layton then presented an idea he had for tracking the errors. Writeback error counter and writeback error code fields would be added to the `address_space` structure and a writeback error counter would be added to `struct file`. When a writeback error occurs, the counter in `address_space` would be incremented and the error code recorded. At `fsync()` or `close()`, that error would be reported, but only if the counter in the `file` structure is less than that in `address_space`. In that case, the `address_space` counter value would be copied to the `file` structure. 

Monakhov asked why a counter was needed; Layton said it was to handle multiple overlapping writebacks. Effectively, the counter would record whether a writeback had failed since the file was opened or since the last `fsync()`. Ts'o said that should be fine; applications that want more information should use `O_DIRECT`. For most applications, knowledge that an error occurred somewhere in the file is all that is necessary; applications that require better granularity already use `O_DIRECT`. 

Layton's approach will result in some false positives, but there will be no false negatives, Ts'o said. Guaranteeing anything more would be much more expensive; avoiding the false positives would be a great goal, but the proposed method is simpler and has minimal performance impact. Layton said that it would definitely improve the situation for user space. 

Ts'o noted that there are some parallels in the handling of `ENOMEM` (out of memory). If that error gets returned deep in the guts of writeback, many of the same problems that Layton described occur. So, some filesystems desperately try not to return `ENOMEM` but, to avoid that, they have to hold locks, which gets in the way of the out-of-memory (OOM) killer. Ts'o has patches to defer writeback and leave pages in a dirty state rather than acquiring a set of locks and preventing OOM kills. 

But Layton thinks the first step should be to get better generic error handling for writeback. Filesystems avoid using the current error handling because it doesn't work well. After that, better handling of temporary errors (e.g. `ENOMEM`, `EAGAIN`, and `EINTR`) could be added. Another thing that should be addressed down the road is to have more [killable](<https://lwn.net/Articles/288056/>) or interruptible calls when doing writeback; that would help with NFS blocking indefinitely when the server goes away, for example. 

One other question Layton had is what should be done with the page when writeback fails. Right now, it is left clean, which is something of a surprise. If there is a hard error that will never be retried, should the page be invalidated? Ts'o said that they can't be left dirty in that case because the system will run out of memory if there are lots of writeback errors. If the radix tree could track the information, though, some advanced filesystem might try writing the page somewhere else. 

There are four bytes per 64 pages available in the radix tree, Wilcox said, if that would be useful for this tracking. Mason said that he would rather have one or two more tags available, but Wilcox said that could only happen with a 17% increase in the size of the tree—in which case, there would be lots more tags available. Jan Kara suggested storing the error information somewhere outside of the radix tree. The only applications that would benefit if filesystems could do something smarter are databases (many of which already use `O_DIRECT`) and large loopback files, Mason said. 

Steve French thought that the error handling should be done at a higher level, but Layton said that is how we got to where things are now. It is a bad API that is not clear how to use. He is going to try to fix that, he said, and developers should be looking for his [patches](<https://lwn.net/Articles/718648/>) to do so.
