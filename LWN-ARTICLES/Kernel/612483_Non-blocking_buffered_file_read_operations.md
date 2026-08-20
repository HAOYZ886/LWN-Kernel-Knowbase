---
title: "Non-blocking buffered file read operations"
url: https://lwn.net/Articles/612483/
date: "September 23, 2014"
category: Asynchronous IO; System calls
author: "By Jonathan Corbet September 23, 2014"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
September 23, 2014

It is natural to think of buffered I/O on Unix-like systems as being asynchronous. In Linux, the page cache exists to separate process-level I/O requests from the physical requests sent to the storage media. But, in truth, some operations are synchronous; in particular, a read operation cannot complete until the data is actually read into the page cache. So a call to `read()` on a normal file descriptor can always block; most of the time this blocking causes no difficulties, but it can be problematic for programs that need to always be responsive. Now, a partial solution is in the works for this kind of code, but it comes at the cost of adding as many as four new system calls. 

The problem of blocking buffered reads is not new, of course, so applications have worked around it in a number of ways. One common approach is to create a set of threads dedicated to performing buffered I/O. Those threads can block while other threads in the program continue to do other work. This solution works and can be efficient, but it inevitably adds a certain amount of inter-thread communication overhead, especially in cases where the desired data is already in the page cache and a `read()` call could have completed immediately. 

A recent [patch set](<https://lwn.net/Articles/612421/>) from Milosz Tanski attempts to solve the problem a different way. Milosz's approach is to allow a program to request non-blocking behavior at the level of a single `read()` call. Unfortunately, the current `read()` and variants do not have a "flags" argument, so there is no way to express that request using them. So Milosz adds two new versions of each of `read()` and `write()`: 

```
int readv2(unsigned long fd, struct iovec *vec, unsigned long vlen, int flags);
        int writev2(unsigned long fd, struct iovec *vec, unsigned long vlen, int flags);
        int preadv2(unsigned long fd, struct iovec *vec, unsigned long vlen,
    		unsigned long pos_l, unsigned long pos_h, int flags);
        int pwritev2(unsigned long fd, struct iovec *vec, unsigned long vlen,
    		 unsigned long pos_l, unsigned long pos_h, int flags);
```

In each case, the system call is just like its predecessor with the exception of the addition of the `flags` argument. Note that the two offset parameters (`pos_l` and `pos_h`) to `preadv2()` and `pwritev2()` will be combined into a single `off_t` parameter at the C library level. 

In Milosz's patch set, the only supported flag is `RWF_NONBLOCK`, which requests non-blocking operation. If a read request is accompanied by this flag, it will only complete if (at least some of) the requested data is already in the page cache; otherwise it returns `EAGAIN`. The current patch does not start any sort of readahead operation if it is unable to satisfy a non-blocking read request. The new write operations do not support non-blocking operation; the `flags` argument must be zero when calling them. Adding non-blocking behavior to `write()` is possible; such a write would only complete if memory were immediately available for a copy of the data in the page cache. But that implementation has been left as a future exercise. 

#### Considering the alternatives

The patch is relatively simple and straightforward, but one might well wonder: why is it necessary to add a new set of system calls for non-blocking operation when the kernel has long supported this mode via either the `O_NONBLOCK` flag to `open()` or `fcntl()`? There are, it seems, a couple of reasons for not wanting to implement ordinary non-blocking I/O behavior for regular files, the first of which being that it will break applications. 

Given that non-blocking I/O is an optional behavior that must be explicitly requested, it is not obvious that supporting it for regular files would create trouble. It comes down to the fact that passing `O_NONBLOCK` to an `open()` call actually requests two different things: (1) that the `open()` call, itself, not block, and (2) that subsequent I/O be non-blocking. There are applications that use `O_NONBLOCK` for the first reason; Samba uses it, for example, to keep an `open()` call from blocking in the presence of locks on the file. Since buffered reads have always blocked regardless of `O_NONBLOCK`, applications do not concern themselves with calling `fcntl()` to reset the flag before calling `read()`. If those `read()` calls start returning `EAGAIN`, the application, which is not expecting that behavior, will fail. 

One could argue that this behavior is incorrect, but it has worked for decades; breaking these applications with a kernel change is not acceptable. Samba is not the only application to run into trouble here; evidently [squid and GQview](<https://lwn.net/Articles/612805/>) fail as well. So the problem is clearly real. 

Beyond that, as Volker Lendecke [explained](<https://lwn.net/Articles/612806/>), full non-blocking behavior would not play well with how applications like Samba want to use this feature. The wish is to attempt to read the needed data in the non-blocking mode; should the data not be available, the request will be turned over to the thread pool for synchronous execution. If the thread pool is using the same file descriptor, its attempts to perform blocking reads will fail. If it uses a different file descriptor, it can run into weird problems relating to the surprising semantics of POSIX file locks (see [this article](<https://lwn.net/Articles/586904/>) for more information). So the ability to request non-blocking behavior on a per-read basis is needed. 

Another possibility would be to add a version of the `fincore()` system call, which allows a process to ask the kernel whether a specific range of file data is present in the page cache. Patches adding `fincore()` have been around since [at least 2010](<https://lwn.net/Articles/371538/>). But `fincore()` is seen as a bit of an indirect route toward the real goal, and there is always the possibility that the situation might change between a call to `fincore()` and the application's decision to do something based on the result. Requesting non-blocking behavior with the `read()` avoids that particular race condition. 

Finally, one could also consider the kernel's asynchronous I/O subsystem, which allows an application to obtain non-blocking behavior on a per-request basis. But asynchronous I/O has never been supported for buffered I/O, and [attempts to add that functionality](<https://lwn.net/Articles/73847/>) have bogged down in the sheer complexity of the problem. Adding non-blocking behavior to `read()` — where, unlike with asynchronous I/O, a request can simply fail if it cannot be satisfied immediately — is far simpler. 

So the end result would appear to be that we will get a new set of Linux-specific system calls allowing applications to request non-blocking `read()` behavior on regular files. The rate of change on this patch set is slowing — though it is worth noting that `readv2()` and `writev2()` have been removed from [the latest version](<https://lwn.net/Articles/613068/>) (as of this writing) of the patch set. It is getting late to have this code ready for the 3.18 development cycle, but it should be more than ready for 3.19.
