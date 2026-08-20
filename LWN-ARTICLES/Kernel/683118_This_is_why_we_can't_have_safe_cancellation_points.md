---
title: "This is why we can't have safe cancellation points"
url: https://lwn.net/Articles/683118/
date: "April 13, 2016"
category: System calls
author: "April 13, 2016 This article was contributed by Neil Brown"
---

April 13, 2016

This article was contributed by Neil Brown

Signals [have been described](<https://lwn.net/Articles/414618/>) as an "unfixable design" aspect of Unix. A [recent discussion](<https://lore.kernel.org/all/06079088639eddd756e2092b735ce4a682081308.1457486598.git.luto@kernel.org/>) on the linux-kernel mailing list served to highlight some of the difficulties yet again. There were two sides to the discussion, one that focused on solving a problem by working with the known challenges and the existing semantics, and one that sought to fix the purportedly unfixable.

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The context for this debate is the [`pthread_cancel(3)`](<http://man7.org/linux/man-pages/man3/pthread_cancel.3.html>) interface in the [Pthreads](<http://man7.org/linux/man-pages/man7/pthreads.7.html>) POSIX threading API. Canceling a thread is conceptually similar to killing a process, though with significantly different implications for resource management. When a process is killed, the resources it holds, like open file descriptors, file locks, or memory allocations, will automatically be released. 

In contrast, when a single thread in a multi-threaded process is terminated, the resources it was using cannot automatically be cleaned up since other threads might be using them. If a multi-threaded process needs to be able to terminate individual threads — if for example it turns out that the work they are doing is no longer needed — it must keep track of which resources have been allocated and where they are used. These resources can then be cleaned up, if a thread is canceled, by a cleanup handler registered with [`pthread_cleanup_push(3)`](<http://man7.org/linux/man-pages/man3/pthread_cleanup_push.3.html>). For this to be achievable, there must be provision for a thread to record the allocation and deallocation of resources atomically with respect to the actual allocation or deallocation. To support this Pthreads introduces the concept of "cancellation points".

These cancellation points are optional and can be disabled with a call to [`pthread_setcanceltype(3)`](<http://man7.org/linux/man-pages/man3/pthread_setcanceltype.3.html>). If the cancel type is set to `PTHREAD_CANCEL_ASYNCHRONOUS` then a cancellation can happen at any time. This is useful if the thread is not performing any resource allocation or not even making any system calls at all. In this article, though, we'll be talking about the case where cancellation points are enabled. 

#### On cancellation points and their implementation

From the perspective of an application, a "cancellation point" is any one of a number of POSIX function calls such as `open()`, and `read()`, and many others. If a cancellation request arrives at a time when none of these functions is running, it must take effect when the next cancellation-point function is called. Rather than performing the normal function of the call, it must call all cleanup handlers and cause the thread to exit. 

If the cancellation occurs while one of these function calls is waiting for an event, the function must stop waiting. If it can still complete successfully, such as a `read()` call for which some data has been received but a larger amount was requested, then it may complete and the cancellation will be delayed until the next cancellation point. If the call cannot complete successfully, the cancellation must happen within that call. The thread must clean up and exit and the interrupted function will not return. 

From the perspective of a library implementing the POSIX Pthreads API, such as the [musl C library](<http://www.musl-libc.org/>) (which was the focus of the discussions), the main area of interest is the handling of system calls that can block waiting for an event, and how this interacts with resource allocation. Assuming that `pthread_cancel()` is implemented by sending a signal, and there aren't really any alternatives, the exact timing of the arrival of the cancellation signal can be significant. 

  * If the signal arrives after the function has checked for any pending cancellation, but before actually making a system call that might block, then it is critical that the system call is not made at all. The signal handler must not simply return but must arrange to perform the required cleanup and exit, possibly using a mechanism like [`longjmp()`](<http://man7.org/linux/man-pages/man3/longjmp.3.html>). 

  * If the signal arrives during or immediately after a system call that performs some sort of resource allocation or de-allocation, then the signal handler must behave differently. It must let the normal flow of code continue so that the results can be recorded to guide future cleanup. That code should notice if the system call was aborted by a cancellation signal and start cancellation processing. The signal handler cannot safely do that directly; it must simply set a flag for other code to deal with. 

There are quite a number of system calls that can both wait for an event and allocate resources; `accept()` is a good example as it waits for an incoming network connection and then allocates and returns a file descriptor describing that connection. For this class of system calls, both requirements must be met: a signal arriving immediately before the system call must be handled differently than a signal arriving during or immediately after the system call. 

There are precisely three Linux system calls for which the distinction between "before" and "after" is straightforward to manage: `pselect()`, `ppoll()`, and `epoll_pwait()`. Each of these takes a `sigset_t` argument that lists some signals that are normally blocked before the system call is entered. These system calls will unblock the listed signals, perform the required action, then block them again before returning to the calling thread. This behavior allows a caller to block the cancellation signal, check if a signal has already arrived, and then proceed to make the system call without any risk of the signal being delivered just before the system call actually starts. Rich Felker, the primary author of musl, did [lament](<http://mid.gmane.org/20160310164104.GM9349@brightrain.aerifal.cx>) that if all system calls took a `sigset_t` and used it this way, then implementing cancellation points correctly would be trivial. Of course, as he acknowledged, ""this is obviously not a practical change to make."" 

Without this ability to unblock signals as part of every system call, many implementations of Pthread cancellation are racy. The [ewontfix.com](<http://ewontfix.com/2/>) web site goes into quite some detail on this race and its history and reports that [the approach taken in glibc](<https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/unix/sysdep.h;h=94a2ce0e37a1d531064923f249d9a8ef9c3e5739;hb=ab30899d880f9741a409cbc0d7a28399bdac21bf#l53>) is:

```
ENABLE_ASYNC_CANCEL();
        ret = DO_SYSCALL(...);
        RESTORE_OLD_ASYNC_CANCEL();
        return ret;
```

where `ENABLE_ASYNC_CANCEL()` directs the signal handler to terminate the thread immediately and `RESTORE_OLD_ASYNC_CANCEL()` directs it to restore the behavior appropriate for the `pthread_setcanceltype()` setting. 

If the signal is delivered before or during the system call this works correctly. If, however, the signal is delivered after the system call completes but before `RESTORE_OLD_ASYNC_CANCEL()` is called, then any resource allocation or deallocation performed by the system call will go unrecorded. The `ewontfix.com` site provides a simple test case that reportedly can demonstrate this race.

#### A clever hack

The last piece of background before we can understand the debate about signal handling is that musl has a solution for this difficulty that is "clever" if you ask Andy Lutomirski and "[a hack](<http://mid.gmane.org/CA+55aFzOcxbhXCm01+NMgY9=THYgjojvDGeYsnxe-vWfiX4X0g@mail.gmail.com>)" if you ask Linus Torvalds. The solution is almost trivially obvious once the problem is described as above so it should be no surprise that the description was developed with the solution firmly in mind.

The signal handler's behavior must differ depending on whether the signal arrives just before or just after a system call. The handler can make this determination by looking at the code address (i.e. instruction pointer) that control will return to when the handler completes. The details of getting this address may require poking around on the stack and will differ between different architectures but the information is reliably available.

As Lutomirski explained when starting the thread, `musl` uses a single code fragment (a [thunk](<https://en.wikipedia.org/wiki/Thunk>)) like:

```
cancellable_syscall:
            test whether a cancel is queued
            jnz cancel_me
            int $0x80
        end_cancellable_syscall:
```

to make cancellable system calls. ("`int $0x80`" is the traditional way to enter the kernel for a system call by behaving like an interrupt). If the signal handler finds the return address to be at or beyond `cancellable_syscall` but before `end_cancellable_syscall`, then it must arrange for termination to happen without ever returning to that code or letting the system call be performed. If it has any other value, then it must record that a cancel has been requested so that the next cancellable system call can detect that and jump to `cancel_me`.

This "clever hack" works correctly and is race free, but is not perfect. Different architectures have different ways to enter a system call, including `sysenter` on x86_64 and `svc` (supervisor call) on ARM. For 32-bit x86 code there are three possibilities depending on the particular hardware: `int $0x80` always works but is not always the fastest. The `syscall` and `sysenter` instructions may be available and are significantly faster. To achieve best results, the [preferred way](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/arch/x86/entry/vdso/vdso32/system_call.S?id=b562e44f507e863c6792946e4e1b1449fbbac85d#n29>) to make system calls on a 32-bit x86 CPU is to make an indirect call through the `kernel_vsyscall()` entry point in the "[vDSO](<https://lwn.net/Articles/446528/>)" virtual system call area. This function will use whichever instruction is best for the current platform. If musl tried to use this for cancellable system calls it would run into difficulties, though, as it has no way to know where the instruction is, or to be certain that any other instructions that run before the system call are all located before that instruction in memory. So musl currently uses `int $0x80` on 32-bit x86 systems and suffers the performance cost. 

#### Cancellation for faster system calls

Now, at last, we come to Lutomirski's [simple patch](<http://mid.gmane.org/06079088639eddd756e2092b735ce4a682081308.1457486598.git.luto@kernel.org>) that started the thread of discussion. This patch adds a couple of new entry points to the vDSO, the important one for us is `pending_syscall_return_address`, which determines if the current signal happened during `kernel_vsyscall` handling and reports the address of the system call instruction. The caller can then determine if the signal happened before, during, or after that system call.

Neither Linus nor Ingo Molnar liked this approach, though their exact reasons weren't made clear. Part of the reason may have been that the semantics of cancellation appear clumsy so it is hard to justify much effort to support them. [According to Molnar](<http://mid.gmane.org/20160309085631.GA3247@gmail.com>), ""it's a really bad interface to rely on"". Even Lutomirski [expressed surprise](<http://mid.gmane.org/CALCETrXXx36buZyOhnYu-N3boRrCdK0a8p8yPHD+te1k3zYY=Q@mail.gmail.com>) that musl ""didn't take the approach of 'pthread cancellation is not such a great idea -- let's just not support it'."" Szabolcs Nagy's [succinct response](<http://mid.gmane.org/20160309113449.GZ29662@port70.net>) ""because of standards"" seemed to settle that issue. 

One clear complaint from Molnar was that there was ""so much complexity"" and it is true that the code would require some deep knowledge to fully understand. This concern is borne out by the fact that Lutomirski, who has that knowledge, hastily [withdrew his first](<http://mid.gmane.org/CALCETrWNSAWHhQmxuJr1eF+3nLkd50NGF3pQ2EiNvzLFKybegQ@mail.gmail.com>) and [second](<http://mid.gmane.org/CALCETrVMp6twHwdTrX3WAzEL+7qi1Aha5BxVz++V_Yc2AP8pBg@mail.gmail.com>) attempts. While complexity is best avoided where possible, complexity should not be, by itself, itself a justification for keeping something out of Linux.

Torvalds and Molnar contributed both by exploring the issues to flesh out the shared understanding and by proposing extra semantics that could be added to the Linux signal facility so that a more direct approach could be used.

Molnar [proposed](<http://mid.gmane.org/20160310111646.GA13102@gmail.com>) "sticky signals" that could be enabled with an extra flag when setting up a signal handler. The idea was that if the signal is handled other than while a system call is active, then the signal remains pending but is blocked in a special new way. When the next system call is attempted, it is aborted with `EINTR` and the signal is only then cleared. This change would remove the requirement that the signal handler must not allow the system call to be entered at all if the signal arrives just before the call, since the system call would now immediately exit.

Torvalds's [proposal](<http://mid.gmane.org/CA+55aFxvMM3j1aWjN-kr5Hn8CUC_RSNw5hc+X8zFXMaMv+mGww@mail.gmail.com>) was similar but involved "synchronous" signals. He saw the root problem being that signals can happen at any time and this is what leads to races. If a signal were marked as "synchronous" then it would only be delivered during a system call. This is exactly the effect achieved with `pselect()` and friends and so could result in a race-free implementation.

The problem with both of these approaches is that they are not selective in the correct way. POSIX does not declare all system calls to be cancellation points and, in fact, does not refer to system calls at all. It is only certain API functions that are defined as cancellation points and, as Torvalds clearly agreed that being able to use the faster system call entry made available in the vDSO was important, but neither he nor Molnar managed to provide a workable alternative to the solution proposed by Lutomirski.

Felker [made his feelings](<http://mid.gmane.org/20160312180531.GD9349@brightrain.aerifal.cx>) on the progress of the discussion quite clear:

I'm really frustrated that, again and again, we have kernel folks with no experience with libc implementation trying to redesign something that already has a simple zero-cost design that works on all existing systems, and proposing things that have a mix of immediately-obvious flaws and potential future problems we haven't even thought of yet. 

It is certainly important to get the best design, and exploring alternatives to understand why they were rejected is a valid part of the oversight provided by a maintainer. When that leads to the design being improved, we can all rejoice. When it leads to an understanding that the original design, while not as elegant as might be hoped, is the best we can have, it shouldn't prevent that design from being accepted. Once Lutomirski is convinced that he has all the problems resolved, it is to be hoped that a re-submission results in further progress towards efficient race-free cancellation points. Maybe that would even provide the incentive to get race-free cancellation points in other libraries like glibc.
