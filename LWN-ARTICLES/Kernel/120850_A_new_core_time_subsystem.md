---
title: A new core time subsystem
url: https://lwn.net/Articles/120850/
date: "January 26, 2005"
category: Timekeeping
author: 
---

Keeping track of the current time is one of the kernel's many jobs. In the Linux kernel, this task is handled in a very architecture-dependent way. Each architecture has its own sources of high-resolution time, and each performs its own calculations. This system works, but it results in quite a bit of code being duplicated across architectures, and it can be brittle. Patches which change time-related code often do not manage to correctly update all architectures. 

John Stultz has been working for some months on a cleaner alternative. The result is [a new time subsystem](<https://lwn.net/Articles/120588/>) which, he hopes, will improve the situation. 

Much of the patch can be seen as a refactoring of the time code. Common calculations are now performed in the timeofday core, rather than in architecture-specific code. The code for implementing the network time protocol (NTP), an interesting exercise in complexity itself, has been separated from the rest of the time code and hidden in its own file. Most of the core time code has been reworked to deal with time in nanoseconds, a format which gives adequate time resolution but which, in a 64-bit variable, is still good for centuries. The timeofday code no longer depends on the `jiffies` variable, meaning that it can work independently of the timer interrupt, which may be disabled in some situations. The overall result is kernel timing code which is much easier to read and understand. 

In the end, however, the timing code must go to the hardware to actually get high-resolution time values. John made a couple of observations here. One is that, while time sources are architecture-dependent, many architectures share the same types of timing hardware. The other was that the code which deals with a time source is really just another device driver. So he isolated the time source information into its own structure: 

```
struct timesource_t {
    		char* name;
    		int priority;
    		enum {
    			TIMESOURCE_FUNCTION,
    			TIMESOURCE_CYCLES,
    			TIMESOURCE_MMIO_32,
    			TIMESOURCE_MMIO_64
    		} type;
    		cycle_t (*read_fnct)(void);
    		void __iomem* mmio_ptr;
    		cycle_t mask;
    		u32 mult;
    		u32 shift;
    		void (*update_callback)(void);
    	};
```

Here, `name` is just a name for the source, and `priority` is used to choose between multiple available sources. The `type` field tells how this source can be read. If `type` is `TIMESOURCE_FUNCTION`, the `read_fnct()` will be called to read the source. The two `_MMIO_` variants are for hardware which can be read directly from I/O memory; in that case, the time code can just obtain a value from the location indicated by `mmio_ptr` with no need to call any outside functions. `TIMESOURCE_CYCLES` indicates that the processor's time stamp counter (TSC) is being used, so `get_cycles()` is called to get the actual value. In any of the above cases, the value returned by the time source is assumed to be some sort of counter. The `mask`, `mult`, and `shift` values are applied to turn a delta between two such values into a number of nanoseconds for the rest of the timekeeping code. 

With this structure in place, architecture-specific code need only fill in a `timesource_t` structure (possibly implementing a read function in the process) and pass it to `register_timesource()`. All the rest is then handled in the common code. John has provided [a set of time source drivers](<https://lwn.net/Articles/120590/>) for a few architectures which demonstrates how they can be written. 

The discussion of the patches suggests that, while developers like the general intent, there are some remaining concerns - especially among the architecture maintainers. In some architectures, the `gettimeofday()` system call can be handled entirely in user space, but the current patches do not yet support that. The current NTP implementation is also seen as being too expensive. Finding a way to cut the cost of NTP while maintaining accuracy could be a bit of a challenge, but John is working at it. Expect to see some more iterations on this one.
