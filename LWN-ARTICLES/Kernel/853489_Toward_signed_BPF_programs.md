---
title: Toward signed BPF programs
url: https://lwn.net/Articles/853489/
date: "April 22, 2021"
category: "BPF; Modules-Signed"
author: "By Jonathan Corbet April 22, 2021"
---

By **Jonathan Corbet**  
April 22, 2021

The kernel's [BPF virtual machine](<https://lwn.net/Articles/740157/>) is versatile; it is possible to load BPF programs into the kernel to carry out a large (and growing) set of tasks. The growing body of BPF code can reasonably be thought of as kernel code in its own right. But, while the kernel can check signatures on loadable modules and prevent the loading of modules that are not properly signed, there is no such mechanism for BPF programs; any sufficiently privileged process can load any program that will pass the verifier. One might think that adding this checking for BPF would be straightforward, but that subsystem has some unique characteristics that make things more challenging than one might expect. There may be a solution in the works, though; fittingly, it works by loading yet another BPF program. 

Loadable kernel modules are stored as executable images in the ELF format. When one is loaded, the kernel parses that format and does the work needed to enable the module to run within the kernel; this work includes allocating memory for variables, performing relocations, resolving symbols, and more. All of the necessary information exists within the ELF file. Applying a signature to that file is simply a matter of checksumming the relevant sections and signing the result. 

BPF programs have similar needs, but the organization of the requisite information is a bit more, for lack of a better word, messy. The code itself is compiled as an executable section that is then linked into a loader program that runs in user space and invokes the [`bpf()`](<https://man7.org/linux/man-pages/man2/bpf.2.html>) system call to load the BPF program into memory. But BPF programs, too, need to have data areas allocated in the form of BPF maps, and they need relocations (of a sort) applied to be able to cope with different structure layouts on different systems. The necessary maps are "declared" as special ELF sections in the loader program; the [libbpf](<https://github.com/libbpf/libbpf>) library finds those sections and turns them into more `bpf()` calls. The BPF program itself is then modified (before loading into the kernel) so that it can find its maps when it runs. 

This structure poses a challenge for anybody wanting to implement signed BPF programs. The maps are a part of the program itself; if they are not established as intended, a BPF program might misbehave in interesting ways. But the kernel has no way to enforce any specific map configuration, and thus cannot ensure that a signed BPF program has been properly set up. Additionally, the need to modify the BPF program itself will break signature verification; after all, modifications to BPF programs are just the sort of thing this mechanism is expected to prevent. So, somehow, the kernel has to take a more active role in the loading of BPF programs. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

#### In-kernel BPF loading

The old-timers among us will remember that, once upon a time, the kernel's module loader lived in user space. Moving it into the kernel was one of many causes of extended pain during the 2.5 development cycle; 20 years later, some developers still hold a grudge against Rusty Russell for that experience. But those problems are long past and the in-kernel loader has long since ceased to create problems. So one might logically expect that moving the user-space BPF loader into the kernel would be a sensible approach to take. 

According to Alexei Starovoitov in [the cover letter](<https://lwn.net/ml/bpf/20210417033224.8063-1-alexei.starovoitov@gmail.com/>) to a new patch set, that approach has been tried in a couple of forms and ""discarded after months of work"". Evidently an attempt was made to move libbpf into the kernel; it is not entirely surprising that this complex body of code did not fit comfortably there. Another idea was to create a new executable file format that contained, in essence, a series of system calls needed to set up a specific BPF program. 

The problems that were encountered while implementing that second approach are not spelled out. But the third approach, found in Starovoitov's patch set, can be thought of as a variant on that idea. Rather than have the kernel step through a series of system calls, though, user space loads a special BPF program that does that work instead. 

Specifically, the patch set creates yet another type of BPF program — one that exists to execute system calls. This program will run in the context of the process that runs it, and will be limited to a small set of system calls; only `bpf()` and `close()` are allowed in the proposed patch set. This program will be expected to make the necessary set of `bpf()` calls to load and set up the BPF program that the user really wants to run. 

The generation of this "loader program" is done by watching what libbpf does to load the BPF program of interest and capturing each of the resulting `bpf()` calls. Those calls are then collected to generate the loader program, which will reproduce that work, from within the kernel, at the right time. So the kernel is, indeed, stepping through a canned series of system calls to load the program; it's just that this series is formatted as a BPF program in its own right. 

The problem of patching the BPF program to find its maps is addressed in the usual way: adding another layer of indirection. An array of file descriptors is set up, and the BPF program references maps by way of that array. When the program is loaded, this array can be populated with the actual file descriptors corresponding to the necessary maps. 

#### Next steps

As Starovoitov noted in the cover letter, this work is not yet a complete solution to the problem; it is a first step to show the direction that this work is taking. A big remaining piece is the offset relocations needed for BPF programs to access structure fields in a configuration-independent way. These relocations, too, require changing the BPF program text, so the solution may not be entirely trivial; more indirection-based schemes run the risk of imposing more of a performance cost than some users may want to pay. 

There is also, of course, the little matter of signing BPF programs and checking those signatures; this problem is not addressed in this patch set either. There is a brief mention of putting together a skeleton that would allow BPF programs to be packaged into a kernel module; if that were done, then the existing system for checking module signatures could be used for BPF programs as well. 

At a first glance, the BPF loader looks like a bit of a convoluted solution to the problem. It is worth noting, though, that this mechanism is not all that far removed from what happens in user space, where running a program usually involves launching [`ld.so`](<https://man7.org/linux/man-pages/man8/ld.so.8.html>) to assemble the required pieces for that program to run. So there are well-established precedents to this sort of solution. Whether this design will make it into the mainline kernel is yet to be seen, though; this work is young and has not yet seen much review.
