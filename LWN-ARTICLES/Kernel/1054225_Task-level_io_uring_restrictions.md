---
title: "Task-level io_uring restrictions"
url: https://lwn.net/Articles/1054225/
date: "January 19, 2026"
category: "BPF-io uring; Releases-7.0; io uring-Security"
author: "By Jonathan Corbet January 19, 2026"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
January 19, 2026

The [io_uring subsystem](<https://man7.org/linux/man-pages/man7/io_uring.7.html>) is more than an asynchronous I/O interface for Linux; it is, for all practical purposes, an independent system-call API. It has enabled high-performance applications, but it also brings challenges for code built around classic, Unix-style system calls. For example, the [`seccomp()`](<https://man7.org/linux/man-pages/man2/seccomp.2.html>) sandboxing mechanism does not work with it, causing applications using `seccomp()` to disable io_uring outright. Io_uring maintainer Jens Axboe is seeking to improve that situation with a rapidly evolving patch series adding a new restrictive mechanism to that subsystem. 

The core feature of `seccomp()` is restricting access to system calls; an installed filter can examine each system call (along with its arguments) made by a thread and decide whether to allow the call to proceed or not. The operations provided by io_uring are analogous to system calls, so one might well want to restrict them in the same way. But `seccomp()` has no visibility into — and thus no way to control — operations requested via io_uring. Running a program under `seccomp()` and allowing it access to io_uring almost certainly gives that program a way to bypass the sandboxing entirely. 

As it turns out, io_uring itself supports a mechanism that allows the placement of limits on io_uring operations; LWN [covered an early version of this feature](<https://lwn.net/Articles/826053/>) in 2020\. To create an operation-restricted ring, a process fills in an array of `io_uring_restriction` structures: 

```
struct io_uring_restriction {
    	__u16 opcode;
    	union {
    		__u8 register_op; /* IORING_RESTRICTION_REGISTER_OP */
    		__u8 sqe_op;      /* IORING_RESTRICTION_SQE_OP */
    		__u8 sqe_flags;   /* IORING_RESTRICTION_SQE_FLAGS_* */
    	};
    	/* Some reserved fields omitted */
        };
```

While the term "restriction" is used throughout the API, what these structures are doing is describing the _allowed_ operations. Each has a sub-operation code affecting what is allowed: 

  * `IORING_RESTRICTION_REGISTER_OP` allows a specific [registration operation](<https://man7.org/linux/man-pages/man2/io_uring_register.2.html>) — an operation that affects the ring itself. These operations include registering files or buffers, setting the clock to use, and even imposing these restrictions, among many others. 
  * `IORING_RESTRICTION_SQE_OP` enables an operation that can be queued in the ring; these include all of the I/O and networking operations supported by io_uring. The [`io_uring_enter()` man page](<https://man7.org/linux/man-pages/man2/io_uring_enter.2.html>) has a list of available operations. 
  * `IORING_RESTRICTION_SQE_FLAGS_ALLOWED` sets the list of operation flags that are allowed to appear in io_uring operations; these flags, listed in the `io_uring_enter()` man page, control the sequencing of operations, use of registered buffers, and more. 
  * `IORING_RESTRICTION_SQE_FLAGS_REQUIRED` creates a set of flags that _must_ appear in each operation. Making a flag required implicitly sets it as being allowed as well. 

The array of these structures can be installed with an `IORING_REGISTER_RESTRICTIONS` operation, after which it will be effective on the ring. This restriction mechanism is not as capable as what `seccomp()` can do; it cannot look at operation arguments, for example. But it is fast enough to not interfere with the performance goals of io_uring, and is sufficient to wall off significant parts of the API. 

There is, however, a significant limitation to the current restriction mechanism: restrictions can only be applied to an existing ring, and that ring must be in the disabled state at the time. It works well for an application that, for example, needs to create a ring, add restrictions, then pass it into a container. It falls short, though, for use cases that want to allow io_uring in general, but with a specified subset of operations. Axboe's work is intended to address this limitation by allowing restrictions to be applied to a task rather than to a specific ring. 

Specifically, this work [started](<https://lwn.net/ml/all/20260109185155.88150-1-axboe@kernel.dk>) by adding a new operation, `IORING_REGISTER_RESTRICTIONS_TASK`, that can accept the same list of `io_uring_restriction` structures. That list will be stored with the calling task itself, though, rather than with a specific ring, and the restrictions will be applied to all rings subsequently created by that task. The list is applied to children during a fork, so the restrictions will apply to all child processes created after they are set up. These restrictions thus govern any rings created in the future, without the controlling task having to participate in that creation. 

Once the restrictions have been set, they are immutable, with a couple of exceptions. The `IORING_REG_RESTRICTIONS_MASK` flag allows restrictions to be tightened further by removing allowed operations and flags, or by adding new required flags. The process that initially added the restrictions retains the power to modify them or remove them entirely. That process's children, instead, will remain stuck with the restrictions that were created for them. 

At least, that was the state of things as of the second RFC version of the patch set. The [third version](<https://lwn.net/ml/all/20260115165244.1037465-1-axboe@kernel.dk>) made a number of changes, starting with the removal of `IORING_REG_RESTRICTIONS_MASK` and any other ability to change the restrictions once they have been put into place. The bigger change, though, was the addition of more flexible filtering using, inevitably, a set of BPF programs. Interestingly, that flexibility was reduced somewhat in later versions, as will be seen. 

The current BPF implementation is a bit of a proof of concept. Among other things, it currently only properly filters the `IORING_OP_SOCKET` operation, which is the io_uring equivalent to the [`socket()`](<https://man7.org/linux/man-pages/man2/socket.2.html>) system call. Operations can be controlled, but registration requests are not currently included in the BPF mechanism. 

There is a new registration operation, `IORING_REGISTER_BPF_FILTER`, which adds a new BPF program to a ring; the program is associated with a specific `IORING_` operation code. It will be invoked after the initial preparation for a new operation has been done; as a result, any structures provided by user space as part of the operation will have been copied into the kernel and will be available for the program to inspect. That gives these filters an advantage over `seccomp()`, which generally cannot access data in user space that is passed to the kernel via pointers. 

The program will also be passed context specific to the operation in question; for `IORING_OP_SOCKET`, that context includes the address family, socket type, and protocol provided by user space. A non-zero return value from the BPF program allows the operation to proceed; otherwise it will be blocked. There can be multiple BPF programs attached to any given operation; they will be invoked in sequence, and any one of them can block an operation. While the current patch set does not implement this behavior, Axboe has [said](<https://lwn.net/ml/all/eafcaf17-dc4b-4f59-b120-7efcf93048f3@kernel.dk>) that he intends to change the behavior to ""deny by default"" in the future; if BPF is in use, then an operation will be disallowed unless a BPF program explicitly allows it. 

By the time the patch set reached [version 5](<https://lwn.net/ml/all/20260118172328.1067592-1-axboe@kernel.dk>) (with the "RFC" tag removed) things had changed again in an interesting way. There are two versions of BPF in the kernel, the "extended BPF" that is normally just called "BPF" in recent times, and "classic BPF", which is the earlier, BSD-derived variant that was designed for packet filtering. Classic BPF is far less capable and lacks compiler support; there have been no new users of it added to the kernel for years. But the current version of the io_uring patches now uses classic BPF rather than extended BPF. 

Axboe noted that the usability of the feature is reduced by this change: ""This obviously comes with a bit of pain on the usability front, as you now need to write filters in cBPF bytecode"". The change was driven by the fact that classic BPF can be used by unprivileged processes, while extended BPF requires privilege (specifically, the `CAP_BPF` capability). For the desired use case of sandboxing containers, accessibility without privilege is important. It is worth noting that `seccomp()` also still uses classic BPF, for the same reason. The hooks for extended BPF are still there, but cannot be used. 

As one might surmise, this patch set seems to be evolving quickly, and may well have changed again by the time you read this. It seems clear, though, that it will soon be possible to control access to io_uring at a level that, previously, has not been possible. Just as brakes allow a car to go faster, fine-grained control may make io_uring available in contexts where, until now, it has been blocked.
