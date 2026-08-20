---
title: The x32 system call ABI
url: https://lwn.net/Articles/456731/
date: "August 29, 2011"
category: "User-space API; x32"
author: "By Jonathan Corbet August 29, 2011"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
August 29, 2011

The 32-bit x86 architecture has a number of well-known shortcomings. Many of these were addressed when this architecture was extended to 64 bits by AMD, but running in 64-bit mode is not without problems either. For this reason, a group of GCC, kernel, and library developers has been working on a new machine model known as the "x32 ABI." This ABI is getting close to ready, but, as a recent discussion shows, wider exposure of x32 is bringing some new issues to the surface. 

Classic 32-bit x86 has easily-understood problems: it can only address 4GB of memory and its tiny set of registers slows things considerably. Running a current processor in the 64-bit mode fixes both of those problems nicely, but at a cost: expanding variables and pointers to 64 bits leads to expanded memory use and a larger cache footprint. It's also not uncommon (still) to find programs that simply do not work properly on a 64-bit system. Most programs do not actually need 64-bit variables or the ability to address massive amounts of memory; for that code, the larger data types are a cost without an associated benefit. It would be really nice if those programs could take advantage of the 64-bit architecture's additional registers and instructions without simultaneously paying the price of increased memory use. 

That best-of-both-worlds situation is exactly what the [x32 ABI](<https://sites.google.com/site/x32abi/>) is trying to provide. A program compiled to this ABI will run in native 64-bit mode, but with 32-bit pointers and data values. The full register set will be available, as will other advantages of the 64-bit architecture like the faster `SYSCALL64` instruction. If all goes according to plan, this ABI should be the fastest mode available on 64-bit machines for a wide range of programs; it is easy to see x32 widely displacing the 32-bit compatibility mode. 

One should note that the "if" above is still somewhat unproven: actual benchmarks showing the differences between x32 and the existing pure modes are hard to come by. 

One outstanding question - and [the spark for the current discussion](<https://lwn.net/Articles/456739/>) \- has to do with the system call ABI. For the most part, this ABI looks similar to what is used by the legacy 32-bit mode: the 32-bit-compatible versions of the system calls and associated data structures are used. But there is one difference: the x32 developers want to use the `SYSCALL64` instruction just like native 64-bit applications do for the performance benefits. That complicates things a bit, since, to know what data size to expect, the kernel needs to be able to distinguish system calls made by true 64-bit applications from those running in the x32 mode, regardless of the fact that the processor is running in the same mode in both cases. As an added challenge, this distinction needs to be made without slowing down native 64-bit applications. 

The solution involves using an expanded version of the 64-bit system call table. Many system calls can be called directly with no compatibility issues at all - a call to `fork()` needs little in the translation of data structures. Others do need the compatibility layer, though. Each of those system calls (92 of them) is assigned a new number starting at 512\. That leaves a gap above the native system calls for additions over time. Bit 30 in the system call number is also set whenever an x32 binary calls into the kernel; that enables kernel code that cares to implement "compatibility mode" behavior. 

Linus didn't seem to mind the mechanism used to distinguish x32 system calls in general, but he [hated the use of compatibility mode](<https://lwn.net/Articles/456741/>) for the x32 ABI. He asked: 

I think the real question is "why?". I think we're missing a lot of background for why we'd want yet another set of system calls at all, and why we'd want another state flag. Why can't the x32 code just use the native 64-bit system calls entirely? 

There are legitimate reasons why some of the system calls cannot be shared between the x32 and 64-bit modes. Situations where user space passes structures containing pointers to the kernel (`ioctl()` and `readv()` being simple examples) will require special handling since those pointers will be 32-bit. Signal handling will always be special. Many of the other system calls done specially for x32, though, are there to minimize the differences between x32 and the legacy 32-bit mode. And those calls are the ones that Linus [objects to](<https://lwn.net/Articles/456745/>) most strongly. 

It comes down, for the most part, to the format of integer values passed to the kernel in structures. The legacy 32-bit mode, naturally, uses 32-bit values in most cases; the x32 mode follows that lead. Linus is saying, though, that the 64-bit versions of the structures - with 64-bit integer values - should be used instead. At a minimum, doing things that way would minimize the differences between the x32 and native 64-bit modes. But there is also a correctness issue involved. 

One place where the 32- and 64-bit modes differ is in their representation of time values; in the 32-bit world, types like `time_t`, `struct timespec`, and `struct timeval` are 32-bit quantities. And 32-bit time values will overflow in the year 2038. If the year-2000 issue showed anything, it's that long-term drop-dead days arrive sooner than one tends to think. So it's not surprising that Linus is [unwilling](<https://lwn.net/Articles/456750/>) to add a new ABI that would suffer from the 2038 issue: 

2038 is a long time away for legacy binaries. It's *not* all that long away if you are introducing a new 32-bit mode for performance. 

The width of `time_t` cannot change for legacy 32-bit binaries. But x32 is an entirely new ABI with no legacy users at all; it does not have to retain any sort of past compatibility at this point. Now is the only time that this kind of issue can be fixed. So it is probably entirely safe to say that an x32 ABI will not make it into the mainline as long as it has problems like the year-2038 bug. 

At this point, the x32 developers need to review their proposed system call ABI and find a way to rework it into something closer to Linus's taste; that process is [already underway](<https://lwn.net/Articles/456760/>). Then developers can get into the serious business of building systems under that ABI and running benchmarks to see whether it is all worth the effort. Convincing distributors (other than Gentoo, of course) to support this ABI will take a fairly convincing story, but, if this mode lives up to its potential, that story might just be there.
