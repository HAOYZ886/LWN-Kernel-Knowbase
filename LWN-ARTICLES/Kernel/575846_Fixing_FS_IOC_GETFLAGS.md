---
title: Fixing FS_IOC_GETFLAGS
url: https://lwn.net/Articles/575846/
date: "December 11, 2013"
category: "Development model-User-space ABI; ioctl"
author: "By Jake Edge December 11, 2013"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
December 11, 2013

Making kernel interfaces that work for both 32- and 64-bit processors has proved to be something of a challenge over the years. One of the more problematic areas has been passing arguments to `ioctl()` so that the same code will work on both types of processor—in both big- and little-endian varieties. As a recent thread shows, not all of those problems have been completely worked out over time. 

Aurelien Jarno [posted a question](<https://lwn.net/Articles/576273/>) to the linux-fsdevel mailing list about `FS_IOC_GETFLAGS` and `FS_IOC_SETFLAGS` (which query and set inode flags on files). He noted that the definitions of those requests in `include/uapi/linux/fs.h` listed the argument types as a pointer to `long`, except for the 32-bit compatibility versions, which specify an `int *`. The code in the kernel filesystems expects and uses a 32-bit quantity, and most—but not all—user-space code passes a pointer to a 32-bit integer. 

Any application that passed a pointer to a 64-bit integer would work, but only on little-endian systems. Since the kernel code treats the pointer as one to a 32-bit quantity, it's a matter of which four bytes are accessed when the pointer is dereferenced. On big-endian processors, it is the most significant four bytes, whereas little-endian systems reference the least significant end. Since all of the flags live in the least significant four bytes, the big-endian systems effectively pass zero to `FS_IOC_SETFLAGS` or retrieve a value with (undefined) high bits set with `FS_IOC_GETFLAGS`. 

Darrick J. Wong [pointed out](<https://lwn.net/Articles/576284/>) that the kernel FUSE driver uses the types from that header, which also causes a problem. The kernel driver expects to transfer a 64-bit quantity, but most user-space programs only provide 32 bits. He plans to special case those `ioctl()` requests for FUSE. 

The number of big-endian 64-bit systems (e.g. PowerPC, MIPS, and s390) is dwarfed by the number of x86_64 little-endian processors. That means that few have seen the problem, but it also means that any fix needs to be made carefully to avoid breaking millions of existing systems. That is always an important—overriding—consideration for changes to the kernel, of course, but Ted Ts'o [highlighted](<https://lwn.net/Articles/576377/>) that concern when he explained a bit about how this had come about and why changing to a `long *` everywhere would not work. Because the majority of user-space programs pass an `int *`, a change like that would cause them to break on 64-bit systems regardless of the endian-ness. 

But, as Jarno [pointed out](<https://lwn.net/Articles/576378/>), anyone trying to do the right thing and look up the argument type in `<linux/fs.h>` will get it wrong. ""The bare minimum would be to add a comment close to the definition to explain to use an int and not a long."" It turns out that there are four `ioctl()` requests (`FS_IOC_GETVERSION` and `FS_IOC_SETVERSION` in addition to the get/set flags mentioned above) that have the problematic definition. Jarno [posted a patch](<https://lwn.net/Articles/576446/>) to make that change by adding a warning to `include/uapi/linux/fs.h` (which gets installed in `include/linux`): 

```
/*
     * WARNING: The next four following ioctls actually take an int argument
     * despite their definition. This is important to support 64-bit big-endian
     * machines.
     */
```

One might think that just changing the type of the argument to 32 bits in the header file would be a possibility, but that cannot be done either. The mapping of the request name to a number is done using a set of macros that use `sizeof()` on the "`type`" argument. For 64-bit systems that is eight for a `long`, but 32-bit uses four. Since the numbers calculated for the requests are now a fixed part of the Linux ABI, changing the type of the argument in that header would not solve the problem. 

Several suggested adding a new request type (`FS_IOC_GETFLAGS_NEW` or `FS_IOC_GETFLAGS_WIDE` for example). It would take a pointer to a 64-bit integer on _all_ architectures. That would have the advantage of doubling the number of available flags, which may be getting close to being consumed. There are perhaps ten bits available today for expansion, adding another 32 might cover any upcoming use cases, though some are rather skeptical that 32 will be enough. 

The `FS_IOC_[GS]ETFLAGS` requests were originally added for the ext* filesystems, but have also been used by other filesystems over time. In addition, there are flags for other filesystems that are only available via filesystem-specific `ioctl()` requests. According to Dave Chinner, [XFS already has roughly ten flags](<https://lwn.net/Articles/576460/>) available using a different request (`XFS_IOC_FSGETXATTR`); other filesystems have their own sets. So, if a change is going to be made, Chinner said, why not create one that unifies all of the disparate inode flag handling; one that allows for more than 64 flags that might be completely exhausted soon. 

Ts'o is [not convinced](<https://lwn.net/Articles/576461/>) that the additional complexity is worth it. But Chinner [sees](<https://lwn.net/Articles/576462/>) XFS adding ""tens of new inode flags"" over the coming years. Other filesystems may well have similar needs. So a fixed-length bitmap may not be the best solution long-term, but there was little agreement on which alternative should be pursued. 

Chinner suggested some kind of attribute-based interface that is open-ended so that it could be expanded to handle any inode flags for any filesystems down the road. He also mentioned the [`xstat()` system call](<https://lwn.net/Articles/394298/>) as another possibility. But, as Andreas Dilger [pointed out](<https://lwn.net/Articles/576466/>), `xstat()` has been proposed many times, but has never made it into the kernel. 

So there are some possible solutions that ""solve the problem once and for all"" (as Chinner put it), but it is not at all clear that anyone is planning to push for one of them. In the meantime, Jarno's "fix" to the header file will at least help users pass the right argument types. The user-space applications that pass `long` pointers (bup and libexplain were mentioned) will need to change, but that shouldn't be too onerous. A more ambitious, global solution may not be in the works anytime soon.
