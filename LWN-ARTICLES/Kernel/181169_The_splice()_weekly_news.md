---
title: The splice() weekly news
url: https://lwn.net/Articles/181169/
date: "April 24, 2006"
category: "System calls-vmsplice; splice"
author: 
---

Jens Axboe sent around [a note on the status of `splice()`](<https://lwn.net/Articles/181170/>). He notes that the `splice()` and `tee()` interfaces - on both the user and kernel side - should be stable now, with no further changes anticipated. The `sendfile()` system call has been reworked to use the `splice()` machinery, though that process will not be complete until after the 2.6.18 kernel cycle opens. 

While `splice()` might be stable, things are still happening. In particular, Jens has added [yet another system call](<http://lwn.net/Articles/180936/>): 

```
long vmsplice(int fd, void *buffer, size_t len, unsigned int flags);
```

While the regular `splice()` call will connect a pipe to a file, this call, instead, is designed to feed user-space memory directly into a pipe. So the memory range of `len` bytes starting at `buffer` will be pushed into the pipe represented by `fd`. The `flags` argument is not currently used. 

Using `vmsplice()`, an application which generates data in a memory buffer can send that data on to its eventual destination in a zero-copy manner. With a suitably-sized buffer, the application can do easy double-buffering; half of the buffer can be under I/O with `vmsplice()` while the other half is being filled. If the buffer is big enough, the application need only call `vmsplice()` each time half of the buffer has been filled, and the rest will simply work with no need for multiple threads or complicated synchronization mechanisms. 

Getting the buffer size right is important, however. If the buffer is at least twice as large as the maximum number of pages that the kernel will load into a pipe at an given time, a successful `vmsplice()` of half of the buffer can be safely interpreted by the application as meaning that the _other_ half of the buffer is no longer under I/O. Since half of the buffer will completely fill the space available within a kernel pipe, that half can only be inserted when all other data has been consumed out of the pipe - in simple situations, anyway. So, after `vmsplice()` succeeds, the application can safely refill the second half with new data. If the application gets confused, however, it could find itself overwriting data which has not yet been consumed by the kernel. 

Jens's patch adds a couple of `fcntl()` operations intended to help in this regard. The `F_GETPSZ` operation will return the maximum number of pages which can be inserted into a pipe buffer, which is also the maximum number of pages which can be under I/O from a `vmsplice()` operation. There is also `F_SETPSZ` for changing the maximum size, though that operation just returns `EINVAL` for now. Linus, however, [worries](<https://lwn.net/Articles/181176/>) that this information is not enough to know that a given page is no longer under I/O. In situations where there are other buffers in the kernel - perhaps just another pipe in series - the kernel could still have references to a page even after that page has been consumed out of the original pipe. Networking adds some challenges of its own: if a page has been `vmsplice()`ed to a TCP socket, it will not be reusable until the remote host has acknowledged the receipt of the data contained within that page. That acknowledgment will arrive long after the page has been consumed out of the pipe buffer. 

What this all means is that the `vmsplice()` interface probably needs a bit more work. In particular, there may need to be yet another system call which will allow an application to know that the kernel is done with a specific page. The current `vmsplice()` implementation is also unable to connect an incoming pipe to user-space memory. Making the read side work is a rather more complicated affair, and may not happen anytime in the near future.
