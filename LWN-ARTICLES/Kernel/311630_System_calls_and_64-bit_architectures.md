---
title: "System calls and 64-bit architectures"
url: https://lwn.net/Articles/311630/
date: "December 17, 2008"
category: "Architectures; User-space API"
author: "By Jake Edge December 17, 2008"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
December 17, 2008

Adding a system call to the kernel is never done lightly. It is important to get it right before it gets merged because, once that happens, it must be maintained as part of the kernel's binary interface forever. The proposal to [add `preadv()` and `pwritev()` system calls](<http://lwn.net/Articles/311092/>) provides an excellent example of the kinds of concerns that need to be addressed when adding to the kernel ABI. 

The two system calls themselves are quite straightforward. Essentially, they combine the existing `pread()` and `readv()` calls (along with the write variants of course) into a way to do scatter/gather I/O at a particular offset in the file. Like `pread()`, the current file position is unaffected. The calls, which are available on various BSD systems, can be used to avoid races between an `lseek()` call and a read or write. Currently, applications must do some kind of locking to prevent multiple threads from stepping on each other when doing this kind of I/O. 

The prototypes for the functions look much like readv/writev, simply adding the offset as the final parameter: 

```
ssize_t preadv(int d, const struct iovec *iov, int iovcnt, off_t offset);
        ssize_t pwritev(int d, const struct iovec *iov, int iovcnt, off_t offset);
```

But, because `off_t` is a 64-bit quantity, this causes problems on some architectures due to the way system call arguments are passed. After Gerd Hoffmann posted [version 2 of the patchset](<https://lwn.net/Articles/311729/>), Matthew Wilcox was quick to [point out](<https://lwn.net/Articles/311730/>) a problem: 

Are these prototypes required? MIPS and PARISC will need wrappers to fix them if they are. These two architectures have an ABI which requires 64-bit arguments to be passed in aligned pairs of registers, but glibc doesn't know that (and given the existence of syscall(3), can't do much about it even if it knew), so some of the arguments end up in the wrong registers. 

Several other architectures (ARM, PowerPC, s390, ...) have similar constraints. Because the offset is the fourth argument, it gets placed in the r3 and r4 32-bit registers, but some architectures need it in either r2/r3 or r4/r5. This led some to advocate reordering the parameters, putting the offset before `iovcnt` to avoid the problem. As long as that change doesn't bubble out to user space, Hoffmann is [amenable to making the change](<https://lwn.net/Articles/311731/>): ""I'd *really* hate it to have the same system call with different argument ordering on different systems though"". 

Most seemed to agree that the user-space interface as presented by glibc should match what the BSDs provide. It causes too many headaches for folks trying to write standards or portable code otherwise. To fix the alignment problem, the system call itself has the reordered version of the arguments. That led to Hoffmann's [third version of the patchset](<https://lwn.net/Articles/311734/>), which still didn't solve the whole problem. 

There are multiple architectures that have both 32 and 64-bit versions and the 64-bit kernel must support system calls from 32-bit user-space programs. Those programs will put 64-bit arguments into two registers, but the 64-bit kernel will expect that argument in a single register. Because of this, Arnd Bergmann [recommended splitting the offset into two arguments](<https://lwn.net/Articles/311736/>), one for the high 32 bits and one for the low: ""This is the only way I can see that lets us use a shared compat_sys_preadv/pwritev across all 64 bit architectures"". 

When a 32-bit user-space program makes a system call on a 64-bit system, the `compat_sys_*` version is used to handle differences in the data sizes. If a pointer to a structure is passed to a system call, and that structure has a different representation in 32-bits than it does in 64-bits, the `compat` layer makes the translation. Because different 64-bit architectures do things differently in terms of calling conventions and alignment requirements, the only way to share `compat` code is to remove the 64-bit quantity from the system call interface entirely. 

That just leaves one final problem to overcome: endian-ness. As Ralf Baechle [notes](<https://lwn.net/Articles/311737/>), MIPS can be either little or big-endian, so the `compat_sys_preadv/pwritev()` needs to put the two 32-bit offset values together in the proper way. He recommended moving the MIPS-specific `merge_64()` macro into a common `compat.h` include file, which could then be used by the common `compat` routines. So far, version 4 of the patchset has not emerged, but one suspects that the offset argument splitting and use of `merge_64()` will be part of it. 

The implementation of the operation of `preadv()` and `pwritev()` is very obvious, certainly in comparison to the intricacies of passing its arguments. The VFS implementations of `readv()`/`writev()` already take an offset argument, so it was simply a matter of calling those. It is interesting to note that as part of the review, Christoph Hellwig [spotted a bug](<https://lwn.net/Articles/311738/>) in the existing `compat_sys_readv/writev()` implementations which would lead to accounting information not being updated for those calls. 

This is not the first time these system calls have been proposed; way back in 2005, we [looked at some patches from Badari Pulavarty](<http://lwn.net/Articles/164887/>) that added them. Other than a brief appearance in the -mm tree, they seem to have faded away. Even if this edition of `preadv()` and `pwritev()` do not make it into the mainline--so far there are no indications that they won't--the code review surrounding it was certainly useful. Getting a glimpse of the complexities around 64-bit quantities being passed to system calls was quite informative as well.
