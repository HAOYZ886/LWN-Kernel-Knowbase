---
title: Operations restrictions for io_uring
url: https://lwn.net/Articles/826053/
date: "July 15, 2020"
category: "Asynchronous IO; io uring-Security"
author: "By Jonathan Corbet July 15, 2020"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
July 15, 2020

The [io_uring subsystem](<https://lwn.net/Articles/776703/>) is not much over one year old, having been merged for the 5.1 kernel in May 2019. It was initially added as a better way to perform asynchronous I/O from user space; over time it has gained numerous features and support for functionality beyond just moving bits around. What it has not yet gained is any sort of security mechanism beyond what the kernel already provides for the underlying system calls. That may be about to change, though, as the result of [this patch set](<https://lwn.net/ml/linux-kernel/20200710141945.129329-1-sgarzare@redhat.com/>) from Stefano Garzarella adding a set of user-configurable restrictions to io_uring. 

As one might expect from its name, io_uring is based around a ring buffer shared between the kernel and user space that allows user space to submit operations to the kernel. There is a second ring that is filled with the results of those operations. Each operation can be thought of as a way of expressing a system call; operations may read or write buffers, open files, send network messages, or request any of [a number of other actions](<https://lwn.net/Articles/810414/>). Operations can be made contingent on the successful completion of previous operations. In short, the operation stream feeding into the kernel is a sort of language expressing a program that the kernel should execute asynchronously. 

Operations executed by io_uring result in calls to the code within the kernel that implements the corresponding system calls; an `IORING_OP_READV` operation, for example, ends up in the same place as a `readv()` system call. That code will perform the usual privilege checks, using the credentials of the process that created the ring in the first place. So, in the absence of bugs, a process can do nothing with io_uring that it would not be allowed to do with direct system calls — with the exception that [`seccomp()`](<https://www.man7.org/linux/man-pages/man2/seccomp.2.html>) filters do not apply to io_uring. This model has worked well for io_uring so far, but it turns out that there is a use case that could use a bit more control. 

In particular, what happens if a process wants to create a ring and hand it over to another, less-trusted process? For example, I/O from within virtualized guests could perhaps be accelerated considerably if it used io_uring. This I/O, which often goes through the [Virtio](<https://www.linux-kvm.org/page/Virtio>) mechanism now, involves a certain amount of data copying and context shifting that could be avoided this way. The hypervisor could create whatever file descriptors the client would need, which would correspond to specific devices or open network connections, then let the guest handle things directly through the ring from there. 

The problem with this idea is that the guest could then perform _any_ operation that io_uring supports. Remember that the ring retains the credentials of the creator, which would be the hypervisor in this case; giving such a ring to a client would open the door to actions like accessing other file descriptors opened by the hypervisor or opening new files with the hypervisor's credentials. This is likely to prove extremely disappointing to anybody counting on virtualization as a security barrier. 

The answer to this problem, according to Garzarella, is to allow the registration of restrictions on what a specific ring can do. He adds a new opcode (`IORING_REGISTER_RESTRICTIONS`) for this purpose. There are a few types of restrictions that can be added: 

> `IORING_RESTRICTION_REGISTER_OP`
>      Provides a list of registration operations that can be carried out with this ring. Registration operations install file descriptors and buffers in the ring, optimizing their use in subsequent operations. These are, in other words, setup operations for the ring itself that do not actually perform I/O. 
> 
> `IORING_RESTRICTION_SQE_OP`
>      The operations (actual system calls) that will be allowed in this ring are provided as a list. It's called a "whitelist" within the code, a term that seems more than usually likely to change before the patches find their way into the mainline. Any operation that does not appear in this list will be disallowed on the restricted ring. 
> 
> `IORING_RESTRICTION_FIXED_FILES_ONLY`
>      If this restriction is applied, only file descriptors that have been previously registered in the ring can be used in operations. In other words, this restriction can be used to limit a ring to operating on a specific set of known files. 

Most of the "restrictions" above are thus actually permissions; they specify the things that the ring is allowed to do. Among other things, the allowlist approach here will help prevent future surprises when new operations are inevitably added to the io_uring roster. Restrictions can be applied exactly once, after which they are fixed for as long as the ring exists. 

One final piece, [suggested](<https://lwn.net/ml/io-uring/f7f2841e-3dbb-377f-f8f8-826506a938a6@kernel.dk/>) by io_uring maintainer Jens Axboe in response to a previous version of the patch set, is a new flag (`IORING_SETUP_R_DISABLED`) that can be provided when the ring is first created. When present, that flag causes the ring to start in a disabled state; registration operations will still succeed, but any other operations will fail. That allows the ring creator to perform the necessary registrations and add restrictions without having to worry about any other thread starting to use the ring for I/O. Once the registration phase is complete, the `IORING_REGISTER_ENABLE_RINGS` registration operation will complete the ring setup and enable all (allowed) operations. 

This restrictions mechanism appears to be sufficient for the described use case of allowing restricted access to a specific set of file descriptors. It seems probable that somebody will want to add more sophisticated policy mechanisms at some point; a proposal to add a BPF hook for security decisions seems unavoidable. For the near future, though, the proposed restriction mechanism may help to speed up I/O in virtual machines or other untrusted environments, which seems like a useful improvement.
