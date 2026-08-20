---
title: "Modifying another process's system calls"
url: https://lwn.net/Articles/1004455/
date: "January 14, 2025"
category: ptrace
author: "By Jonathan Corbet January 14, 2025"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
January 14, 2025

The [`ptrace()`](<https://man7.org/linux/man-pages/man2/ptrace.2.html>) system call allows a suitably privileged process to modify another in a large number of ways. Among other things, `ptrace()` can intercept system calls and make changes to them, but such operations can be fiddly and architecture-dependent. [This patch series from Dmitry Levin](<https://lwn.net/ml/all/20250107230153.GA30560@strace.io>) seeks to improve that situation by adding a new `ptrace()` operation to make changes to another process's system calls in an architecture-independent manner. 

`ptrace()` has, since the 5.3 release in 2019, supported an operation, `PTRACE_GET_SYSCALL_INFO`, that can be used when the traced process has been stopped at a system call. It is used by, for example, the [`strace`](<https://man7.org/linux/man-pages/man1/strace.1.html>) utility to obtain information about the system calls made by a process of interest. The addition of this operation made life easier for programs like `strace` which, previously, had needed special code to handle the unique way in which each architecture manages system-call arguments and return values. Now, the same `ptrace()` call works on all architectures supported by Linux. 

System calls can be intercepted — and information gathered — at three different points: on system-call entry, on the return to user space after the system call completes, or when a [`seccomp()`](<https://man7.org/linux/man-pages/man2/seccomp.2.html>) trace rule is executed. The information available at each point varies; on entry, for example, the system-call number and arguments are available. On exit, instead, `PTRACE_GET_SYSCALL_INFO` will provide the return value from the executed system call. This information comes back in the [`ptrace_syscall_info`](<https://elixir.bootlin.com/linux/v6.12.6/source/include/uapi/linux/ptrace.h#L82>) structure; the `ptrace()` manual page describes each of the returned fields. 

While `ptrace()` can be used to obtain system-call information, there is no equivalent way to change that information in an architecture-independent way. Any process that wants to mess with another process's interactions with the kernel must, thus, resort to lower-level means. Levin suggests that things should have been done differently: ""Ideally, PTRACE_SET_SYSCALL_INFO should have been introduced along with PTRACE_GET_SYSCALL_INFO, but it didn't happen"". Even in our less-than-ideal reality, though, that capability can be added now. 

Within the kernel, there exists a function to change a process's system-call arguments in an architecture-independent way. Or, at least, there once was. Roland McGrath [added `syscall_set_arguments()`](<https://git.kernel.org/linus/828c365cc8b8>) to the 2.6.27 kernel in 2008, but that function never acquired any users, so Peter Collingbourne [duly removed it](<https://git.kernel.org/linus/7962c2eddbfe7>) during the 5.16 development cycle in 2021. Levin starts by reverting that patch — partially, at least. The implementation of `syscall_set_arguments()` on some architectures was evidently buggy enough that it was better to just provide new versions outright. 

Levin also adds an internal `syscall_set_nr()` function to set the requested system-call number in an architecture-independent way; as can be seen from [the patch adding this function](<https://lwn.net/ml/all/20250107230438.GC30633@strace.io>), that operation must be done differently for each architecture. Of course, the level of architecture-independence achieved here is relative, since the system-call numbers themselves can vary from one architecture to the next. 

With that infrastructure in place, adding `PTRACE_SET_SYSCALL_INFO` to `ptrace()` is a relatively straightforward task. At system-call entry, this call can change both the system-call number and the arguments provided, possibly yielding a result that is rather different from what the calling process intended. The system-call number can also be set to `-1`, which will result in the call being skipped altogether and the `errno` value being set to `ENOSYS`. The same changes can be made for system calls intercepted by `seccomp()`. At system-call exit, instead, only the system call's return value can be changed. 

The other values found in the `ptrace_syscall_info` structure, including the instruction and stack pointers, cannot be modified with the new operation. That could possibly change in the future, Levin said in the cover letter, should there be a need to modify those values. There is a set of three padding bytes in the structure that must be set to zero in the current version of the patch; future versions could look there for flags indicating other changes to be made. The size of this structure is passed in from user space, meaning that it could be expanded in a compatible manner if the desire to change even more system-call-related parameters were to arise. 

For now, though, the patch set is limited to the basic operations described above. One thing that is missing from the submission is a description of how this new feature might be used. One can imagine various types of sandboxing solutions that, among other things, limit the system calls a process can make, with the ability to make changes (or even emulate system calls) as needed; enhancements to `seccomp()` [have been targeted at this sort of use case](<https://lwn.net/Articles/899281/>) in the past. The development community may want to see more information about the intended uses this time around, but any sort of concerted opposition to this functionality would be surprising. In the end, it does not allow anything that `ptrace()` cannot already do.
