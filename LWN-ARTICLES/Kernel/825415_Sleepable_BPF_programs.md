---
title: Sleepable BPF programs
url: https://lwn.net/Articles/825415/
date: "July 7, 2020"
category: BPF
author: "By Jonathan Corbet July 7, 2020"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
July 7, 2020

When support for classic BPF was added to the kernel many years ago, there was no question of whether BPF programs could block in their execution. Their functionality was limited to examining a packet's contents and deciding whether the packet should be forwarded or not; there was nothing such a program could do to block. Since then, BPF has changed a lot, but the assumption that BPF programs cannot sleep has been built deeply into the BPF machinery. More recently, classic BPF has been pushed aside by the [extended BPF](<https://lwn.net/Articles/740157/>) dialect; the wider applicability of extended BPF is now forcing a rethink of some basic assumptions. 

BPF programs can now do many things that were not possible for classic BPF programs, including calling helper functions in the kernel, accessing data structures ("maps") shared with the kernel or user space, and synchronizing with spinlocks. The core assumption that BPF programs are atomic has not changed, though. Once the kernel jumps into a BPF program, that program must complete without doing anything that might put the thread it is running in to sleep. BPF programs themselves have no way of invoking any sort of blocking action, and the helper functions exported to BPF programs by the kernel are required to be atomic. 

As BPF gains functionality and grows toward some sort of sentient singularity moment, though, the inability to block is increasingly getting in the way. There has, thus, been interest in making BPF programs sleepable for some time now, and that interest has recently expressed itself as code in the form of [this patch set](<https://lwn.net/ml/netdev/20200630043343.53195-1-alexei.starovoitov@gmail.com/>) from Alexei Starovoitov. 

The patch adds a new flag, `BPF_F_SLEEPABLE`, that can be used when loading BPF programs into the kernel; it marks programs that may sleep during their execution. That, in turn, informs the BPF verifier about the nature of the program, and brings a number of new restrictions into effect. Most of these restrictions are the result of the simple fact that the BPF subsystem was never designed with sleepable programs in mind. Parts of that subsystem have been updated to handle sleeping programs correctly, but many other parts have not. That is likely to change over time but, until then, the functionality implemented by any part of the BPF subsystem that still expects atomicity is off-limits to sleepable programs. 

For example, of the many types of BPF programs supported by the kernel, only two are allowed to block: those run from the Linux security module subsystem and tracing programs (`BPF_PROG_TYPE_LSM` and `BPF_PROG_TYPE_TRACING`). Even then, tracing programs can only sleep if they are attached to security hooks or are attached to functions that have been set up for [error injection](<https://lwn.net/Articles/740146/>). Other types of programs are likely to be added in the future, but the coverage will never be universal. Many types of BPF programs are invoked from within contexts that, themselves, do not allow sleeping — deep within the network packet-processing code or attached to atomic functions, for example — so making those programs sleepable is just not going to happen. 

Sleepable BPF programs are also limited in the types of maps they can access; anything but hash or array maps is out of bounds. There is a further restriction with hash maps: they must be preallocated so that elements need not be allocated while a sleepable program is running. Most of the internal housekeeping within maps is currently done using read-copy-update (RCU), a protection scheme that breaks down if a BPF program is blocked while holding a reference to a map entry. The hash and array maps have been extended to use [RCU tasks](<https://lwn.net/Articles/607117/>), which can handle sleeping code, for some operations. Once again, it seems likely that support for the combination of BPF maps and sleepable programs will grow over time. 

Sleepable BPF programs, thus, run with a number of restrictions that do not apply to the atomic variety. With Starovoitov's patch set, there is exactly one thing they can do that's unavailable to atomic BPF programs; it takes the form of a new helper function: 

```
long bpf_copy_from_user(void *dest, u32 size, const void *user_ptr);
```

This helper is a wrapper around the kernel's `copy_from_user()` function, which copies data from user space into the kernel. User-space data may not be resident in RAM when a call like this is made, so callers must always be prepared to block while one or more pages are brought in from secondary storage. This has prevented BPF programs from reading user-space data directly; now sleepable BPF programs will be able to do so. One potential use for this ability would be to allow security-related BPF programs to follow user-space pointers and get a better idea of what user space is actually up to. 

This patch set is in its fifth revision as of this writing and seems likely to find its way into the mainline during the 5.9 merge window. After that, the restrictions on what sleepable BPF programs can do are likely to start getting in the way of users, so it would not be surprising to see work loosening those restrictions showing up relatively quickly. For some use cases, at least, BPF insomnia should soon be a thing of the past.
