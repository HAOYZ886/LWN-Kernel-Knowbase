---
title: "User-space interrupts"
url: https://lwn.net/Articles/871113/
date: "September 30, 2021"
category: "Architectures-x86"
author: "By Jonathan Corbet September 30, 2021 LPC"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
September 30, 2021

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2021-Linux_Plumbers_Conference>)

The term "interrupt" brings to mind a signal that originates in the hardware and which is handled in the kernel; even software interrupts are a kernel concept. But there is, it seems, a use case for enabling user-space processes to send interrupts directly to each other. An upcoming Intel processor generation includes support for this capability; at the [2021 Linux Plumbers Conference](<https://linuxplumbersconf.org/>), Sohil Mehta ran [a Kernel-Summit session](<https://linuxplumbersconf.org/event/11/contributions/985/>) on how Linux might support that feature. 

At its core, Mehta began, the user-space interrupts (or simply "user interrupts") feature is a fast way to do event signaling. It delivers signals directly to user space, bypassing the kernel to achieve lower latency. Our existing interprocess communication mechanisms all have limitations, he said. The synchronous mechanisms often require a dedicated thread, have high latency, and are generally inefficient. Asynchronous mechanisms [![\[Sohil Mehta\]](https://static.lwn.net/images/conf/2021/lpc/SohilMehta-sm.png)](<https://lwn.net/Articles/871117/>) (signals, for example) have even higher latency. So often the only alternative is polling, which wastes CPU time. It would be nice to have a fast, efficient alternative. 

That alternative is user-space interrupts, which will first appear in Intel's "Sapphire Rapids" CPUs. [RFC patches](<https://lwn.net/ml/linux-kernel/20210913200132.3396598-1-sohil.mehta@intel.com/>) supporting this feature were posted in mid-September. Those patches support user-to-user signaling without going through the kernel; instead, the new `SENDUIPI` instruction allows one process to send an interrupt directly to another process. Future versions will also include kernel-to-user signaling and, eventually, interrupts sent directly to user space from devices. 

Mehta put up some benchmark results (which can be seen in [the slides](<https://linuxplumbersconf.org/event/11/contributions/985/attachments/756/1417/User_Interrupts_LPC_2021.pdf>)) showing that user-space interrupts are nine times faster than using [`eventfd()`](<https://man7.org/linux/man-pages/man2/eventfd.2.html>), and 16 times faster than using pipes or signals. The advantage is lower if the receiving process is blocked in the kernel, since it is not possible to avoid a context switch in that case. Even then, user-space interrupts are 10% faster for the recipient, and significantly faster for the sender, which need not enter the kernel at all. Florian Weimer asked how user-space interrupts compared to futexes, but evidently that testing has not been done. 

Use cases for this feature include fast interprocess communication, of course. User-mode CPU schedulers should be able to benefit from it, as can user-space I/O stacks (networking, for example). Getting the full benefit from this feature will require enhancements to libraries like libevent and liburing. There are no real-world applications using this feature yet, Mehta said; he is interested in hearing about other applications that might benefit from it. Ted Ts'o suggested host-to-guest wakeups in virtualization environments; evidently that use case is being investigated, but there are no real results yet. 

For any number of good reasons, user-space processes cannot just arbitrarily send interrupts to others; there is some setup required. On the receiving side, it all starts with a call to: 

```
uintr_register_handler(handler, flags);
```

Where `handler()` is the function that is called to handle a user-space interrupt, and `flags` must be zero. The definition of the handler function requires a bit of special care; its prototype is: 

```
void __attribute__ ((interrupt))
            handler(struct __uintr_frame *frame, unsigned long long vector);
```

The next step is to create at least one file descriptor associated with this handler: 

```
int uintr_create_fd(u64 vector, unsigned int flags);
```

Here, `vector` is a number between zero and 63; one file descriptor can be created for each vector. The process then hands that file descriptor to the sending side. If the sender is another thread in the same process, the hand-off is trivial; otherwise a Unix-domain socket can be used to transfer the descriptor. The sender then performs its setup with: 

```
int uintr_register_sender(int fd, unsigned int flags);
```

Where `fd` is the file descriptor passed by the recipient and `flags`, as always, is zero. The return value is a handle that can be used with the `_senduipi()` intrinsic that is supported by GCC 11 to actually send an interrupt to the receiver. 

Actual delivery of the interrupt depends on what the receiver is doing at the time; if that process is running in user space, the handler function will be called immediately with the appropriate vector number. Once the handler returns, execution will continue at the point of interruption. If the receiver is blocked in a system call in the kernel, the interrupt will be delivered on return to user space without interrupting the in-progress system call. There is a `uintr_wait()` system call in the patch set that will block until a user-space interrupt arrives then return immediately, but it is described as a "placeholder" until the desired behavior for this case is worked out. 

Prakesh Sangappa asked whether it was really necessary to exchange the file descriptor with all senders; in a system where there could be large numbers of senders, that could get expensive. Mehta replied that there are a couple of optimizations that are being looked at. Ts'o asked whether user-space interrupts could be broadcast to multiple recipients; the answer is that broadcast is not supported. 

Arnd Bergmann wanted to know if any thought had been given to emulating this feature on older CPUs. The answer appears to be yes; the kernel will trap the relevant instructions and transparently emulate them. Mehta asked for feedback on the emulation mechanism and, in particular, whether it should be implemented for other architectures. Bergmann discouraged that idea, saying that if user-space interrupts are implemented for those architectures, they will surely not be compatible with the emulated version. Emulation for other architectures, he said, should only be done once those architectures have defined their own instructions. 

Greg Kroah-Hartman asked about whether the Clang compiler has support for the `_senduipi()` intrinsic; that support is being worked on, but is not yet ready. Kroah-Hartman also asked about more details on workloads that benefit from this feature, to which Mehta replied that he did not have anything specific to point to yet. 

Mehta closed the session (which was running out of time) by asking what should happen when the recipient is blocked in a system call. As mentioned, the current patch set waits for the system call to return before delivering the interrupt. Should the behavior be changed to be closer to signals, with the interrupt delivered immediately and the system call returning with an `EINTR` status? Nobody had an opinion to share on that question, so the session ended there. 

The video of this talk is [available on YouTube](<https://www.youtube.com/watch?v=gpFfJkrrEEs&t=2h27m18s>).
