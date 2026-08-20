---
title: "Fixing asynchronous I/O, again"
url: https://lwn.net/Articles/671649/
date: "January 13, 2016"
category: Asynchronous IO
author: "By Jonathan Corbet January 13, 2016"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
January 13, 2016

The process of adding asynchronous I/O (AIO) support to the kernel [began](<https://lwn.net/Articles/2965/>) with the 2.5.23 development kernel in June 2002\. Sometimes it seems that the bulk of the time since then has been taken up by complaints about AIO in the kernel. That said, AIO meets a specific need and has users who depend on it. A current attempt to improve the AIO subsystem has brought out some of those old complaints along with some old ideas for improving the situation. 

Linux AIO does suffer from a number of ailments. The subsystem is quite complex and requires explicit code in any I/O target for it to be supported. The [API](<http://man7.org/linux/man-pages/man2/io_submit.2.html>) is not considered to be one of our best and is not exposed by the GNU C library; indeed, the POSIX AIO support in glibc is implemented in user space and doesn't use the kernel's AIO subsystem at all. For files, only direct I/O is supported; despite [various attempts](<https://lwn.net/Articles/73847/>) over the years, buffered I/O is not supported. Even direct I/O can block in some settings. Few operations beyond basic reads and writes are supported, and those that are (`fsync()`, for example) are incomplete at best. Many have wished for a better AIO subsystem over the years, but what we have now still looks a lot like what was merged in 2002. 

Benjamin LaHaise, the original implementer of the kernel AIO subsystem, has recently returned to this area with [this patch set](<https://lwn.net/Articles/671632/>). The core change here is to short out much of the kernel code dedicated to the tracking, restarting, and cancellation of AIO requests; instead, the AIO subsystem simply fires off a kernel thread to perform the requested operation. This approach is conceptually simpler; it also has the potential to perform better and, in many cases, makes cancellation more reliable. 

With that core in place, Benjamin's patch set adds a number of new operations. It starts with `fsync()`, which, in current kernels, only works if the operation's target supports it explicitly. A quick `grep` shows that, in the 4.4 kernel, there is not a single `aio_fsync()` method defined, so asynchronous `fsync()` does not work at all. With AIO based on kernel threads, it is a simple matter to just call the regular `fsync()` method and instantly have working asynchronous `fsync()` for any I/O target supporting AIO in general (though, as Dave Chinner [pointed out](<https://lwn.net/Articles/671660/>), Benjamin's current implementation does not yet solve the whole problem). 

In theory, `fsync()` is supported by AIO now, even if it doesn't actually work. A number of other things are not. Benjamin's patch set addresses some of those gaps by adding new operations, including `openat()` (opens are usually blocking operations), `renameat()`, `unlinkat()`, and `poll()`. Finally, it adds an option to request reading pages from a file into the page cache (readahead) with the intent that later attempts to access those pages will not block. 

For the most part, adding these features is easy once the thread mechanism is in place; there is no longer any need to track partially completed operations or perform restarts. The attempts to add buffered I/O support to AIO in the past were pulled down by their own complexity; adding that support with this mechanism (not done in the current patch set) would not require much more than an internal `read()` or `write()` call. The one exception is the `openat()` support, which requires the addition of proper credential handling to the kernel thread. 

The end result would seem to be a significant improvement to the kernel's AIO subsystem, but Linus [still didn't like it](<https://lwn.net/Articles/671657/>). He is happy with the desired result and with much of the implementation, but he would like to see the focus be on the targeted capabilities rather than improving an AIO subsystem that, in his mind, is not really fixable. As he put it: 

If you want to do arbitrary asynchronous system calls, just *do* it. But do _that_, not "let's extend this horrible interface in arbitrary random ways one special system call at a time". 

In other words, why is the interface not simply: "do arbitrary system call X with arguments A, B, C, D asynchronously using a kernel thread". 

That's something that a lot of people might use. In fact, if they can avoid the nasty AIO interface, maybe they'll even use it for things like read() and write(). 

Linus suggested that the thread-based implementation in Benjamin's patch set could be adapted to this sort of use, but that the interface needs to change. 

Thread-based asynchronous system calls are not a new idea, of course; it has come around a number of times in the past under names like [fibrils](<https://lwn.net/Articles/219954/>), [threadlets](<https://lwn.net/Articles/223899/>), [syslets](<https://lwn.net/Articles/221913/>), and [acall](<https://lwn.net/Articles/316806/>). Linus even once [posted an asynchronous system call patch](<https://lwn.net/Articles/221914/>) of his own as these discussions were happening. There are some challenges to making asynchronous system calls work properly; there would have to be, for example, a whitelist of the system calls that can be safely run in this mode. As Andy Lutomirski [pointed out](<https://lwn.net/Articles/671661/>), ""exit is bad"". Linus also [noted](<https://lwn.net/Articles/671662/>) that many system calls and structures as presented by glibc differ considerably from what the kernel provides; it would be difficult to provide an asynchronous system call API that could preserve the interface as seen by programs now. 

Those challenges are real, but they may not prevent developers from having another look at the old ideas. But, as Benjamin was [quick to point out](<https://lwn.net/Articles/671658/>), none of those approaches ever got to the point where they were ready to be merged. He seemed to think that another attempt now might run into the same sorts of complexity issues; it is not hard to conclude that he would really rather continue with the approach he has taken thus far. 

Chances are, though, that this kind of extension to the AIO API is unlikely to make it into the mainline until somebody shows that the more general asynchronous system call approach simply isn't workable. The advantages of the latter are significant enough — and dislike for AIO strong enough — to create a lot of pressure in that direction. Once the dust settles, we may finally see the merging of a feature that developers have been pondering for years.
