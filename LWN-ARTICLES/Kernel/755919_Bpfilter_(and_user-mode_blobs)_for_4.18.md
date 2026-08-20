---
title: "Bpfilter (and user-mode blobs) for 4.18"
url: https://lwn.net/Articles/755919/
date: "May 30, 2018"
category: "BPF-bpfilter; Modules-ELF modules; Networking-Packet filtering"
author: "By Jonathan Corbet May 30, 2018"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 30, 2018

In February, the [bpfilter mechanism](<https://lwn.net/Articles/747551/>) was first posted to the mailing lists. Bpfilter is meant to be a replacement for the current in-kernel firewall/packet-filtering code. It provides little functionality itself; instead, it creates a set of hooks that can run BPF programs to make the packet-filtering decisions. [A version of that patch set](<https://lwn.net/Articles/755165/>) has been merged into the net-next tree for 4.18. It will not be replacing any existing packet filters in its current form, but it does feature a significant change to one of its more controversial features: the new user-mode helper mechanism. 

The core motivation behind bpfilter is performance. An in-kernel, general-purpose packet filter must necessarily offer a wide range of features; any given site will probably only use a small subset of those features. The result is a lot of unused code and time spent checking for whether a given feature is in use, slowing the whole thing down. A packet-filtering configuration expressed as a BPF program, instead, contains only the code needed to implement the desired policy. Once that code is translated to native code by the just-in-time compiler, it should be both compact and fast. The networking developers hope that it will be fast enough to win back some of the users who have moved to proprietary user-space filtering implementations. 

If bpfilter is to replace netfilter, though, it must provide ABI compatibility so that existing configurations continue to work. To that end, the bpfilter developers intend to implement the current netfilter configuration protocol; bpfilter will accept iptables rules and compile them to BPF transparently. That compilation is not a trivial task, though, and one that could present some security challenges, so the desire is to do it in user space, but under kernel control. 

To make that possible, the initial proposal included a new type of kernel module. Rather than containing kernel code, it contained a normal ELF executable that would be run as a special type of kernel thread. Using the module mechanism allowed this code to be packaged and built with the rest of the kernel; user-mode modules could also be subjected to the same signing rules. There were a number of concerns about how these modules worked, though, which led to some significant changes this time around. 

For example, the user-mode helper code is no longer packaged as a module. It is, instead, a blob of code that is built into a normal kernel subsystem (which may be built into the kernel image or packaged as a module). In the sample code, the user-mode component is built as a separate program then, in a process involving ""quite a bit of objcopy and Makefile magic"", it is turned into an ordinary object file that can be linked into the `bpfilter.ko` kernel module. 

Kernel code that wants to run a blob of code in user space will do so using the new helper code. That is done by calling: 

```
int fork_usermode_blob(void *data, size_t len, struct umh_info *info);
```

where `data` points to the code to be run, and `len` is the length of that code in bytes. The `info` structure is defined as: 

```
struct umh_info {
    	struct file *pipe_to_umh;
    	struct file *pipe_from_umh;
    	pid_t pid;
        };
```

Assuming the user-space process is successfully created, the kernel will place its process ID into `pid`. The kernel will also create a pair of pipes for communicating with the new process; they will be stored in `pipe_to_umh` (for writing) and `pipe_from_umh` (for reading). The code itself is copied into a tmpfs file and executed from there; that allows it to be paged if needed. If the built-in copy of the code is marked as "initdata" (and thus placed in a different section), the caller can free it once the helper is running. 

Kernel code that creates this type of helper process must take care to clean it up when the time comes. The process ID can be used to kill the process, and the pipes need to be closed. 

The bpfilter module itself, as found in 4.18, does not actually do much. It creates the helper process and can pass a couple of no-op commands to it, but there is no packet-filtering machinery in place yet. That code exists (and has been [posted](<https://lwn.net/Articles/753460/>) recently) but has evidently been held back to give the user-mode helper mechanism a cycle to stabilize. Bpfilter is thus starting small, but it may have a big impact in the end; perhaps that's why Dave Miller said ""let the madness begin"" as he [merged](<https://lwn.net/Articles/755933/>) the code for 4.18. 

The replacement of netfilter, even if it happens as expected, will take years to play out, but we may see a number of interesting uses of the new user-mode helper mechanism before then. The kernel has just gained a way to easily sandbox code that is carrying out complex tasks and which does not need to be running in a privileged mode; it doesn't take much effort to think of other settings where this ability could be used to isolate scary code. Just be careful not to call the result a "microkernel" or people might get upset.
