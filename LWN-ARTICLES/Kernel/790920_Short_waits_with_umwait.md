---
title: Short waits with umwait
url: https://lwn.net/Articles/790920/
date: "June 13, 2019"
category: umwait
author: "By Jonathan Corbet June 13, 2019"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
June 13, 2019

If a user-space process needs to wait for some event to happen, there is a whole range of mechanisms provided by the kernel to make that easy. But calling into the kernel tends not to work well for the shortest of waits — those measured in small numbers of microseconds. For delays of this magnitude, developers often resort to busy loops, which have a much smaller potential for turning a small delay into a larger one. Needless to say, busy waiting has its own disadvantages, so Intel has come up with a set of instructions to support short delays. A [patch set](<https://lwn.net/ml/linux-kernel/1559944837-149589-1-git-send-email-fenghua.yu@intel.com/>) from Fenghua Yu to support these instructions is currently working its way through the review process. 

The problem with busy waiting, of course, is that it occupies the processor with work that is even more useless than cryptocoin mining. It generates heat and uses power to no useful end. On hyperthreaded CPUs, a busy-waiting process could prevent the sibling thread from running and doing something of actual value. For all of these reasons, it would be a lot nicer to ask the CPU to simply wait for a brief period until something interesting happens. 

To that end, Intel is providing three new instructions. `umonitor` provides an address and a size to the CPU, informing it that the currently running application is interested in any writes to that range of memory. A `umwait` instruction tells the processor to stop executing until such a write occurs; the CPU is free to go into a low-power state or switch to a hyperthreaded sibling during that time. This instruction provides a timeout value in a pair of registers; the CPU will only wait until the timestamp counter (TSC) value exceeds the given timeout value. For code that is only interested in the timeout aspect, the `tpause` instruction will stop execution without monitoring any addresses. 

It's worth noting that these are unprivileged instructions; any process can execute them. As a general rule, instructions that can halt a processor (or put it into a low-power state) are not available to unprivileged code for fairly obvious reasons. In this case, these instructions have (hopefully) been rendered safe by allowing the kernel to set an upper bound on how long the `umwait` and `tpause` instructions can wait before normal execution continues. Yu's patch set makes that upper bound available to system administrators in a sysfs file: 

```
/sys/devices/system/cpu/umwait_control/max_time
```

Since the TSC is involved, this value is in processor cycles; the default is 100,000, or about 100µs on a 1GHz CPU. This value was [suggested](<https://lwn.net/ml/linux-kernel/CALCETrVenEpKH5O=Z6J+faWTcdZpa1xx4-dYwdWu1OSEDyiUWw@mail.gmail.com/>) by Andy Lutomirski during a discussion on a previous version of the patch set in January; his reasoning was: 

I think we should set the default to something quite small, maybe 100 microseconds. IMO the goal is to pick a value that is a high enough multiple of the C0.2 entry+exit latency that we get most of the power and SMT resource savings while being small enough that no one thinks that UMWAIT is more than a glorified, slightly improved, and far more misleading version of REP NOP. 

The "C0.2" mentioned above is one of two special low-power states that the CPU can go into while waiting with one of these instructions; the other is, unsurprisingly, named C0.1. The C0.1 state is a "light" low-power state that doesn't reduce power usage that much, but which can be exited with relatively little latency. C0.2 is a deeper sleep that saves more power and takes longer to get out of. 

It is conceivable that system administrators might not want to allow the system to go into C0.2 if, for example, it is handling workloads with realtime response requirements. The `enable_c02` file in the same sysfs directory can be used to restrict the processor to C0.1. The default is to allow the deeper sleeps. 

In the same message linked above, Lutomirski worried about the security implications of instructions that allow a process to monitor when a range of memory is touched. As he put it, `umwait` ""seems quite delightful as a tool to create a highish-bandwidth covert channel, and it's possibly quite nice to augment Spectre-like attacks"". Exactly how useful it would be has not really been described anywhere, though doubtless there will be an academic paper on the topic in the near future. Yu did [answer](<https://lwn.net/ml/linux-kernel/20190117000731.GA226938@romley-ivt3.sc.intel.com/>) that these instructions can be disabled outright (with a significant performance cost), though no administrator-level knob has been provided to do that. 

Meanwhile, these instructions (which should appear in the upcoming "Tremont" processors) do appear to offer some value to specific types of workloads. Most of the comments on the patches have been addressed, with seemingly little left to fix at this point. So, most likely, there will be kernel support for the `umwait` family of instructions in the near future.
