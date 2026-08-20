---
title: BPF meets io_uring
url: https://lwn.net/Articles/847951/
date: "March 4, 2021"
category: "BPF-io uring; io uring-BPF support"
author: "By Jonathan Corbet March 4, 2021"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
March 4, 2021

Over the last couple of years, a lot of development effort has gone into two kernel subsystems: [BPF](<https://lwn.net/Articles/740157/>) and [io_uring](<https://lwn.net/Articles/810414/>). The BPF virtual machine allows programs from user space to be safely run within the context of the kernel, while io_uring addresses the longstanding problem of running system calls asynchronously. As the two subsystems expand, it was inevitable that the two would eventually meet; the first encounter happened in mid-February with [this patch set](<https://lwn.net/ml/io-uring/cover.1613563964.git.asml.silence@gmail.com/>) from Pavel Begunkov adding the ability to run BPF programs from within io_uring. 

The patch set itself is relatively straightforward, adding less than 300 lines of new code. It creates a new BPF program type (`BPF_PROG_TYPE_IOURING`) for programs that are meant to be run in the io_uring context. Any such programs must first be created with the [`bpf()`](<https://man7.org/linux/man-pages/man2/bpf.2.html>) system call, then registered with the ring in which they are intended to run using the new `IORING_ATTACH_BPF` command. Once that has been done, the `IORING_OP_BPF` operation will cause a program to be run within the ring. The final step in the patch series adds a helper function that BPF programs can use to submit new operations into the ring. 

As a proof of concept, the patch series does a good job of showing how BPF programs might be run from an io_uring. This work does not, though, really enable any new capabilities in its current form, which may be part of why there have been no responses to it on the list. There is little value to running a BPF program asynchronously to submit another operation; one could simply submit that operation directly instead. As is acknowledged in the patch set, more infrastructure will be needed before this capability will become useful to users. 

The obvious place where BPF can add value is making decisions based on the outcome of previous operations in the ring. Currently, these decisions must be made in user space, which involves potential delays as the relevant process is scheduled and run. Instead, when an operation completes, a BPF program might be able to decide what to do next without ever leaving the kernel. "What to do next" could include submitting more I/O operations, moving on to the next in a series of files to process, or aborting a series of commands if something unexpected happens. 

Making that kind of decision requires the ability to run BPF programs in response to other events in the ring. The sequencing mechanisms built into io_uring now would suffice to run a program once a specific operation completes, but that program will not have access to much useful information about _how_ the operation completed. Fixing that will, as Begunkov noted, require a way to pass the results of an operation into a BPF program when it runs. An alternative would be to tie programs directly to submitted operations (rather than making them separate operations, as is done in the patch set) that would simply run at completion time. 

With that piece in place, and with the increasing number of system calls supported within io_uring, it will become possible to create complex, I/O-related programs that can run in kernel space for extended periods. Running BPF programs may look like an enhancement to io_uring, but it can also be seen as giving BPF the ability to perform I/O and run a wide range of system calls. It looks like a combination that people might do some surprising things with. 

That said, this is not a feature that is likely to be widely used. On its own, io_uring brings a level of complexity that is only justified for workloads that will see a significant performance improvement from asynchronous processing. Adding BPF into the mix will increase the level of complexity significantly, and long sequences of operations and BPF programs could prove challenging to debug. Finally, loading io_uring programs requires either of the `CAP_BPF` or `CAP_SYS_ADMIN` capabilities, which means "root" in most configurations. As long as the current [hostility toward unprivileged BPF programs](<https://lwn.net/Articles/796328/>) remains, that is unlikely to change; as a result, relatively few programs are likely to use this feature. 

Still, the combination of these two subsystems provides an interesting look at where Linux may go in the future. Linux will (probably) never be a [unikernel](<https://en.wikipedia.org/wiki/Unikernel>) system, but the line between user space and the kernel does appear to be getting increasingly blurry.
