---
title: Suspending and resuming BPF programs
url: https://lwn.net/Articles/1076210/
date: "June 19, 2026"
category: BPF
author: "By Daroc Alden June 19, 2026 LSFMM+BPF"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Daroc Alden**  
June 19, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

BPF programs can be used to extend many aspects the Linux kernel, but BPF programs must run to completion in the same context that they began. Kumar Kartikeya Dwivedi is working on changing that by allowing BPF programs to be expressed as coroutines. He spoke about his work at the 2026 [ Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>). While still experimental, the change promises to make long-running BPF tasks significantly easier to write. 

Frequently, a single logical task is spread across both space and time, he explained. Execution jumps between different locations, computations can be suspended, and so on. Being able to express this in BPF would make some kinds of extensions to kernel functionality much easier to write. For example, consider the task of collecting stack traces. The kernel has tracing facilities to gather combined stack traces for kernel and user-space code. It's more efficient to collect the user-space portion of the stack trace right before the kernel performs a context switch back to user space. When a stack trace is requested, the kernel part runs immediately, then the computation is suspended, and the user-space part of the collection runs later. Adding BPF into this workflow requires splitting a single logical operation across multiple independent functions, because there is currently no way to suspend an executing BPF program. 

Similar problems come up when implementing user-space networking. Attempting to send a packet with [`sendmsg()`](<https://man7.org/linux/man-pages/man2/sendmsg.2.html>) eventually makes a call to [`qdisc_run()`](<https://elixir.bootlin.com/linux/v7.0.12/source/include/net/pkt_sched.h#L117>), which can also do work for other threads, sending something that they queued. This is another case of follow-up work being done at a different time and place than the main task. People have experimented with writing whole applications in BPF, Dwivedi said, and that turns up even more of the same kind of problem. 

[ ![\[Kumar Kartikeya Dwivedi\]](https://static.lwn.net/images/2026/kumar-dwivedi-lsfmmbpf-small.png) ](<https://lwn.net/Articles/1077738>)

Historically, BPF programs have used hooks and callbacks. There's nothing wrong with that approach; it can express all of the semantics that kernel developers need. It does force programmers to write custom suspend/resume logic and break their program up over multiple functions, however. Dwivedi's solution is to introduce coroutines to BPF: functions that can be suspended, transfer control back to the kernel, and then be resumed later. He then went over his design for BPF coroutines in the kernel. 

The best solution would be stackless coroutines, akin to Rust's asynchronous functions or C++'s coroutines, he said. In these languages, the compiler is responsible for rewriting straight-line code into a form that can be suspended and resumed. The advantage is that, from the BPF verifier's point of view, not much would need to change. Resuming a coroutine would look like a normal indirect function call, and the compiler would handle the details of saving and loading intermediate values. The verifier would still need to ensure that the overall program's control flow is valid, and may need some changes to handle code that is split across kernel contexts. 

Dwivedi went into a bit of detail about how C++ implements coroutines to illustrate the interface that the verifier would see. In short, he said, C++ creates a structure that contains two function pointers that can be used to `resume()` or `destroy()` the coroutine. `resume()` uses an index recording where the coroutine last suspended, stored in that same structure, to choose which code to execute using a switch statement. Any variables that need to be saved across suspension points are stored in the same structure; any that don't need to be saved across suspension points remain on the stack, and are implicitly discarded when the task is suspended. The verifier's normal check that kernel resources are freed instead of just forgotten about prevents BPF programs from doing that with any locks or reference-counted structures. The two function pointers are always the first two elements of the coroutine structure, so that generic code doesn't need to know how big the coroutine's frame is or how it is laid out. 

This is pretty similar to how Rust compiles asynchronous functions, except that it uses its type system to convey the `resume()` and `destroy()` callbacks. The Rust compiler can be easily coerced to produce a structure with the same layout as C++, however, so the verifier will probably only need to handle that one layout. From the point of view of tracking the types of BPF values, the coroutine's associated structure can be handled in the same way as the stack: values are spilled to it and loaded from it, but it doesn't need special handling. 

The things the verifier will have to do to make sure a coroutine is safe, beyond verifying the constraints that apply to all BPF code, include making sure the `resume()` and `destroy()` function pointers are not overwritten, making sure that the index only takes on valid values, and making sure that it's always legal to call `destroy()` when the coroutine is suspended. The verifier will also need to check that locks aren't held across suspension points, but that check can reuse the verifier's logic for determining that locks are released before a function returns. By the same logic, the verifier may need to invalidate map values held across suspension points, depending on their types. 

Andrii Nakryiko asked how the verifier is supposed to ensure that the coroutine doesn't enter an infinite loop, perhaps by setting the index back to an earlier value. The body of the coroutine is still verified, and at every suspension point the verifier checks the call to `resume()`, so the verifier can identify any loops in the same way that it finds existing loops, Dwivedi said. Nakryiko asked about subtler loops, but Dwivedi pointed out that it was already possible to have two BPF programs arm each other's timers, and that such infinite loops don't cause an actual problem as long as they don't cause the kernel to become unavailable. For that matter, even with the verifier limiting BPF programs to less than one million verified instructions, there's nothing preventing a user from attaching a 999,999 instruction BPF program that makes lots of expensive kfunc calls to every available kernel hook and slowing the system to a crawl. The point of the limit is to prevent infinite loops that might cause deadlocks, so that the system can continue to make forward progress, not to prevent BPF programs from wasting CPU time. 

As long as the verifier checks that, from every suspended state, it is always valid to `destroy()` the BPF program (in case it is unloaded) and `resume()` it, the program can't subvert BPF's safety guarantees. Even if there is a clever way to set up an infinite loop, it still won't be able to deadlock the kernel, since every time the coroutine is suspended it will have to release any held locks and give control back to the kernel. 

Dwivedi has a prototype implementation in progress, but there is still more work to be done. He wants to extend the BTF debugging information for programs that use coroutines, for example. His prototype also had to enable aggregate return types from functions to make his test C++ programs work correctly, so that will need verifier support. Adding Rust support should not be too much more difficult. 

A more experimental idea for the future is to allow suspended computations to switch between user space and kernel space. Dwivedi had one of his students work on a prototype for that, but it is definitely not ready yet. If that ever does work, it will enable applications to perform setup in user space, then transition to the kernel and use native BPF capabilities, before potentially switching back to user space. That kind of interface would blur the distinction between user space and the kernel — but it will be a long time before it becomes a reality, if it ever does. 

In the nearer term, Dwivedi intends to polish up his current work to prepare it for the kernel. While incorporating coroutines into BPF will not technically enable anything new, his hope is that it will make it easier to integrate BPF programs into the parts of the kernel that can't easily be simplified down to a single hook or callback.
