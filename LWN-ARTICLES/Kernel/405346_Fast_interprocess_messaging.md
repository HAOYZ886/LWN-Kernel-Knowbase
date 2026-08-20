---
title: Fast interprocess messaging
url: https://lwn.net/Articles/405346/
date: "September 15, 2010"
category: "D-Bus; Message passing; Networking-D-Bus"
author: "By Jonathan Corbet September 15, 2010"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
September 15, 2010

As the number of cores in systems increases, the need for fast communications between processes running on those cores will also increase. This week has seen the posting of a couple of patches aimed at making interprocess messaging faster on Linux systems; both have the potential to significantly improve system performance. 

The first of these patches is motivated by a desire to make [MPI](<http://www.mcs.anl.gov/research/projects/mpi/>) faster. Intra-node communications in MPI are currently handled with shared memory, but that is still not fast enough for some users. Rather than copy messages through a shared segment, they would rather deliver messages directly into another process's address space. To this end, Christopher Yeoh has posted a patch implementing what he calls [cross memory attach](<http://lwn.net/Articles/405284/>). 

This patch implements a pair of new system calls: 

```
int copy_from_process(pid_t pid, unsigned long addr, unsigned long len,
                              char *buffer, int flags);
        int copy_to_process(pid_t pid, unsigned long addr, unsigned long len,
                            char *buffer, int flags);
```

A call to `copy_from_process()` will attempt to copy `len` bytes, starting at `addr` in the address space of the process identified by `pid` into the given `buffer`. The current implementation does not use the `flags` argument. As would be expected, `copy_to_process()` writes data into the target process's address space. Either both processes must have the same ownership or the copying process must have the `CAP_SYS_PTRACE` capability; otherwise the copy will not be allowed. 

The patch includes benchmark numbers showing significant improvement with a variety of different tests. The reaction to the concept was positive, though some problems with the specific patch have been pointed out. Ingo Molnar [suggested](<https://lwn.net/Articles/405352/>) that an iovec-based interface (like `readv()` and `writev()`) might be preferable; he also suggested naming the new system calls `sys_process_vm_read()` and `sys_process_vm_write()`. Nobody has expressed opposition to the idea, so we might just see these system calls in a future kernel. 

Many of us do not run MPI on our systems, but the use of [D-Bus](<http://www.freedesktop.org/wiki/Software/dbus>) is rather more common. D-Bus was not designed for performance in quite the same way as MPI, so its single-system operation is somewhat slower. There is a central daemon which routes all messages, so a message going from one process to another must pass through the kernel twice; it is also necessary to wake the D-Bus daemon in the middle. That's not ideal from a performance standpoint. 

Alban Crequy has [written about an alternative](<http://alban.apinc.org/blog/2010/09/15/d-bus-in-the-kernel-faster/>): performing D-Bus processing in the kernel. To that end, the "kdbus" kernel module introduces a new `AF_DBUS` socket type. These sockets behave much like the `AF_UNIX` variety, but the kernel listens in on the message traffic to learn about the names associated with every process on the "bus"; once it has that information recorded, it is able to deliver much of the D-Bus message traffic without involving the daemon (which still exists to handle things the kernel doesn't know what to do with). 

When the daemon can be shorted out, a message can be delivered with only one pass through the kernel and only one copy. Once again, significant performance improvements have been measured, even though larger messages must still be routed through the daemon. People have occasionally complained about the performance of D-Bus for years, so there may be real value in improving the system in this way. 

It may be some time, though, before this code lands on our desktops. There is [a git tree](<http://git.collabora.co.uk/?p=user/alban/linux-2.6.35.y/.git;a=shortlog;h=refs/heads/kdbus>) available with the patches, but they have never been cleaned up and posted to the lists for review. The patch set is not small, so chances are good that there will be a lot of things to fix before it can be considered for mainline inclusion. The D-Bus daemon, it seems, will be busy for a little while yet.
