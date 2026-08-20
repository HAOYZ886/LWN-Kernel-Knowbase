---
title: "Kernel Summit: Asynchronous I/O"
url: https://lwn.net/Articles/94566/
date: "July 21, 2004"
category: Asynchronous IO
author: 
---

This article is part of LWN's [2004 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2004/>).   
---  
Suparna Bhattacharya has done much of the recent work on asynchronous I/O, so she was the logical person to lead a session on that subject. She noted that numerous patches exist which address many of the current shortcomings of mainline AIO; these patches include the retry mechanism, support for asynchronous buffered filesystem I/O, pipe I/O, and the `poll()` system call. Vector AIO (a mechanism for joining multiple asynchronous requests and submitting them together) is also in the works. 

Beyond simply completing the implementation, Linux AIO could use a number of other changes. One idea calls for separating the current `aio_read()` `file_operations` method into `submit_aio_read()` and `complete_aio_read()` (and similarly for the write side). Splitting the AIO method in that way would apparently make retry-based implementations easier to implement. The kernel currently supports "synchronous kiocbs," which are a way of requesting synchronous operations via the AIO paths; it may come as little surprise that nobody is actually using this mechanism, so it can come out. 

In a more general sense, it is perhaps time to reexamine the relationship between synchronous and asynchronous I/O. One could look at synchronous operations as really just being the asynchronous variety with a wait added at the end. If the kernel were restructured along those lines, some things would be simplified, but it would imply that retry-based schemes (used for buffered filesystem AIO - see [LWN's coverage](<http://lwn.net/Articles/73847/>)) would be used for synchronous I/O as well. 

The topic turned to asynchronous `fsync()`; Linus noted that, internally, this call is implemented by starting the requisite I/O, then waiting for completion. It would not be hard to export a separate, asynchronous `fsync()` operation. Why not, asked Suparna, simply use the existing AIO interface for that? The answer, it seems, is that nobody likes the current AIO interface. Few people use it; even glibc avoids it. 

So, one might ask, how can this interface be fixed? One idea was to add an `aio_suspend()` system call which could wait for (one or more of) multiple I/O contexts to complete. Other ideas included using signals to notify processes of completion, or simply to use the epoll calls. One way of doing that could be to create a new file descriptor for each outstanding AIO operation which could then be passed to epoll. 

The real problem with AIO, however, remains the (perceived) lack of users for this capability. The existing AIO code is used by a few high-end databases for their file I/O, and MySQL is apparently being extended to make use of it - but that is about it. Really pushing AIO forward will require that more users step up, and that the performance benefits of working in this mode be better demonstrated. 

[>>Next: Multipath I/O](<https://lwn.net/Articles/94569/>).
