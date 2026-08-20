---
title: unsafe_put_user() turns out to be unsafe
url: https://lwn.net/Articles/736348/
date: "October 13, 2017"
category: "Security-Vulnerabilities"
author: "By Jonathan Corbet October 13, 2017"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
October 13, 2017

When a veteran kernel developer introduces a severe security hole into the kernel, it can be instructive to look at how the vulnerability came about. Among other things, it can point the finger at an API that lends itself toward the creation of such problems. And, as it turns out, the knowledge that the API is dangerous at the outset and marking it as such may not be enough to prevent problems. 

Back in late 2015, kernel developers were looking for ways to speed up the movement of data between user space and kernel space. Accessing user data from the kernel, whether for reading or writing, has clear security implications, so the kernel must ensure that the requested access is something that the requester is allowed to do. That involves checking that the intended area of memory is indeed accessible; on newer hardware, it can also require temporarily disabling mechanisms like [supervisor mode access prevention (SMAP)](<https://lwn.net/Articles/517475/>). Those preparations are expensive relative to a simple data copy, and they can add up if the kernel must perform a sequence of several reads and/or writes to user space. 

The obvious solution is to do the preparations once for a series of operations, then perform the operations themselves directly; that amortizes the cost of a single set of checks across multiple accesses. To that end, Linus Torvalds [added a new set of access functions](<https://git.kernel.org/linus/5b24a7a2aa2040c8c50c3b71122901d01661ff78>) for the 4.5 kernel: 

```
unsafe_get_user(value, source);
        unsafe_put_user(value, destination);
```

These "functions" are actually macros that expand differently depending on the type of `source` or `destination`. If, for example, `source` is a pointer to a `u16` value, then `unsafe_get_user()` will fetch an unsigned 16-bit value from that location and store it in `value`. These functions are thus like the traditional `get_user()` and `put_user()` macros, but with one change: they dispense with the normal permission checking. 

The lack of checks is the clear motivation for the "unsafe" naming, even if, according to [the comments in the code](<http://elixir.free-electrons.com/linux/latest/source/arch/x86/include/asm/uaccess.h#L689>), that term does not really apply: 

```
/*
         * The "unsafe" user accesses aren't really "unsafe", but the naming
         * is a big fat warning: you have to not only do the access_ok()
         * checking before using them, but you have to surround them with the
         * user_access_begin/end() pair.
         */
```

The required `access_ok()` call ensures that the pointer involved refers to a user-space address; it is there to prevent user space from asking the kernel to overwrite itself. The `user_access_begin()` and `user_access_end()` calls, instead, disable and enable SMAP. The whole idea behind the "unsafe" functions is that `access_ok()` and (especially) `user_access_begin()` can be called once before several user-space accesses, thus speeding up the code overall. 

During the 4.13 merge window, Al Viro [reworked](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4c48abe91be03d191d0c20cc755877da2cb35622>) the implementation of the [`waitid()` system call](<http://man7.org/linux/man-pages/man2/waitpid.2.html>) to use the "unsafe" functions. This variant of `wait()` requests that the kernel fill in a `siginfo_t` structure in user space with information on how the waited-for process died; doing so requires writing several values to user space from the kernel. The new code switched to `unsafe_put_user()` to speed these writes. Viro duly called `user_access_begin()` and `user_access_end()`, but left out the `access_ok()` calls. As a result, a call to `waitid()` could ask that the `siginfo_t` structure be stored in kernel space, overwriting a piece of kernel memory. Local attackers tend to be most pleased by this sort of unintended functionality. 

The result is deemed CVE-2017-5123; it only affects the 4.13 and 4.14-rc kernels. It is fixed by [this commit](<https://git.kernel.org/linus/96ca579a1ecc943b75beba58bebb0356f6cc4b51>) in the mainline, and will be fixed in the [4.13.7 stable update](<https://lwn.net/Articles/736354/>). The good news is that it was caught early enough that the number of machines that are exposed to the problem should be quite small. 

The bad news, of course, is that it happened at all. It turns out that this was [the first use of `unsafe_put_user()`](<https://lwn.net/Articles/736353/>) in the mainline kernel; this use was written by a developer who understands the issues involved, and he still got it wrong. Even better, the patch tickled a bug in that until-then unused function, with the result that Torvalds looked at the specific patch in question, but the problem still was not noticed. It would seem that, contrary to the comments, the "unsafe" functions are well named indeed, and that truly unsafe use does not jump out at people looking at the code — even if the functions proclaim themselves to be unsafe. 

Hopefully reviewers in the future will be better tuned into the fact that there are two necessary preconditions to the use of those functions. Even better would be some code that runs in kernels configured for debugging that would detect a lack of checking and raise the alarm. It might well be possible to apply some sort of static checking to the problem as well. Without such assistance, it seems likely that this type of bug will find its way into the kernel again in the future; it may not be so quickly detected next time.
