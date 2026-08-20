---
title: "Emulating Windows system calls, take 2"
url: https://lwn.net/Articles/826313/
date: "July 17, 2020"
category: System calls
author: "By Jonathan Corbet July 17, 2020"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
July 17, 2020

Back in June, LWN [covered a patch set](<https://lwn.net/Articles/824380/>) adding a mechanism intended to help systems like [Wine](<https://winehq.org/>) emulate Windows system calls on a Linux system. That patch set got a lot of attention and comments, with the result that its form has changed considerably. Gabriel Krisman Bertazi has now posted [a new patch set](<https://lwn.net/ml/linux-kernel/20200712044516.2347844-1-krisman@collabora.com/>) that takes a different approach to solving the same problem. 

As a reminder, the intent of this work is to enable the running of Windows binaries that call directly into the Windows kernel without going through the Windows API. Those system calls must somehow be trapped and emulated for the program to run correctly; this must be done without modifying the Windows program itself, lest Wine run afoul of the cheat-detection mechanisms built into many of those programs. The previous attempt added a new [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>) flag that would mark regions of the program's address space as unable to make direct system calls. That was coupled with a new [`seccomp()`](<https://man7.org/linux/man-pages/man2/seccomp.2.html>) mode that would trap system calls made from the marked range(s). There were a number of concerns raised about this approach, starting with the fact that using `seccomp()` might cause some developers to think that it could be used as a security mechanism, which is not the case. 

The new patch set has thus moved away from `seccomp()` and uses `[prctl()](<https://man7.org/linux/man-pages/man2/prctl.2.html>)` instead, following one of the suggestions that was made in response to the first attempt. Specifically, a program wanting to enable system-call trapping and emulation would make a call to: 

```
prctl(PR_SET_SYSCALL_USER_DISPATCH, operation, start, end, selector);
```

Where `operation` is either `PR_SYS_DISPATCH_ON` or `PR_SYS_DISPATCH_OFF`. In the former case, system-call trapping will be enabled; in the latter it is turned off. The `start` and `end` addresses indicate a range of memory from which system calls will not be trapped, even when it is enabled; that is the place to put the code that performs the system-call emulation. The `selector` argument is a pointer to a byte in memory that provides another mechanism to toggle system-call trapping. 

When this feature is enabled with `PR_SYS_DISPATCH_ON`, the kernel sets a flag (`_TIF_SYSCALL_USER_DISPATCH`) in the process's task structure. This flag is tested whenever the process makes a system call. If it is set, a further check is made to the memory pointed to by the provided `selector`; if the value stored there is `PR_SYS_DISPATCH_OFF`, the system call will be executed normally. If, instead, that location holds `PR_SYS_DISPATCH_ON`, the kernel will deliver a `SIGSYS` signal to the process. 

The signal handler can then examine the register set at the time of the trap; that will indicate which system call was being made and its arguments. That call can be emulated in the handler; once the handler returns, the process will resume after the trapped system call. This handler must be placed in the special system-calls-allowed region of memory or things will not work well, even in the unlikely case that the handler makes no system calls of its own. In Linux, returning from a signal handler involves the special [`sigreturn()`](<https://man7.org/linux/man-pages/man2/sigreturn.2.html>) system call, which must be able to execute without trapping (recursively) into the handler. 

The special `selector` variable allows trapping to be quickly enabled and disabled without the need to call into the kernel every time. For a system like Wine, which moves frequently between Windows and Linux-native code, that should result in a measurable performance improvement. 

The initial implementation of this mechanism received a lot of comments on the mailing list. This time, comments are limited to [one from Kees Cook](<https://lwn.net/ml/linux-kernel/202007131715.5598282C@keescook/>), who said that it ""looks great"". So the way would seem to be clear for this feature to get into the mainline in the relatively near future.
