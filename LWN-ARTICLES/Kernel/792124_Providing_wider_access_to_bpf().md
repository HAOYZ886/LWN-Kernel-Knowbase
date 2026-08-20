---
title: Providing wider access to bpf()
url: https://lwn.net/Articles/792124/
date: "June 27, 2019"
category: "BPF-Unprivileged"
author: "By Jonathan Corbet June 27, 2019"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
June 27, 2019

The [`bpf()`](<http://man7.org/linux/man-pages/man2/bpf.2.html>) system call allows user space to load a BPF program into the kernel for execution, manipulate BPF maps, and carry out a number of other BPF-related functions. BPF programs are verified and sandboxed, but they are still running in a privileged context and, depending on the type of program loaded, are capable of creating various types of mayhem. As a result, most BPF operations, including the loading of almost all types of BPF program, are restricted to processes with the `CAP_SYS_ADMIN` capability — those running as root, as a general rule. BPF programs are useful in many contexts, though, so there has long been interest in making access to `bpf()` more widely available. One step in that direction has been [posted](<https://lwn.net/ml/netdev/20190625182303.874270-1-songliubraving@fb.com/>) by Song Liu; it works by adding a novel security-policy mechanism to the kernel. 

This approach is easy enough to describe. A new special device, `/dev/bpf` is added, with the core idea that any process that has the permission to open this file will be allowed ""to access most of sys_bpf() features"" — though what comprises "most" is never really spelled out. A non-root process that wants to perform a BPF operation, such as creating a map or loading a program, will start by opening this file. It then must perform an `ioctl()` call (`BPF_DEV_IOCTL_GET_PERM`) to actually enable its ability to call `bpf()`. That ability can be turned off again with the `BPF_DEV_IOCTL_PUT_PERM` `ioctl()` command. 

Internally to the kernel, this mechanism works by adding a new field (`bpf_flags`) to the `task_struct` structure. When BPF access is enabled, a bit is set in that field. If this patch goes forward, that detail is likely to change since, as Daniel Borkmann [pointed out](<https://lwn.net/ml/netdev/9bc166ca-1ef0-ee1e-6306-6850d4008174@iogearbox.net/>), adding an `unsigned long` to that structure for a single bit of information is unlikely to be popular; some other location for that bit will be found. 

The next step is the addition of little function to determine whether the current process is capable of performing BPF operations: 

```
static inline bool bpf_capable(int cap)
        {
    	return test_bit(TASK_BPF_FLAG_PERMITTED, &current->bpf_flags) ||
    	    capable(cap);
        }
```

Calls to `bpf_capable()` then replace the various `capable(CAP_SYS_ADMIN)` (or sometimes `CAP_NET_ADMIN`) calls that currently protect access to BPF functionality. While the cover letter says that access is provided to "most of" the available BPF features, the patch appears to change every `capable()` call in the `kernel/bpf` directory. 

The end result of all this work is that a system administrator could, for example, create a new group called `bpf`; that group would be the group owner of the `/dev/bpf` file. The permissions on `/dev/bpf` would be set to allow group read access (write access is not required to make the `ioctl()` calls); thereafter, any process with membership in the `bpf` group would be able to use the `bpf()` system call. 

It's worth noting that most interesting things that can be done with BPF involve subsystems beyond the BPF virtual machine itself. Attaching a BPF program to a tracepoint requires the cooperation of the tracing code, for example, and using BPF programs in networking necessarily involves the networking subsystem. There are usually permission checks in those subsystems as well; tracepoint access requires the ability to call [`perf_event_open()`](<http://man7.org/linux/man-pages/man2/perf_event_open.2.html>), for example, which may be restricted depending on the system's configuration. This patch does not change those checks, with one exception: the restrictions on what can be done with BPF socket-filter programs are removed if the BPF capability has been turned on. 

In summary, what this patch is doing is creating a new capability bit that exists outside of the normal Linux capability mechanism, and which can be turned on or off by any process with read access to `/dev/bpf`. This new capability is recognized within the BPF subsystem, and in one place in the networking code; it seems highly likely that its use could expand to other parts of the kernel as well. This is a bit of a twist on the usual kernel security model. 

There are reasons why one might not want to just add another capability bit instead (`CAP_SYS_BPF`, say). Existing capability-aware programs would not know what the new bit means and may well mishandle it, for example. But it is not clear that creating what is essentially a capability bit in a separate guise improves on that situation. 

It seems likely that, at some point, somebody will want to be able to enable BPF functionality with finer-grained control. The good news is that the low-level machinery to do that is already there in the form of a set of Linux security module (LSM) hooks. Given the increasing use of LSMs to give administrators control over security policies in the kernel, it's perhaps surprising that an LSM-based approach was apparently not considered for this case. That could perhaps change as this patch set moves beyond the BPF community and is reviewed more widely.
