---
title: Volatile ranges with fallocate()
url: https://lwn.net/Articles/500382/
date: "June 5, 2012"
category: Memory management; Volatile ranges; fallocate
author: "By Jonathan Corbet June 5, 2012"
---

By **Jonathan Corbet**  
June 5, 2012

Last November LWN looked at the [volatile ranges patch set](<https://lwn.net/Articles/468896/>) from John Stultz. This patch is intended to bring an Android feature into the mainline, but it is a reimplemented feature that is more deeply tied into the memory management subsystem. That patch has now returned, but the API has changed so another look is warranted. 

A "volatile range" is a set of pages in memory containing data that might be useful to an application at some point in the future; a key point is that, if the need arises, the application is able to reacquire (or regenerate) that data from another source. A web browser's in-RAM image cache is a classic example. Keeping the images around can reduce net traffic and improve page rendering times, but, should the cached images vanish, the browser can request a new copy from the source. Thus, while it makes sense to keep this data around, it also makes sense to get rid of it if a more pressing need for memory arises. 

If the kernel knew about this sort of cached data, it could dump that data during times of memory stress and quickly reclaim the underlying memory. In such a situation, applications could cache more data than they otherwise would, knowing that there are limits to how much that caching can affect the system as a whole. The result would be better utilization of memory and a system that performs better overall. 

Google's Robert Love implemented such a mechanism for Android as "ashmem." There is a desire to get the ashmem functionality into the mainline kernel, but the implementation and API were not to everybody's taste. To get around that problem, John took the core ashmem code, reworked the virtual memory integration, and hooked it into the `posix_fadvise()` system call; that is the version of the patch that was described last November. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Dave Chinner subsequently pointed out that this functionality might be better suited to the `fallocate()` system call. That call looks like this: 

```
int fallocate(int fd, int mode, off_t offset, off_t len);
```

This system call is meant to operate on ranges of data within a file. Of particular interest, perhaps, is the `FALLOCATE_FL_PUNCH_HOLE` operation, which removes a block of data from an arbitrary location within a file. Declaring a volatile range can be thought of as a form of hole punching, but with a kernel-determined delay. If memory is tight, the hole could be punched immediately; otherwise the operation could complete at some later time, or not at all. Given the similarity between these two operations, it made sense to implement them within the same system call; John duly reworked the patch along those lines. 

With the [new patch set](<https://lwn.net/Articles/498894/>), to declare that a range of a file's contents is volatile, an application would call: 

```
fallocate(fd, FALLOCATE_FL_MARK_VOLATILE, offset, len);
```

Where `offset` and `len` describe the actual range to be marked. After the call completes, the kernel is not obligated to keep that range in memory, and is not obligated to write that range to backing store before reclaiming it. The application should not attempt to access that portion of the file while it has been marked volatile, since the contents could disappear at any time. Instead, if and when the data turns out to be useful, a call should be made to: 

```
fallocate(fd, FALLOCATE_FL_UNMARK_VOLATILE, offset, len);
```

If the indicated range is still present in memory, the call will return zero and the application can proceed to work with the data. If, instead, any part of the range has been purged by the kernel since it was marked volatile, a non-zero return value will inform the application that it needs to find that data somewhere else. 

Any filesystem could conceivably implement this functionality, but, in practice, it only makes sense for a RAM-based filesystem like tmpfs, so it is only implemented there. 

The patch is in its third revision as of this writing, having gotten a number of comments in its first two iterations. The number of complaints has fallen off considerably, though, suggesting that most reviewers are happy now. So this feature may just find its way into the 3.6 kernel.
