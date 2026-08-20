---
title: Extending system calls
url: https://lwn.net/Articles/281965/
date: "May 14, 2008"
category: System calls; indirect
author: "By Jake Edge May 14, 2008"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
May 14, 2008

Getting interfaces right is a hard, but necessary, task, especially when that interface has to be supported "forever". Such is the case with the system call interface that the kernel presents to user space, so adding features to it must be done very carefully. Even so, when Ulrich Drepper set out to [remove a hole that could lead to a race condition](<http://lwn.net/Articles/236486/>), he probably did not expect all the different paths that would need to be tried before closing in on an acceptable solution. 

The problem stems from wanting to be able to create file descriptors with new properties--things like close-on-exec, non-blocking, or non-sequential descriptors. Those features were not considered when the system call interface was developed. After all, many of those system calls are essentially unchanged from early Unix implementations of the 1970s. The `open()` call is the most obvious way to request a file descriptor from the kernel, but there are plenty of others. 

In fact, `open()` is one of the easiest to extend with new features because of its `flags` argument. Calls like `pipe()`, `socket()`, `accept()`, `epoll_create()` and others produce file descriptors as well, but don't have a `flags` argument available. Something different would have to be done to support additional features for the file descriptors resulting from those calls. 

The close-on-exec functionality is especially important to close a security hole for multi-threaded programs. Currently, programs can use `fcntl()` to change an open file descriptor to have the close-on-exec property, but there is always a window in time between the creation of the descriptor and changing its behavior. Another thread could do an `exec()` call in that window, leaking a potentially sensitive file descriptor into the newly run program. Closing that window requires an in-kernel solution. 

Back in June of last year, after some false starts, Linus Torvalds [suggested adding an `indirect()` system call](<http://lwn.net/Articles/237722/>), as a way to pass flags to system calls that don't currently support them. The `indirect()` call would apply a set of flags to the invocation of an existing system call. This would allow existing calls to remain unchanged, with only new uses calling `indirect()`. User space programs would be unlikely to call the new function directly, instead they would call `glibc` functions that handled any necessary `indirect()` calls. 

Davide Libenzi created a `sys_indirect()` patch in July, but Drepper saw it as "more complex than warranted". So Drepper created his own "trivial" implementation, that was [described on this page](<http://lwn.net/Articles/259068/>) in November. It was met with a less than enthusiastic response on linux-kernel for being, amongst other things, an exceedingly ugly interface. 

The alternative to `sys_indirect()` is to create a new system call for each existing call that needed a flags argument. This was seen as messy by some, including Torvalds, leading some kernel hackers into looking for alternatives. The indirect approach also had some other potential benefits, though, because it was seen as something that could be used by [syslets](<http://lwn.net/Articles/261473/>) to allow asynchronous system calls. No decision seemed to be forthcoming, leading Drepper to [ask Torvalds for one](<https://lwn.net/Articles/282210/>): 

Will you please make a decision regarding sys_indirect? There has been no other proposal so the alternative is to add more syscalls. 

To bolster his argument that `sys_indirect()` was the way to go, Drepper also created a [patch to add some of the required system calls](<https://lwn.net/Articles/282211/>). He started with the `socket()` family, by adding `socket4()`, `socketpair5()`, and `accept4()`--tacking the number of arguments onto the function name a la `wait3()` and `wait4()`. Drepper's intent may not have been well served by choosing those calls as Alan Cox immediately [noted that the `type` argument could be overloaded](<https://lwn.net/Articles/282212/>): 

Given we will never have 2^32 socket types, and in a sense this is part of the type why not just use 

```
socket(PF_INET, SOCK_STREAM|SOCK_CLOEXEC, ...)
```

that would be far far cleaner, no new syscalls on the socket side at all. 

Michael Kerrisk looked over the set of system calls that generate file descriptors, [categorizing them based on whether they needed a flag argument](<https://lwn.net/Articles/282213/>) added. He observed that roughly half of the file descriptor producing calls need not change because they could either use an overloading trick like the socket calls, the glibc API already added a flags argument, or there were alternatives available to provide the same functionality along with flags. 

In response, Drepper made one last attempt to push the indirect approach, [saying](<https://lwn.net/Articles/282214/>): 

Or we just add sys_indirect (which is also usable for other syscall extensions, not just the CLOEXEC stuff) and let userlevel (i.e., me) worry about adding new interfaces to libc. As you can see, for the more recent interfaces like signalfd I have already added an additional parameter so the number of interface changes would be reduced. 

Even though the indirect approach has some good points, Torvalds liked the approach advocated by Cox, [saying](<https://lwn.net/Articles/282215/>): 

Ok, I have to admit that I find this very appealing. It looks much cleaner, but perhaps more importantly, it also looks both readable and easier to use for the user-space programmer. 

Ultimately, developers will only use these new interfaces if they can easily test for the existence of the new code. Torvalds [gives an example](<https://lwn.net/Articles/282216/>) of how that might be done using the `O_NOATIME` flag to `open()`, which has only been available since 2.6.8. It is this testability issue that makes him believe the flags-based approach is the right one: 

And that's the problem with anything that isn't flags-based. Once you do new system calls, doing the above is really quite nasty. How do you statically even test that you have a system call? Now you need to add a whole autoconf thing for it existing, and when it does exist you still need to test whether it works, and you can't even do it in the slow-path like the above (which turns the failure into a fast-path without the flag). 

This new approach, with a scaled down number of new system calls rather than adding a general-purpose system call extension mechanism like `sys_indirect()`, is now being pursued by Drepper. In the [explanatory patch at the start of the series](<https://lwn.net/Articles/282322/>), he lays out which of the system calls will require a new user space interface: `paccept()`, `epoll_create2()`, `dup3()`, `pipe2()`, and `inotify_init1()`, as well as those that do not: `signalfd4()`, `eventfd2()`, `timerfd_create()`, `socket()`, and `socketpair()`. 

Drepper has already made several iterations of patches addressing most of the concerns expressed by the kernel developers along the way. There have been some architecture specific problems, but Drepper has been knocking those down as well. If no further roadblocks appear, it would seem a likely candidate for inclusion in 2.6.27.
