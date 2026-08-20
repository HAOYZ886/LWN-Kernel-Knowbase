---
title: "Boot-time clock frequency selection"
url: https://lwn.net/Articles/115554/
date: "December 14, 2004"
category: HZ; Latency; Timer frequency
author: 
---

The timer interrupt is the kernel's way of keeping track of the passage of time. Every so often, a programmable timer interrupts the kernel, which responds by updating its internal time value, performing various housekeeping tasks, and executing any delayed kernel work whose time has come. In the 2.6 kernel, on the x86 architecture, by default, the timer interrupt comes 1000 times per second; other architectures and configurations can vary. 

Playing with the timer tick frequency is almost as old as the kernel itself. The frequency with which the hardware timer interrupts the processor is well parameterized into a single compile-time variable (`HZ`); running the system with a nonstandard clock frequency is simply a matter of changing the definition of `HZ` (within reasonable bounds) and building a new kernel. 

There are legitimate reasons for playing with the timer frequency. A faster clock can allow the system to perform more precise delays, and to respond to events more quickly. Systems running at a higher clock frequency should have lower latencies in many situations. There is an overhead associated with the timer interrupt, however; a higher-frequency interrupt will take more CPU time. So, for server loads (where latency is less important), the overhead of a higher timer frequency is not worth it. On laptops, the default 1KHz timer can also defeat the CPU's power management features and significantly reduce battery life. 

In other words, there is no single value for the timer frequency which works for all users. Changing the frequency is still relatively hard, however; some people are more comfortable with building new kernels than others. Wouldn't it be nice if the frequency could be made into a boot-time parameter, so that it could be changed from one boot to the next without a kernel rebuild? 

As it turns out, Andrea Arcangeli has [a patch](<https://lwn.net/Articles/115285/>) which does exactly that. It's not even a new patch: SUSE has been shipping 2.4 kernels with boot-time timer frequency selection for some time. Andrea is now interested in merging this patch into the mainline, should the other developers be willing. 

The patch is relatively intrusive - it touches 143 files around the tree. The core change is the transformation of `HZ` from a constant value into a variable. Much of the kernel does not notice the change at all; a call like: 

```
schedule_timeout(HZ/10);
```

will still set up a wakeup for 100ms in the future. There is some new overhead associated with fetching the value of `HZ` and performing the division at run time, but Andrea states that it is not really measurable. 

There are places in the kernel which require further changes, however. Compile-time initializations which depend on a constant `HZ` value will no longer work; those initializations must be moved to run time, or recast in terms of a known constant value. There are also places where values in timer-tick units are provided by user space. The kernel tries to hide its internal clock frequency from user space, but there are still places where it leaks through. A number of boot-time parameters are expressed in ticks, and some device drivers take parameters in ticks as well. 

To address these problems, Andrea's expands the use of a symbol called `USER_HZ`. It is a constant value, though its actual definition is architecture dependent, varying from 32 to 1200 - though most architectures set it to 100. All remaining compile-time initializations, and all values obtained from user space, are interpreted as being in `USER_HZ` and must be translated to internal values before being used. To that end, some new macros have been provided: 

```
jiffies_to_clock_t(internal_hz);
    	user_to_kernel_hz(user_hz);
```

With these in place, it's just a matter of keeping track of which type of clock value is being used where. Andrea's patch renames variables containing user-space tick values (it prepends "`__`" to the name) as a way of indicating that a special value is contained there. 

Andrew Morton has [said](<https://lwn.net/Articles/115565/>) that some form of this patch is likely to be merged: 

So I guess we're going to have to do this sometime - I don't think there's any other solution apart from going fully tickless, which would be considerably more intrusive. 

Before the patch can be merged, however, a few details must be dealt with - porting it from 2.4 to 2.6, for example. So it's unlikely to go in immediately. Given time, however, it seems likely to be merged in some form.
