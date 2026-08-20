---
title: "4K stacks for everyone?"
url: https://lwn.net/Articles/150580/
date: "September 7, 2005"
category: Kernel stack; NDISwrapper
author: 
---

The 2.6.6 kernel contained, among many other things, a patch implementing single-page (4K) kernel stacks on the x86 architecture. Cutting the kernel stack size in half reduces the kernel's per-process overhead and eliminates a major consumer of multi-page allocations. So running with the smaller stack size is good for kernel performance and robustness. The only problem has been certain code paths in the kernel which require more stack space than that. Overrunning the kernel stack will corrupt kernel memory and lead to unfortunate behavior in a hurry. 

Over time, however, most of these problems have been taken care of, to the point that Adrian Bunk recently [asked](<https://lwn.net/Articles/150582/>): is it time to eliminate the 8K stack option entirely for x86? Some distributors (e.g. Fedora) have been shipping kernels with 4K stacks for some time without ill effect. What problems might result, Adrian asked, if 4K stacks became the only option for everyone? 

It turns out that there are a few problems still. For example, the reiser4 filesystem still cannot work with 4K stacks. There is, however, a patch in the works which should take care of that particular problem. 

A more complicated issue comes up in certain complex storage configurations. If a system administrator builds a fancy set of RAID volumes involving the device mapper, network filesystems, etc., the path between the decision to write a block and the actual issuance of I/O can get quite long. This situation can lead to stack overflows in strange and unpredictable times. 

What happens here is that a filesystem will decide to write a block, which ends up creating a call to the relevant block driver's `make_request()` function (or the block subsystem's generic version of it). For stacked block devices, such as a RAID volume, that I/O request will be transformed into a new request for a different device, resulting in a new, recursive `make_request()` call. Once a few layers have been accumulated, the call path gets deep, and the stack eventually runs out. Neil Brown has posted [a patch](<https://lwn.net/Articles/150583/>) to resolve this problem by serializing recursive `make_request()` calls. With this patch, the kernel keeps an explicit stack of `bio` structures needing submission, and only processes one at a time in any given task. This patch will truncate the deep call paths, and should resolve the problem. 

That leaves one other problem outstanding: [NDISwrapper](<http://ndiswrapper.sourceforge.net/>). This code is a glue layer which allows Windows network drivers to be loaded into a Linux kernel; it is used by people who have network cards which are not otherwise supported by Linux. NDIS drivers, it seems, require larger stacks. Since they are closed-source drivers written for an operating system which makes larger stacks available, there is little chance of fixing them. So a few options have been discussed: 

  * Ignoring the problem. Since NDISwrapper is a means for loading proprietary drivers into the kernel - and Windows drivers at that - many kernel developers will happily refuse to support it at all. The fact is, however, that disallowing 8K stacks would break (formerly) working systems for many users, and there are kernel developers who do not want to do that. 

  * Hack NDISwrapper to maintain its own special stack, and to switch to that stack before calling into the Windows driver. This solution seems possible, but it is a nontrivial bit of hacking to make it work right. 

  * Move NDISwrapper into user space with some sort of mechanism for interrupt delivery and such. These mechanisms exist, so this solution should be entirely possible. 

No consensus solution seems to have emerged as of this writing. There is time, anyway; removing the 8K stack option is not a particularly urgent task, and certainly will not be considered for 2.6.14.
