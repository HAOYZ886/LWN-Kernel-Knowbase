---
title: "KS2007: Scalability"
url: https://lwn.net/Articles/248931/
date: "September 9, 2007"
category: Scalability
author: "By Jonathan Corbet September 9, 2007 LWN.net Kernel Summit 2007 coverage"
---

By **Jonathan Corbet**  
September 9, 2007

* * *

LWN.net [Kernel Summit 2007](<https://lwn.net/Articles/KernelSummit2007/>) coverage 

Scalability was the subject of a day-2 session which focused primarily on small details. For example, there was concern about how well the slab allocator scales on NUMA systems; there was talk of doing more performance testing until it was pointed out that slab's days are numbered. Chances are it will be removed in an upcoming kernel release in favor of the SLUB allocator. 

The SLOB allocator, it was noted, has been thoroughly reworked to help improve its performance on small systems. 

There was a discussion on I/O performance. On a multi-CPU system, there is a cache penalty to be paid whenever the CPU which submits the operation is different from the CPU which handles the completion of that operation. It is hard to see how to solve this problem in the general case. One could take pains to submit operations on the CPU where completion is expected to be handled, but that really just moves the cost around. On sufficiently smart hardware, one can try to direct completion interrupts to the CPU which submitted the operation. Beyond that, there were not a whole lot of ideas going around. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Dave Chinner raised a separate issue: it seems that booting a system with 20,000 block devices can be just a little slow.. He's seeing this problems with installations running a mere 300TB of storage. Asynchronous scanning is seen as at least part of the solution to this problem. There is a related problem, though: it turns out that, with so many drives, finding a specific one can be a bit challenging. There is not much that the kernel can do about that one, though. 

There is, it seems, a need to rework the direct I/O code to separate the memory setup from the I/O submission paths. There are places in the kernel which would like to submit direct I/O requests, but which are not working with user-space buffers. Currently the code is sufficiently intermixed that the direct I/O handling code cannot be used with buffers in kernel space. Another issue has to do with locking when both buffered and direct I/O are being performed on the same file. It is, apparently, difficult to keep the file's contents coherent in such situations. 

There was also some talk about giving the filesystems more flexibility in how they handle writeout of dirty pages from memory. By handling writeout in the right order, it should be possible to improve performance. Increased interest in execute-in-place support was expressed; as the capacity of flash-based devices grows, it will make more sense to store system images there and run directly from the device. 

On the small-systems side of scalability, the big issue remains the overall footprint of the kernel. That footprint is growing, of course, which presents a challenge for embedded deployments. Better ways of quantifying kernel size would be helpful.
