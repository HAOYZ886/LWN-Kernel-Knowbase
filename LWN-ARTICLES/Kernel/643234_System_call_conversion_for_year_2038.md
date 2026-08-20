---
title: System call conversion for year 2038
url: https://lwn.net/Articles/643234/
date: "May 5, 2015"
category: Year 2038 problem
author: "By Jonathan Corbet May 5, 2015"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
May 5, 2015

There are now less than 23 years remaining until that fateful day in January 2038 when signed 32-bit `time_t` values — used to represent time values in Unix-like systems — run out of bits and overflow. As that date approaches, 32-bit systems can be expected to fail in all kinds of entertaining ways and current LWN readers can look forward to being called out of retirement in a heroic (and lucrative) effort to stave off the approaching apocalypse. Or that would be the case if it weren't for a group of spoilsport developers who are trying to solve the year-2038 problem now and ruin the whole thing. The shape of that effort has come a bit more into focus with the posting by Arnd Bergmann of [a new patch set](<https://lwn.net/Articles/643148/>) (later [updated](<https://lwn.net/Articles/643407/>)) showing the expected migration path for time-related system calls. 

Current Linux system calls use a number of different data types to represent times, from the simple `time_t` value through the `timeval` and `timespec` structures and others. Each, though, has one thing in common: an integer value counting the number of seconds since the beginning of 1970 (or from the current time in places where a relative time value is needed). On 32-bit systems, that count is a signed 32-bit value; it clearly needs to gain more bits to function in a world where post-2038 dates need to be represented. 

#### Time representations

