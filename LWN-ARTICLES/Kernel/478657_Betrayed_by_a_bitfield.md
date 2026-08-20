---
title: Betrayed by a bitfield
url: https://lwn.net/Articles/478657/
date: "February 1, 2012"
category: GCC
author: "By Jonathan Corbet February 1, 2012"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
February 1, 2012

Developers tend to fear compiler bugs, and for good reason: such bugs can be hard to find and hard to work around. They can leave traps in a compiled program that spring on users at bad times. Things can get even worse if one person's compiler bug is seen by the compiler's developer as a feature - such issues have a tendency to never get fixed. It is possible that just this kind of feature has turned up in GCC, with unknown impact on the kernel. 

One of the many structures used by the btrfs filesystem, defined in `fs/btrfs/ctree.h`, is: 

```
struct btrfs_block_rsv {
    	u64 size;
    	u64 reserved;
    	struct btrfs_space_info *space_info;
    	spinlock_t lock;
    	unsigned int full:1;
        };
```

Jan Kara recently [reported](<https://lwn.net/Articles/478658/>) that, on the ia64 architecture, the `lock` field was occasionally becoming corrupted. Some investigation revealed that GCC was doing a surprising thing when the bitfield `full` is changed: it generates a 64-bit read-modify-write cycle that reads both `lock` and `full`, modifies `full`, then writes both fields back to memory. If `lock` had been modified by another processor during this operation, that modification will be lost when `lock` is written back. The chances of good things resulting from this sequence of events are quite small. 

One can imagine that quite a bit of work was required to track down this particular surprise. It is also not hard to imagine the dismay that results from a conversation like this: 

I've raised the issue with our GCC guys and they said to me that: "C does not provide such guarantee, nor can you reliably lock different structure fields with different locks if they share naturally aligned word-size memory regions. The C++11 memory model would guarantee this, but that's not implemented nor do you build the kernel with a C++11 compiler." 

Unsurprisingly, Linus was [less than impressed](<https://lwn.net/Articles/478667/>) by this response. Language standards are not written for the unique needs of kernels, he said, and can never "guarantee" the behavior that a kernel needs: 

So C/gcc has never "promised" anything in that sense, and we've always had to make assumptions about what is reasonable code generation. Most of the time, our assumptions are correct, simply because it would be *stupid* for a C compiler to do anything but what we assume it does. 

But sometimes compilers do stupid things. Using 8-byte accesses to a 4-byte entity is *stupid*, when it's not even faster, and when the base type has been specified to be 4 bytes! 

As it happens, the problem is a bit worse than non-specified behavior. Linus [suggested](<https://lwn.net/Articles/478672/>) running a test with a structure like: 

```
struct example {
    	volatile int a;
          	int b:1;
        };
```

In this case, if an assignment to `b` causes a write to `a`, the behavior is clearly buggy: the `volatile` keyword makes it explicit that `a` may be accessed from elsewhere. Jiri Kosina [gave it a try](<https://lwn.net/Articles/478669/>) and reported that GCC is still generating 64-bit operations in this case. So, while the original problem is technically compliant behavior, it almost certainly results from the same decision-making that makes the second example go wrong. 

Knowing that may give the kernel community more ammunition to flame the GCC developers with, but it is not necessarily all that helpful. Regardless of the source of the problem, this behavior exists in versions of the compiler that, almost certainly, are being used outside of the development community to build the kernel. So some sort of workaround is likely to be necessary even if GCC's behavior is fixed. That could be a bit of a challenge; auditing the entire kernel for 32-bit-wide bitfield variables in structures that may be accessed concurrently will not be a small job. But, then, nobody said that kernel development was easy.
