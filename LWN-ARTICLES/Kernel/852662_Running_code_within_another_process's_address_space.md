---
title: "Running code within another process's address space"
url: https://lwn.net/Articles/852662/
date: "April 16, 2021"
category: "System calls-process vm exec"
author: "By Jonathan Corbet April 16, 2021"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
April 16, 2021

One of the key resources that defines a process is its address space — the set of mappings that determines what any specific memory address means within that process. An address space is normally private to the process it belongs to, but there are situations where one process needs to make changes to another process's memory; an interactive debugger would be one case in point. The [`ptrace()`](<https://man7.org/linux/man-pages/man2/ptrace.2.html>) system call makes such changes possible, but it is slow and not always easy to use, so there has been a longstanding quest for better alternatives. One possibility, [`process_vm_exec()`](<https://lwn.net/ml/linux-kernel/20210414055217.543246-1-avagin@gmail.com/>) from Andrei Vagin, was recently posted for review. 

In truth, alternatives to `ptrace()` already exist for some tasks. The [cross-memory attach system calls](<https://lwn.net/Articles/405346/>) were [merged for 3.2](<https://lwn.net/Articles/465066/>) in 2011 as [`process_vm_readv()` and `process_vm_writev()`](<https://man7.org/linux/man-pages/man2/process_vm_readv.2.html>). As their names would suggest, they allow one process to read from and write to another process's memory. Those system calls satisfy many needs, but fall short when even more invasive access is needed to another process's address space. Sometimes, it seems, there is no alternative to running code _within_ the target address space. 

Vagin's patch set gives a couple of examples of where this access would be useful. User-mode kernels, such as [User-mode Linux](<http://user-mode-linux.sourceforge.net/>) and [gVisor](<https://github.com/google/gvisor>), have to be able to intercept system calls made by a sandboxed process and, possibly, run them in the address space of that process. The [Checkpoint/Restore in User space](<https://criu.org/Main_Page>) project needs to reach deeply within a process to extract all of the information needed to checkpoint it. Both use cases are currently handled with `ptrace()` but, once again, better and faster alternatives are wanted. 

The alternative proposed by Vagin is a new system call: 

```
int process_vm_exec(pid_t pid, struct sigcontext uctx, unsigned long flags,
        			siginfo_t siginfo, sigset_t *sigmask, size_t sizemask);
```

A successful call will cause the calling process's address space to be changed to that of the process identified by `pid`. The cover letter notes that using a [pidfd](<https://lwn.net/Articles/794707/>) might be preferable; that would make this system call inconsistent with `process_vm_readv()` and `process_vm_writev()`, though. The values in `uctx` are used to load the processor registers (including the instruction pointer) before resuming execution in the new address space — an important step, since using the previous instruction pointer from the old address space is unlikely to yield satisfactory results in the new address space. 

If `flags` is zero, `process_vm_exec()` will change the address space, then resume execution as indicated by `uctx`; that execution will continue until the process either makes a system call or receives a signal. Either way, the old address space will be restored and `process_vm_exec()` will return to the caller. The `siginfo` structure will describe the event that interrupted execution in the other address space; if it's a system call, `siginfo` will be made to look as if a `SIGSYS` signal had been received. 

If, instead, `flags` contains `PROCESS_VM_EXEC_SYSCALL`, the purpose of the call is to invoke a system call within the target process's address space. In this case, `uctx` should contain the system call number and arguments in the appropriate registers, as would be the case for a real system call. The address space will be switched for the duration of the system call, then restored before returning to the caller. 

This patch series was posted as a proof of concept with the idea of getting comments on the proposed API. Jann Horn was quick to [respond](<https://lwn.net/ml/linux-kernel/CAG48ez0jfsS=gKN0Vo_VS2EvvMBvEr+QNz0vDKPeSAzsrsRwPQ@mail.gmail.com/>) that the proposed system call does not appear to fit the stated use cases well; it is too much for one and not enough for the other. For the case of running code within a different address space (as systems like User-mode Linux do), he suggested, creating a whole new process is overkill; it might be better to have a system call that allows the construction of new address spaces separately. For the checkpoint/restore case, instead, there may still be a need to access resources within a process beyond its address space, though he didn't say which resources those might be. Vagin [responded](<https://lwn.net/ml/linux-kernel/YHdoRAJb9VlZ5lSu@gmail.com/>) that a relatively generic system call seemed better than a whole set of specialized ones, even if the generic alternative is not a perfect fit to all use cases. 

Florian Weimer did [have another resource in mind](<https://lwn.net/ml/linux-kernel/87blahb1pr.fsf@oldenburg.str.redhat.com/>), though, that would be useful for the the GNU C library. There is a difference between how Linux implements [`setuid()`](<https://man7.org/linux/man-pages/man2/setuid.2.html>) and what POSIX requires: Linux only changes the credentials for the calling thread, while POSIX specifies that it must change the credentials for all of the threads running in a process. Currently, glibc implements POSIX semantics on Linux by sending signals to all threads so that they can all call `setuid()` together, which is less than ideal. It would be much nicer to just be able to call `setuid()` within the context of each thread without actually interrupting the threads. Such a feature could also be useful for implementing memory barriers, he said. 

There is clearly some tension here between creating a feature that would be useful in some contexts and trying to solve a larger and more complex problem. In such cases, developers must pick their path carefully; trying to do too much is a good way to ensure that nothing actually gets far enough to land in the mainline kernel. So what will happen with `process_vm_exec()` is far from clear at this point; it may eventually find its way to acceptance, but it could change form considerably before that happens.
