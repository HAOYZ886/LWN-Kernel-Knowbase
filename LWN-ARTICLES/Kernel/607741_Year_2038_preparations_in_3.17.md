---
title: Year 2038 preparations in 3.17
url: https://lwn.net/Articles/607741/
date: "August 6, 2014"
category: Year 2038 problem
author: "By Jonathan Corbet August 6, 2014"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
August 6, 2014

January 19, 2038 is, of course, that fateful date when 32-bit `time_t` variables wrap around, bringing about the end of time as Unix-like systems know it. Though 2038 may seem far away, it is not too soon to be worrying about this problem; code needs to be able to work with future dates, and some systems being deployed now will certainly still be around 24 years from now. A full solution to the problem of keeping 32-bit systems working in 2038 is going to take a while to reach. But some of the first baby steps in that direction were taken during the 3.17 merge window. 

Much of the work involves changes to two structures used within the kernel: `union ktime` (usually referred to as `ktime_t`) and `struct timespec`. The `ktime_t` structure was introduced with the [high-resolution timer patch set](<https://lwn.net/Articles/167897/>) back in 2006; it is meant to be an opaque type for the storage of time values within the kernel. Indeed, it is sufficiently opaque that its definition varies widely depending on the underlying architecture. 

For 64-bit systems, `ktime_t` has always been a simple integer count of nanoseconds. This "scalar" format is easy to manipulate and perform arithmetic on — as long as the CPU has fast 64-bit operations. Since such operations tend to be absent on 32-bit systems, `ktime_t` has often been defined differently there; it is represented as a structure with separate, 32-bit fields for seconds and nanoseconds. Kernel code uses a set of accessor functions for working with `ktime_t` values, so the difference in representation is well hidden and does not affect how the rest of the kernel works. 

That difference will be rather less well hidden in 2038, though, when the 32-bit seconds field overflows with all the rest. So, for things to keep working in 2038, the `ktime_t` structure will have to change. One of the first changes merged for 3.17 is to simply get rid of the non-scalar form of `ktime_t` and force all architectures to use the 64-bit nanosecond count representation. This change may slow things down on 32-bit systems; in particular, conversions from other time formats may be significantly slower. But, as noted in [the changelog](<http://git.kernel.org/linus/24e4a8c3e8868874835b0f1ad6dd417341e99822>), the ARM and x86 architectures were already using the scalar format anyway, so they will not get any slower. 

Regardless of whether conversions between `ktime_t` and other formats are fast or not, avoidance of those conversions when possible seems like a promising way of optimizing code within the kernel. The 3.17 changes include a number of time-format changes within various kernel subsystems, causing them to just work with straight nanosecond time values. The result is generally a simplification of the code and, presumably, faster execution. 

The other time-related structure used heavily within the kernel is `struct timespec`: 

```
struct timespec {
    	__kernel_time_t	tv_sec;			/* seconds */
    	long		tv_nsec;		/* nanoseconds */
        };
```

The `__kernel_time_t` type is just another name for `time_t` in current kernels; it is, thus, a 32-bit value on 32-bit systems. Unlike `ktime_t`, though, `struct timespec` cannot just be changed; it is used in user space as well and is a part of the kernel's ABI. What has been done instead in 3.17 is to add a new structure: 

```
struct timespec64 {
    	time64_t	tv_sec;			/* seconds */
    	long		tv_nsec;		/* nanoseconds */
        };
```

On 64-bit systems, this structure is identical to `struct timespec`. Within the core timekeeping code, every `struct timespec` has been changed to be `struct timespec64` instead. The interfaces providing access to timekeeping functionality to the rest of the kernel have been tweaked to hide this change (so far), and a new set of interfaces has been added for code that is using `struct timespec64`. After this change, there are no more time values using 32-bit seconds counts in the timekeeping core. 

The result of all this work is a long way from a solution to the year-2038 problem. But it is one important step in that direction: the core timekeeping code within the Linux kernel will no longer have problems when 2038 rolls around. With a couple more steps, a full solution to the problem may well be in sight. The first of those steps is to push use of `struct timespec64` outward from the timekeeping core into the rest of the kernel. This task may involve a fair amount of work, but it is an example of the sort of evolutionary change that the kernel community is relatively good at. Given a bit of time, kernel code should be almost entirely free of year-2038 issues. 

The harder step, of course, is to incorporate year-2038-safe structures into the kernel ABI and get user-space developers to change their code over. That will require cooperation with user-space developers from the C-library level on up and a lot of thought into how this change can be made with a minimum of pain. One should not expect it to happen quickly. But the problem is now well established on the radar of many of the relevant developers, so the chances of resolving most of the problems without a last-minute panic seem to be reasonably good. The first steps have been taken; hopefully the rest will follow before too long.