One possibility is to simply create 64-bit versions of these time-related structures and use them. But if an incompatible change is to be made, it might be worthwhile thinking a bit more broadly; to that end, Thomas Gleixner recently [suggested](<https://lwn.net/Articles/643237/>) the creation of a new set of (Linux-specific) system calls that would use a signed, 64-bit nanosecond counter instead. This counter would mirror the `ktime_t` type (defined in `<include/linux/ktime.h>`) used to represent times within the kernel: 

```
union ktime {
    	s64	tv64;
        };
        typedef union ktime ktime_t;		/* Kill this */
```

(Incidentally, the "kill this" comment was [added by Andrew Morton](<http://git.kernel.org/linus/f34c506b0385b43abd25c490335036ecbb173aed>) in 2007; nobody has killed it yet.) 

Having user space work with values that mirror those used within the kernel has a certain appeal; a lot of time-conversion operations could be eliminated. But Arnd Bergmann [pointed out](<https://lwn.net/Articles/643239/>) a number of difficulties with this approach, including the fact that it makes a complicated changeover even more so. The fatal flaw, though, turns up in [this survey of time-related system calls](<https://lwn.net/Articles/643240/>) posted by Arnd shortly thereafter: system calls that deal with file timestamps need to be able to represent times prior to 1970. They also need to be able to express a wider range of times than is possible with a 64-bit `ktime_t`. So some variant of `time_t` must be used with them, at least. (The need to represent times before 1970 also precludes the use of an unsigned value to extend the forward range of a 32-bit `time_t` value). 

So universal use of signed nanosecond time values does not appear to be in the cards, at least not as part of the year-2038 disaster-prevention effort. Still, there is room for some simplification. The current plan is to use the 64-bit version of `struct timespec` (called, appropriately, `struct timespec64` in the kernel, though user space will still see it as simply `struct timespec`) for almost all time values passed into or out of the kernel. The various system calls that use the other (older) time formats can generally be emulated in user space. So, for example, a call to `gettimeofday()` (which uses `struct timeval`) will be turned into a call to `clock_gettime()` before entry into the kernel. That reduces the number of system calls for which compatibility must be handled in kernel space. 

Thus, in the future, a 32-bit system that is prepared to survive 2038 will use `struct timespec64` for all time values exchanged with the kernel. That just leaves the minor problem of how to get there with a minimal amount of application breakage. The current plan can be seen in Arnd's patch set, which includes a number of steps to move the kernel closer to a year-2038-safe mode of operation. 

#### Getting to a year-2038-safe system

The first of those steps is to prepare to support 32-bit applications while moving the kernel's internal time-handling code to 64-bit times in all situations. The internal kernel work has been [underway](<https://lwn.net/Articles/607741/>) for a while, but the user-space interfaces still need work, starting with the implementation of a set of routines that will convert between 32-bit and 64-bit values at the system-call boundary. The good news is that these routines already exist in the form of the "compatibility" system calls used by 32-bit applications running on a 64-bit kernel. In the future, all kernels will be 64-bit when it comes to time handling, so the compatibility functions are just what is needed (modulo a few spots where other data types must be converted differently). So the patch set causes the compatibility system calls to be built into 32-bit kernels as well as 64-bit kernels. These compatibility functions are ready for use, but will not be wired up until the end of the patch series. 

The next step is the conversion of the kernel's native time-handling system calls to use 64-bit values exclusively. This process is done in two broad sub-steps, the first of which is to define a new set of types describing the format of native time values in user space. For example, system calls that currently accept `struct timespec` as a parameter will be changed to take `struct __kernel_timespec` instead. By default, the two structures are (nearly) the same, so the change has no effect on the built kernel. If the new `CONFIG_COMPAT_TIME` configuration symbol is set, though, `struct __kernel_timespec` will look like `struct timespec64` instead. 

The various `__kernel_` types are used at the system-call boundary, but not much beyond that point. Instead, they are immediately converted to 64-bit types on all machines; on 64-bit machines, obviously, there is little conversion to do. Once each of the time-related system calls is converted in this manner, it will use 64-bit time values internally, even if user space is still dealing in 32-bit time values. Any time values returned to user space are converted back to the `__kernel_` form before the system call returns. There is still no change visible to user space, though. 

The final step is to enable the use of 64-bit time values on 32-bit systems without breaking existing 32-bit binaries. There are three things that must all be done together to make that happen: 

  * The `CONFIG_COMPAT_TIME` symbol is set, causing all of the `__kernel_` data structures to switch to their 64-bit versions. 

  * All of the existing time-related system calls are replaced with the 32-bit compatibility versions. So, for example, on the ARM architecture, `clock_gettime()` is system call number 263\. After this change, applications invoking system call 263 will get `compat_sys_clock_gettime()` instead. If the compatibility functions have been done correctly, binary applications will not notice the change. 

  * The native 64-bit versions of the system calls are given new system call numbers; `clock_gettime()` becomes system call 388, for example. Thus, only newly compiled code that is prepared to deal with 64-bit time values will see the 64-bit versions of these calls. 

And that is about as far as the kernel can take things. Existing 32-bit binaries will call the compatibility versions of the time-related system calls and will continue to work — until 2038 comes around, of course. 

That leaves a fair amount of work to be done in user space, of course. In a simplified view of the situation, the C libraries can be changed to use the 64-bit data structures and invoke the new versions of the relevant system calls. Applications can then be recompiled against the new library, perhaps with some user-space fixes required as well; after that, they will no longer participate in the year 2038 debacle. In practice, all of the libraries in a system and all applications may need to be rebuilt together to ensure that they have a coherent idea of how times are represented. The GNU C library uses symbol versioning, so it can be made to work with both time formats simultaneously, but many other libraries lack that flexibility. So converting a full distribution is likely to be an interesting challenge even once the work on the kernel side is complete. 

#### Finishing the job

Even on the kernel side, though, there are a few pieces of the puzzle that have not yet been addressed. One significant problem is `ioctl()` calls; of the thousands of them supported by the kernel, a few deal in `time_t` values. They will have to be located and fixed one-by-one, a process that could take some time. The ext4 filesystem [stores timestamps as 32-bit `time_t` values](<https://ext4.wiki.kernel.org/index.php/Ext4_Disk_Layout#Inode_Timestamps>), though some variants of the on-disk format extend those fields to 34 bits. Ext3 does not support 34-bit timestamps, though, so the solution there is likely to be to drop it entirely in favor of ext4. NFSv3 has a similar problem, and may meet a similar fate; XFS also has [some challenges to deal with](<https://lkml.org/lkml/2014/6/1/240>). The filesystem issues, notably, affect 64-bit systems as well. There are, undoubtedly, many other surprises like this lurking in both the kernel and user space, so the task of making a system ready for 2038 goes well beyond migrating to 64-bit time values in system calls. Still, fixing the system calls is a start. 

Once the remaining problems have been addressed, there is a final patch that can be applied. It makes `CONFIG_COMPAT_TIME` optional, but in a way that leaves the 64-bit paths in place while removing the 32-bit compatibility system calls. If this option is turned off, any binary using the older system calls will fail to run. This is thus a useful setting for testing year-2038 conversions or deploying long-lived systems that must survive past that date. As Arnd put it: 

This is meant mostly as a debugging help for now, to let people build a y2038 safe distro, but at some point in the 2030s, we should remove that option and all the compat handling. 

Presumably somebody will be paying attention and will remember to carry out this removal twenty years from now (if they are feeling truly inspired, they might just kill `ktime_t` while they are at it). At that point, they will likely be grateful to the developers who put their time into dealing with this problem before it became an outright emergency. The rest of us, instead, will just have to find some other way to fund our retirement. 

(Thanks to Arnd Bergmann for his helpful comments and suggestions on an earlier draft of this article.)
