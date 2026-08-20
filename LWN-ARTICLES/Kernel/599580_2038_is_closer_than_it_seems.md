---
title: 2038 is closer than it seems
url: https://lwn.net/Articles/599580/
date: "May 21, 2014"
category: Timekeeping; Year 2038 problem
author: "By Jonathan Corbet May 21, 2014"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 21, 2014

Most LWN readers are likely aware of the doom impending upon us in January 2038, when the `time_t` type used to store time values (in the form of seconds since January 1, 1970) runs out of bits on 32-bit systems. It may be surprising that developers are increasingly worried about this deadline, which is still nearly 24 years in the future, but there are good reasons to be concerned now. Whether those worries will lead to a solution in the near term remains to be seen; not much has happened since [this topic came up](<https://lwn.net/Articles/563285/>) last August. But recent discussions have at least shed a bit of light on the forms such a solution might take. 

At times, developers have hoped that this problem might solve itself. On 64-bit systems, the `time_t` type has always been defined as a 64-bit quantity and will not run out of space anytime soon. Given that 64-bit systems appear to be taking over the world — even phone handsets seem likely to make the switch in the next few years — might the best solution be to just wait for 32-bit systems to die out and take the problem with them? A "no action required" solution has an obvious appeal. 

There are two problems with that reasoning: (1) 32-bit systems are likely to continue to be made for far longer than most people might expect, and (2) there are 32-bit systems being deployed now that can be expected to have lifetimes of 24 years or longer. 32-bit systems will be useful as cheap microcontrollers for a long time, and, once deployed, they will often be expected to work for many years while being difficult or impossible to update. There are almost certainly systems already deployed that are going to provide unpleasant surprises in 2038. 

#### Kernel-based solutions

So it would appear to make sense to solve the problem soon, rather than in, say, 2036 or so. There is only one snag: the problem is not all that easy to solve. At least, it is not easy if one is concerned about little details like not breaking existing programs. Since Linux developers at most levels are quite concerned about compatibility, the simplest solutions (such as a BSD-style ABI break) are not seen as being workable. In a recent discussion, John Stultz [outlined a couple of alternative approaches](<https://lwn.net/Articles/599582/>), neither of which is without its difficulties. 

The first approach would be to change the 32-bit ABI to use a 64-bit version of `time_t` (related data structures like, `struct timespec` and `struct timeval` would also change). Old binaries could be supported through a compatibility interface, but newly compiled code would normally use the new ABI. There are some advantages to this approach, starting with the fact that lots of applications could be updated simply by rebuilding them. Since a couple of BSD variants have already taken this path, a number of the worst application problems have already been fixed. Embedded microcontrollers typically run custom distributions built entirely from source; changing the ABI in this way would make it possible to build 2038-capable systems in the near future with a minimum of pain. 

On the other hand, the kernel would have to maintain a significant compatibility layer for a long time. Developers are also worried that there will be many applications that store 32-bit `time_t` values in their own data structures, in on-disk formats, and more. Many of these applications could break in surprising ways, and they could prove to be difficult to fix. There are also some concerns about the runtime cost of using 64-bit `time_t` values on 32-bit systems. Much of this cost could be mitigated within the kernel by using a different format internally, but applications could slow down as well. 

The alternative approach is to simply define a new set of system calls, all of which are defined to use better time formats from the beginning. The new formats could address other irritations at the same time; not everybody likes the separate seconds and nanoseconds fields used in `struct timespec`, for example. All system calls defined to use the old `time_t` values would be deprecated, with the idea of removing them, if possible, before 2038. 

With this approach, there would be no hard ABI break anytime soon and applications could be migrated gradually. Once again, embedded systems could be built using the new system calls in the relatively near future, while desktop systems could be left alone for another decade or so. And it would be a chance to start over and redesign some longstanding system calls with 21st-century needs in mind. 

Defining new system calls has its downsides as well, though. It would push Linux further away from being a POSIX system, and would take us down a path different from the one chosen by the BSD world. There are a _lot_ of system calls to replace, and `time_t` values show up in other places as well, most notably in a long list of `ioctl()` calls. Applications would have to be updated, including those running only on 64-bit systems, which would not see much of a benefit from the new system calls. And, undoubtedly, there would be lots of applications using the older system calls that would surface in 2037. So this approach is not an easy solution either. 

#### Including glibc

Discussions of these alternatives went on for a surprisingly long time before Christoph Hellwig made an (in retrospect) obvious [suggestion](<https://lwn.net/Articles/599583/>): the C library developers are going to have to be involved in the implementation of any real solution to the year-2038 problem, so perhaps they should be part of the discussion now. For years, communications between the kernel community and the developers of C libraries (including the GNU C library — glibc) have been sporadic at best. The changing of the guard at glibc has made productive conversations easier to have, but changing old habits has proved hard. In any case, it is true that the glibc developers will have to be involved in the design of the solution to this problem; the good news is that such involvement appears likely to happen. 

Glibc developers are not known for their love of ABI breaks — or of non-POSIX interfaces for that matter. So, once glibc developer Joseph Myers [joined the conversation](<https://lwn.net/Articles/599591/>), the tone shifted a bit toward a solution that would allow a smooth transition while retaining existing POSIX system calls and application compatibility. The plan (which was discussed only in rough form and would need a lot of work yet) looks something like this: 

  * Create new, 64-bit versions of the affected system calls. So, for example, there would be a `gettimeofday64()` that returns the time in a `struct timeval64`. The existing versions of these system calls would be unchanged. 

  * Glibc would gain a new feature test macro with a name like `TIME_BITS`. If `TIME_BITS=64` on a 32-bit system, a call to `gettimeofday()` will be remapped to `gettimeofday64()` within the library. So applications can opt into the new world by building with an appropriate value of `TIME_BITS` defined. 

  * Eventually, `TIME_BITS=64` would become the default, probably after distributions had been shipping in that mode for a while. Even in the 64-bit configuration, compatibility symbols would remain so that older binaries would still work against newer versions of the C library. 

Such an approach could possibly allow for a relatively smooth transition to a system that will work in 2038, though, naturally, a number of troublesome details remain. There was talk of remapping `ioctl()` calls in a similar way, but that looks like a recipe for trouble given just how many of those calls there are and how hard it would be to even find them all. Developers in other C library projects, who often don't wish to maintain the sort of extensive compatibility infrastructure found in glibc, may wish to take a different approach. And so on. 

But, even with its challenges, the existence of a vague plan hashed out with participation from kernel and glibc developers is reason for hope. Maybe, just maybe, some sort of reasonably robust solution to the 2038 problem will be found before it becomes absolutely urgent, and, with luck, before lots of systems that will need to function properly in 2038 are deployed. We have the opportunity to avoid a year-2038 panic at a relatively low cost; if we make use of that opportunity, our future selves will thank us.
