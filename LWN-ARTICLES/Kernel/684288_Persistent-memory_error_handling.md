---
title: "Persistent-memory error handling"
url: https://lwn.net/Articles/684288/
date: "April 20, 2016"
category: "Memory management-Nonvolatile memory"
author: "By Jonathan Corbet April 20, 2016 LSFMM 2016"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
April 20, 2016

* * *

[LSFMM 2016](<https://lwn.net/Articles/lsfmm2016/>)

One of the key advantages of persistent memory is that it is, for lack of a better word, persistent; data stored there will be available for recall in the future, regardless of whether the system has remained up in the meantime. But, like memory in general, persistent memory can fail for a number of reasons and, given the quantities in which it is expected to be deployed, failures are a certainty. How should the operating system and applications deal with errors in persistent memory? One of the first plenary sessions at the [2016 Linux Storage, Filesystem, and Memory-Management Summit](<http://events.linuxfoundation.org/events/linux-storage-filesystem-and-mm-summit/>), led by Jeff Moyer, took on this question. 

Error handling with traditional block storage is relatively easy: an I/O request will fail with an `EIO` error, and the application, assuming it is prepared, can handle the error in whatever way seems best. But persistent memory looks like memory to the system, and memory errors are handled differently; in particular, they can trigger a low-level machine-check error. Some systems can recover from that machine check, others will be forced to reboot. Either way, the system has to be able to handle the problem. 

Time for a bit of terminology that caused some confusion in the session. Jeff was talking in particular about errors in "load" operations — reading from persistent memory using normal CPU instructions. Those were differentiated from "reads," which are file operations performed with a system call like `read()`. Similarly, "stores" (using memory operations) and "writes" (file operations) are seen differently. Errors with reads and writes can be returned via the normal system call status; errors with loads and stores are a bit more complicated. 

In cases where a machine check from a load operation is recoverable, the kernel can simply deliver the error to the application via a `SIGBUS` signal. But, even there, it became clear that the situation is not entirely simple: Keith Packard noted that the discussion was about load errors, and asked what happens when a store goes wrong. The problem there is that store operations are not usually synchronous, so there will be no immediate indication of an error. Paranoid software can [![\[Jeff Moyer\]](https://static.lwn.net/images/conf/2016/lsfmm/JeffMoyer-sm.jpg)](<https://lwn.net/Articles/684296/>) do a flush and a load after every store to ensure that the data has been stored properly; there does not seem to be any better way. 

On "less expensive" systems where the machine check is not recoverable, it's entirely possible that the system will end up in a reboot loop where, after each boot, it tries again to access the failing persistent-memory range. This behavior is generally seen as undesirable. As it turns out, even fancier hardware is sometimes subject to non-recoverable machine checks, so something has to be done to ensure reliable operation on all systems. 

The ACPI specification includes a mechanism for scrubbing an address range for errors; the UEFI firmware can run it as part of the boot process. Address ranges with errors can be flagged, and the operating system can, once it boots, query that list of ranges with errors and create a bad-block list that it knows must be avoided. When an application tries to access a range with an error via `mmap()`, the bad pages can be left unmapped and, should the application try to access them, the `SIGBUS` can be delivered. The scrubbing operation is not necessarily fast, so it would be unsurprising if it didn't run on every boot, but it can be run when errors begin happening. 

The solution as described so far, though, only works at the level of pages. The error granularity reported by the hardware can be as fine as a single 64-byte cache line; marking an entire (4KB) page as being bad when only 64 bytes have been truly lost is less than ideal. One way of narrowing things down would be for the application to open the file with the reported data loss and issue a series of 512-byte reads, narrowing the problem down to a single 512-byte "sector." But, Jeff said, that "still seems a little perverse." It would be nice to be able to directly inform an application about exactly what has been lost. 

A number of possibilities for providing that information were discussed. Christoph Hellwig suggested that the information provided with the `SIGBUS` signal could be expanded to include the exact range that was lost. Dan Williams said that the application could read the bad-block list from sysfs, then use the [`FIEMAP` `ioctl()` operation](<https://lwn.net/Articles/440255/>) to figure out which block in the file was bad. That works today, he said, except that the bad-block list is not updated while the system is live. Ted Ts'o said it would be useful to have a new `ioctl()` command to query the failing byte range directly. 

James Bottomley said that the most friendly approach would be to remap the bad block and hide it entirely, only informing applications if data has actually been lost. It was agreed that remapping would work if an error is detected on a write operation, but the real problem is with reads (or, properly, loads) where data is known to have been lost. In that case, applications should not be forced to dig through the bad-block list; there should be a more direct interface. There also needs to be some sort of interface to clear the error (typically remapping the block) so that the given address range becomes usable again. 

As the session wound down, a few residual questions came up, but no real decisions were reached. Ted asked whether the problem of non-recoverable machine checks would go away as the hardware improves; Jeff answered that it might, but that doesn't change the real issue of how to convey problems to user space. Ted also asked whether this information should be provided to applications at all — isn't that assuming a fundamental change in application behavior? Ric Wheeler answered that applications that care about data integrity already keep multiple copies of the data; they just need to know where things go wrong. 

As has been seen for a while, persistent memory raises a number of questions with regard to how it should be presented to user space. While many of the problems are being solved, it seems likely that persistent memory will be a discussion topic at events like this for some time yet.
