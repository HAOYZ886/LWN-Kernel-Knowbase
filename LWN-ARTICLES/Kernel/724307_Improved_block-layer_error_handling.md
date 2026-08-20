---
title: "Improved block-layer error handling"
url: https://lwn.net/Articles/724307/
date: "June 2, 2017"
category: "Block layer-Error handling"
author: "By Jonathan Corbet June 2, 2017"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
June 2, 2017

The kernel's filesystem and block layers are places where a lot of things can go wrong, often with unpleasant consequences. To make things worse, when things do go wrong, informing user space about the problem can be difficult as a consequence of how block I/O works. That can result in user-space applications being unaware of trouble at the I/O level, leading to lost data and enraged users. There are now two separate (and complementary) proposals under discussion that aim to improve how error reporting is handled in the block layer. 

#### Block-layer error codes

One problem with existing reporting mechanisms is that they are based on standard Unix error codes, but those codes were never designed to handle the wide variety of things that can go wrong with block I/O. As a result, almost any type of error ends up being reported back to the higher levels of the block layer (and user space) as `EIO` (I/O error) with no further detail available. That makes it hard to determine, at both the filesystem and user-space levels, what the correct response to the error should be. 

Christoph Hellwig is working to change that situation by adding [a dedicated set of error codes](<https://lwn.net/Articles/724174/>) to be used within the block layer. This patch set adds a new `blk_status_t` type to describe block-level errors. The specific error codes added thus far correspond mostly to the existing Unix codes. So `BLK_STS_TIMEOUT`, indicating an operation timeout, maps to `ETIMEDOUT`, while `BLK_STS_NEXUS`, describing a problem connecting to a remote storage device, becomes `EBADE` ("invalid exchange"). There is, according to Hellwig, ""some low hanging fruit"" that can be improved by additional error codes, but those codes are not added as part of this patch set. 

The new errors can be generated at the lowest levels of the kernel's block drivers, and will be propagated to the point that filesystem code sees them in the results of its block I/O requests. To get there, the `bi_error` field in `struct bio`, which contained a Unix error code, has been renamed to `bi_status`. In-tree filesystems have been changed to use the new field, but they do not yet act on the additional information that may be available there. 

This is, in other words, relatively early infrastructural work that makes it possible for the block layer to produce better error information. Actually making use of that infrastructure will have to wait until this work is accepted and headed toward the mainline. 

#### Reporting writeback errors

One particular challenge for block I/O error reporting is that many I/O requests are not the direct result of a user-space operation. Most file data is buffered through the kernel's page cache, and there can be a significant delay between when an application writes data into the cache and when a writeback operation flushes that data to persistent storage. If something goes wrong during writeback, it can be hard to report that error back to user space since the operation that caused that writeback in the first place will have long since completed. The kernel makes an attempt to save the error and report it on a subsequent system call, but it is easy for that information to be lost with the result that the application is unaware that it has lost data. 

Jeff Layton's [writeback-error reporting patches](<https://lwn.net/Articles/724232/>) are an attempt to improve this situation. He adds a mechanism that is based on the idea that applications that care about their data will occasionally call `fsync()` to ensure that said data has made it to persistent storage. Current kernels _might_ report a writeback error on an `fsync()` call, but there are a number of ways in which that can fail to happen. With the new mechanism in place, any application that holds an open file descriptor will reliably get an error return on the first `fsync()` call that is made after a writeback error occurs. 

To get there, the patch set creates a new type (`errseq_t`) for the reporting of writeback errors. It is a 32-bit value with two separate fields: an error code (of the standard Unix variety) and a sequence counter. That counter tracks the number of times that an error has been reported in that particular `errseq_t` value; kernel code can remember the counter value of the last error reported to user space. If the counter increases on a future check, a new error has been encountered. 

The `errseq_t` variables are added to the `address_space` structure, which controls the mapping between pages in the page cache and those in persistent storage. The writeback process uses this structure to determine where dirty pages should be written to, so it is a logical place to store error information. Meanwhile, any open file descriptor referring to a given file will include a pointer to that `address_space` structure, so this `errseq_t` value is visible (within the kernel) to all processes accessing the file. Each open file (tracked by `struct file`) gains a new `f_wb_err` field to remember the sequence number of the last reported error. 

Storing that value in the `file` structure has an important benefit: it makes it possible to report a writeback error exactly once to every process that calls `fsync()` on that file, regardless of when they make that call. In current kernels, only the first caller after an error occurs has a chance of seeing that error information. It would arguably be better to report the error only to the process that actually wrote the data that experienced the error, but tracking things at that level would be cumbersome and slow. By informing all processes, this mechanism ensures that the right process will get the news. 

The final step is to get the low-level filesystem code to use the new reporting mechanism when something goes wrong. Rather than convert all filesystems at once, Layton chose to add a new filesystem-type flag (`FS_WB_ERRSEQ`) that can be set for filesystems that understand the new scheme. Code at the virtual filesystem layer can then react accordingly depending on whether the filesystem has been converted or not. The intent is to remove this flag and the associated mechanism once all in-tree filesystems have made the change. 

The ideas behind this patch set were [discussed](<https://lwn.net/Articles/718734/>) at the [2017 Linux Storage, Filesystem, and Memory-Management Summit](<https://lwn.net/Articles/lsfmm2017/>) in March; the patches themselves have been through five public revisions since then. There is a reasonable chance that they are approaching a sort of final state where they can be considered for merging in an upcoming development cycle. The result will not be perfect writeback error reporting, but it should be significantly better than what the kernel offers now.
