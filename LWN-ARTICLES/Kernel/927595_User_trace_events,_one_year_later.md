---
title: "User trace events, one year later"
url: https://lwn.net/Articles/927595/
date: "April 3, 2023"
category: Tracing
author: "By Jonathan Corbet April 3, 2023"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
April 3, 2023

The kernel has a well-developed mechanism for the control of tracing of events in kernel space. Developers often want to be able to trace user-space activity as well, using the same interfaces, but that mode is rather less well supported. One year ago, [an attempt to add an API](<https://lwn.net/Articles/889607/>) for the control of user-space trace events ran into trouble and has never been fully enabled. Now, Beau Belgrave is back with [a reworked API](<https://lwn.net/ml/linux-kernel/20230328235219.203-1-beaub@linux.microsoft.com/>) that may finally result in this mechanism becoming generally available. 

User-space trace events, like those in the kernel, tend to be inactive most of the time; nobody is listening, so no data should be generated. In the kernel, inactive tracepoints are patched out of the binary entirely, but that sort of run-time patching is not readily available in user space. So user-space code must actually check whether a given trace event is enabled and, if so, produce the requisite data. Since any given event is almost certainly inactive, that check should be as fast as possible. 

In 2022, the API proposed by Belgrave required programs to map a special page into their address space with [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>). Each event registered with the kernel would be assigned a byte within that mapped page; whenever an event was activated, the kernel would store a non-zero value in the appropriate byte. The program could then check that byte whenever a tracepoint was hit to determine whether the tracing data should be produced. 

There were a few complaints about that API. The use of a whole byte to store a single bit of information was called out as being wasteful. The loss of seven bits of memory may not seem like a heavy cost, but the real problem was with cache utilization. Using a single bit for each event would allow the number of event flags stored in a cache line to be increased by a factor of eight. Since those flags might be checked frequently in a program's hottest paths, cache efficiency can make a significant difference in the overall cost of supporting tracing. The need to map a single page also does not sit well with libraries that might support tracing independently of the programs they are linked into. 

These complaints came late in the development process, after the code had been pulled into the tracing tree. The patches found their way into the 5.18 kernel release despite the criticisms, but the whole mechanism was marked "broken" in the kernel configuration system. As a result, the user-events interface is not enabled by default and is not built into most kernels today. 

The new API does away with the mapped page entirely. To register an event with the kernel, a program should open `/sys/kernel/tracing/user_events_data` and issue a `DIAG_IOCSREG` [`ioctl()`](<https://man7.org/linux/man-pages/man2/ioctl.2.html>) call, passing this structure as an argument: 

```
struct user_reg {
    	__u32 size;
    	__u8 enable_bit;
    	__u8 enable_size;
    	__u16 flags;
    	__u64 enable_addr;
    	__u64 name_args;
    	__u32 write_index;
        };
```

The size of the `user_reg` structure itself should be stored in `size`; that will allow the kernel to provide backward compatibility should this structure grow in the future. The `ioctl()` call will register a location at `enable_addr` that is `enable_size` bytes in length; that size must be four or eight bytes on a 64-bit system. The bit within that value that controls the event is described by `enable_bit`; the kernel will set or clear that bit (without disturbing the rest of the value) whenever the event is activated or deactivated. The `flags` value should be zero, and `name_args` points to a string that describes the data returned by the trace event using [this format](<https://www.kernel.org/doc/html/latest/trace/user_events.html#command-format>). 

On successful registration, the kernel will store a value into `write_index` that should be used whenever the program generates data for the given event. There is another operation (`DIAG_IOCSUNREG`) that can be used to unregister an event 

The new API allows the event-enable bits to be packed densely if that suits an applications needs. Otherwise, each bit could be placed into its own value, with the other bits in each used for other purposes, perhaps by user-space tracers. The result is a more flexible API that, with luck, addresses the criticisms that were aimed at its predecessor. 

This work is in its tenth revision as of this writing; the last few rounds have only brought relatively minor changes. That would suggest that reviewers are happy with the API at this point. The work [appears to be poised](<https://lwn.net/ml/linux-kernel/20230329194516.146147554@goodmis.org/>) to head upstream during the 6.4 merge window, accompanied by [a patch](<https://lwn.net/ml/linux-kernel/20230329194554.139185152@goodmis.org/>) noting that the new API ""appears to be as stable as it will be without the use of a crystal ball"" and finally removing the "broken" marking from the configuration option.
