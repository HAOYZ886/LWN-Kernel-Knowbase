---
title: Virtual time
url: https://lwn.net/Articles/180375/
date: "April 18, 2006"
category: Timekeeping; Virtualization
author: 
---

The developers interested in containers and virtualization have discussed interfaces to virtualize access to a number of system resources. None, however, have talked about virtualizing access to the system time. Until now, that is. With Jeff Dike's [time virtualization patches](<http://lwn.net/Articles/179825/>) any process tree can have its own idea of what time it is. 

Jeff's patch adds a new "time namespace" structure to the task structure. By default, all processes share the normal host system's idea of time. But a new option (`CLONE_TIME`) to the `unshare()` system call allows a process to disconnect from the system time. After such a call, that process - and any children it creates - will be able to keep its own time value. Setting a virtualized time value is, unlike changing the normal system time, an unprivileged operation. 

Internally, a virtualized time is stored as a simple offset; whenever a process requests the current time, the offset is added to the the current system time and the sum is returned. This approach has the advantages of being simple and fast; a process running with virtualized time also does not give up time adjustments made, for example, by NTP. On the other hand, this implementation does not support the ability to confuse processes by messing deeply with their idea of time - running time at a different rate, for example, or even backward. Chances are that this omission will not upset more than a small percentage of potential users of virtualized time, however. 

Jeff's purpose is to speed up the `gettimeofday()` system call in User-mode Linux instances. If the kernel allows process subtrees to have their own time values, then User-mode Linux can simply use the host's `gettimeofday()` call, rather than intercepting that call and implementing it itself. Since `gettimeofday()` is one of the most frequently-used system calls, this optimization can make a significant difference. 

One other change is required, however, for User-mode Linux to get the benefit from this change. UML performs much of its process control using `ptrace()`; in particular, it intercepts and interprets system calls with the `PTRACE_SYSCALL` operation. What is really needed for a fast `gettimeofday()` is the ability to _not_ intercept that particular call. So Jeff's patch also extends `ptrace()` by adding a `PTRACE_SYSCALL_MASK` operation. This new operation can set a bitmask indicating which system calls should be intercepted, and which should be executed without stopping. 

The result, with a suitably patched UML, is a `gettimeofday()` call which runs at about 99% of the native process speed. That may well be good enough to make this patch a piece of the growing set of interfaces supporting virtualization and containers.
