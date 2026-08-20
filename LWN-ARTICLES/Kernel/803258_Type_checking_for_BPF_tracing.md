---
title: Type checking for BPF tracing
url: https://lwn.net/Articles/803258/
date: "October 28, 2019"
category: "BPF-Tracing; Tracing-with BPF"
author: "By Jonathan Corbet October 28, 2019"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
October 28, 2019

The [BPF in-kernel virtual machine](<https://lwn.net/Articles/740157/>) has brought a new set of capabilities to a number of functional areas in the kernel, including, significantly, [tracing](<https://lwn.net/Articles/787131/>). Since BPF programs run in the kernel, much effort goes into ensuring that they will not cause problems for the running system; to that end, the BPF verifier checks every possible aspect of each BPF program's behavior to ensure that it is safe to run in the kernel — with one notable exception. With a patch set titled "[revolutionize bpf tracing](<https://lwn.net/ml/netdev/20191016032505.2089704-1-ast@kernel.org/>)", Alexei Starovoitov aims to close that loophole and eliminate a set of potential problems in a widely used class of BPF programs. 

BPF is heavily used in tracing applications to gain access to useful kernel information and to perform data aggregation in kernel space. There are two variants of these programs. If a tracepoint has been placed in a useful location in the kernel, a BPF program can be attached there; otherwise, a kprobe can be placed at (almost) any kernel location and used to trigger a BPF program. Either way, the BPF verifier currently has little visibility into the data that will be passed to those programs. 

Consider, for example, the `trace_kfree_skb` tracepoint placed in `[net_tx_action()](<https://elixir.bootlin.com/linux/v5.4-rc2/source/net/core/dev.c#L4575>)`. When this tracepoint triggers, any handlers (including attached BPF programs) will be passed two pointers, one to the `sk_buff` structure representing the network packet of interest, and one to the function that is freeing that packet. The type information associated with those pointers is lost, however; the program itself just sees a pair of 64-bit unsigned integers. Accessing the kernel data of interest requires casting those integers into pointers of the correct type, then using helpers like `bpf_probe_read()` to read the data behind those pointers. A series of `bpf_probe_read()` calls may be needed to walk through a data structure and get to the data the tracing program is actually looking for. 

The problem is that a BPF program can cast one of these values into any type it likes; the result need not correspond to the actual type of that data. A mistake could cause a BPF program to go off into the weeds; in one worst-case scenario, the program could wander into a memory-mapped I/O area and cause some real damage. This isn't generally a security issue, since tracing is a privileged operation to begin with, but it is a safety issue — exactly the sort of issue that the BPF verifier is meant to prevent. 

This problem has existed since the kernel first gained the ability to attach BPF programs to tracepoints and kprobes. Meanwhile, BPF developers have been working on [an entirely different problem](<https://lwn.net/Articles/773198/>): the lack of binary portability for BPF programs. These programs go digging around in kernel data structures, but the layout and content of those structures varies depending on the kernel configuration, the underlying architecture, and more. The data of interest to any given program may be located 12 bytes into a structure on one kernel, but only 8 bytes into that structure on a different kernel. Without the ability to "relocate" these references, BPF users must rebuild their programs on every target system. 

The "compile once run everywhere" effort has, over the last couple of years or so, worked to address this problem through the creation of a compact, machine-readable description of the kernel's data structures. This "BPF type format" (BTF) data is provided by the kernel itself, but it can be used by user-space support libraries to adjust a binary BPF program for a local kernel before loading it, mostly solving the binary portability issue. But it turns out that BTF information has other uses as well. 

In particular, it is possible to annotate tracepoints with information about the types of the data values passed to handler programs. That allows the verifier to ensure that those programs are working with the correct data types. It also makes it possible for the C handler programs to follow pointers directly; when those programs are compiled to BPF and loaded into the kernel, the verifier can implicitly substitute the `bpf_probe_read()` calls where they are needed — after performing the necessary type checking, of course. 

The end result of all this will be BPF tracepoint handler programs that are safer and far less error-prone to write. Whether tracing is "revolutionized" remains to be seen, but it is clearly improved in a significant way. 

What is decidedly not revolutionized is data access within kprobe handlers. A kprobe can be set anywhere in the running kernel, and it is given access to the contents of the processor registers when the probe is hit. It is not, at this point, possible for the verifier to know what will be in those registers at that time, so this kind of checking cannot be done. That means that, especially in the parts of the kernel that are not amenable to the addition of proper tracepoints, the use of BPF programs without this sort of type checking will have to continue. 

That said, progress is progress, and this work will increase the safety of much of the tracing code that is currently in use. It has been queued in the bpf-next tree so, barring some sort of last-minute hitch, it can be expected to show up in the 5.5 kernel.
