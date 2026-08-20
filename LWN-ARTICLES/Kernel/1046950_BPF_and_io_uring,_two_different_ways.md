---
title: "BPF and io_uring, two different ways"
url: https://lwn.net/Articles/1046950/
date: "November 20, 2025"
category: "BPF-io uring; io uring-BPF support"
author: "By Jonathan Corbet November 20, 2025"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
November 20, 2025

BPF allows programs uploaded from user space to be run, safely, within the kernel. The io_uring subsystem, too, can be thought of as a way of loading programs in the kernel, though the programs in question are mostly a sequence of I/O-related system calls. It has sometimes seemed inevitable that io_uring would, like many other parts of the kernel, gain BPF capabilities as a way of providing more flexibility to user space. That has not yet happened, but there are currently two patch sets under consideration that take different approaches to the problem. 

An io_uring "program" is built by placing a series of entries in a submission queue managed in a ring buffer shared between the kernel and user space. Each submission-queue entry (SQE) describes a system call to be performed, and may make use of special buffers and file descriptors maintained within io_uring itself. Each SQE is normally executed asynchronously, but it is possible to link a series of SQEs so that each is only executed after the successful completion of the previous one. The result of each operation is stored in a completion-queue entry (CQE) in a second shared ring. Using io_uring, an application can keep many streams of I/O going concurrently with a minimum of system calls. 

The io_uring linkage mechanism enables simple sequences of operations, such as creating a file, writing a buffer to that file, and closing the file. It does not offer much flexibility; one operation cannot pass information to the next or change how subsequent operations may execute. So it is not surprising that adding BPF support is seen as a way of filling that gap. So far, though, no attempts at adding that support have been seriously considered for merging into the mainline. 

#### BPF operations

In early November, Ming Lei posted [a patch set](<https://lwn.net/ml/all/20251104162123.1086035-1-ming.lei@redhat.com>) adding BPF support to io_uring in the form of a new operation, `IORING_OP_BPF`, that can be placed in the submission queue. The linkage mechanism can be used, for example, to cause a BPF program to be run between two other io_uring operations. The programs themselves can be set up as, essentially, new io_uring operations. 

Specifically, the patch creates a new [struct ops program type](<https://lwn.net/Articles/811631/>) for defining BPF operations. A user-space program will fill in and register a `uring_bpf_ops` structure: 

```
struct uring_bpf_ops {
    	unsigned short		id;
    	uring_io_prep_t		prep_fn;
    	uring_io_issue_t	issue_fn;
    	uring_io_fail_t		fail_fn;
    	uring_io_cleanup_t	cleanup_fn;
        };
```

The `id` is an operation ID; only the bottom eight bits are used, meaning that a program can establish up to 256 separate BPF-based operations. The rest of the fields are BPF programs that will implement the functions required by io_uring to set up, execute, and clean up after I/O operations. There are a couple of new kfuncs provided for those programs to obtain the request data from an SQE and to store a result of the operation in the proper CQE. 

Once the operation has been set up, io_uring SQEs can make use of it with an `IORING_OP_BPF` operation specifying the appropriate ID. Two buffers can be passed to a BPF operation in each request; a new kfunc has been added to allow BPF programs to bulk-copy data between buffers. One of the use cases targeted by this work is to make it easy to copy data between user space and in-kernel buffers that are not readily accessible from user space; this feature would evidently be helpful for the increasingly capable [ublk io_uring-based block driver subsystem](<https://lwn.net/Articles/903855/>). 

The number of review comments on this work has been relatively small. Stefan Metzmacher [said](<https://lwn.net/ml/all/94f94f0e-7086-4f44-a658-9cb3b5496faf@samba.org>): ""This sounds useful to me"". But Pavel Begunkov [was rather more negative](<https://lwn.net/ml/all/891f4413-9556-4f0d-87e2-6b452b08a83f@gmail.com>), saying that attempts to add BPF operations to io_uring in the past did not work well. The performance of BPF programs in that context, he said, is poor due to the associated io_uring overhead. He has a different approach, he added, that seems more promising. 

#### Hooking into the control loop

Shortly thereafter, Begunkov posted [a new version](<https://lwn.net/ml/all/cover.1763031077.git.asml.silence@gmail.com>) of a series he [has been working on sporadically](<https://lwn.net/Articles/847951/>) to add BPF support to io_uring in a different way. Rather than add a new operation type, this series adds a new hook into the io_uring completion loop, allowing a BPF program to be run as operations finish. This implementation, he said, can improve performance by moving CQE processing from user space into the kernel. It also, he said, could eventually allow for the removal of the io_uring linkage mechanism, which he called ""a large liability"" due to the complexity it adds, entirely. 

This series, which shows some signs of having been prepared in a hurry, also sets up a struct ops hook. It adds a single callback which, according to [the changelog](<https://lwn.net/ml/all/5bcb51a2d391e7dfb732c5cdbd152dfbd021b51b.1763031077.git.asml.silence@gmail.com>), should be called `handle_events()`, but is actually: 

```
int (*loop)(struct io_ring_ctx *ctx, struct iou_loop_state *ls);
```

The `ctx` field gives information about the submission and completion queues, while the `iou_loop_state` parameter can be used to control how often `loop()` is to be called, determined by the number of available CQEs and a timeout. When this program is called, it can look at the completed operations, if any, and possibly enqueue new operations in response. 

There is a pair of new kfuncs to go along with this mechanism. Pointers to the various parts of the ring buffer can be had with `bpf_io_uring_get_region()` (though Begunkov says that this interface is likely to be replaced in a future version), and `bpf_io_uring_submit_sqes()` can be used to submit new operations. Using these kfuncs, a BPF program could replace links by waiting for operations of interest to complete, then submitting the next operation that should follow, perhaps using information from the operations that have already completed. 

Lei, having looked at Begunkov's patches, [said](<https://lwn.net/ml/all/aQ4WTLX9ieL5J7ot@fedora>) that they do not solve the problem as well as his operation-based approach. The key difference Lei pointed out is that, with `IORING_OP_BPF`, the bulk of the application logic, including the creation of SQEs, remains in user space. With Begunkov's series, instead, much of the application logic must be pushed into the kernel, necessitating a lot of communication between user space and the kernel that has the potential to hurt performance. Begunkov [answered](<https://lwn.net/ml/all/9b59b165-1f57-4cb6-ae62-403d922ad4da@gmail.com>) that the communication can be handled efficiently using a [BPF arena](<https://lwn.net/Articles/1019885/>), and that his approach provides a greater level of flexibility to handle more types of applications. 

Neither developer appears to have convinced the other. Lei [intends](<https://lwn.net/ml/all/aRVcAFOsb7X3kxB9@fedora>) to continue work on `IORING_OP_BPF`, while Begunkov is likely to do the same with his patch. Both developers have said that there might be room in the kernel for both approaches, though one might reasonably expect resistance from the wider BPF community to adding what appears to be redundant functionality. A third possibility — that io_uring and BPF remain unintegrated as they have for years — remains a possibility as well.
