---
title: Seeking an API for protection keys supervisor
url: https://lwn.net/Articles/894531/
date: "May 11, 2022"
category: Memory protection keys
author: "By Jonathan Corbet May 11, 2022 LSFMM"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
May 11, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

Memory protection keys are a CPU feature that allows additional access restrictions to be imposed on regions of memory and changed in a fast and efficient way. Support for protection keys in user space has been in the kernel for some time, but kernel-side protection (often called "protection keys supervisor" or PKS) remains unsupported — on x86, at least. At the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM), Ira Weiny provided an update on the state of PKS and led a discussion on what the proper in-kernel API for PKS should be. 

Weiny began by saying that [version 10 of his PKS patch set](<https://lwn.net/ml/linux-kernel/20220419170649.1022246-1-ira.weiny@intel.com/>) had been posted in April. It adds additional protections for kernel-space mappings on x86 systems. On that architecture, memory protection keys control read and write access, but cannot affect execute access. The permissions set by PKS apply only on the local CPU; that means that they can changed quickly, with no need for expensive translation lookaside buffer (TLB) flushes. PKS protections can apply to persistent memory as well as normal RAM; the initial goal of Weiny's patch series is [![\[Ira Weiny\]](https://static.lwn.net/images/conf/2022/lsfmm/IraWeiny-sm.png)](<https://lwn.net/Articles/894545/>) to [use PKS to protect persistent memory](<https://lwn.net/Articles/883352/>) against stray writes from the kernel. There is also [a patch series from Rick Edgecombe](<https://lwn.net/ml/linux-kernel/20210830235927.6443-1-rick.p.edgecombe@intel.com/>) that uses PKS to protect page tables from corruption. 

Protecting memory from unwanted changes by the kernel is a good thing, but this protection cannot get in the way of legitimate changes. So PKS protections must be lifted when such changes are being made. Trying to find every site in the kernel where PKS-protected memory is being accessed would be futile, and the result would be unmaintainable, so Weiny has, instead, "abused" the `kmap()` interface for this purpose. But `kmap()` is not the best tool for the job, for a couple of reasons. 

The `kmap()` API was initially introduced to enable the kernel to manage (relatively) large amounts of memory on 32-bit processors. On such machines, there are not enough available address bits to directly map more than (usually) about 1GB of memory into the kernel's address space. The memory that _can_ be mapped this way was called "low memory", while all of the memory that could not be directly mapped was "high memory". When the kernel needs access to a page in high memory, it must first make a temporary mapping in its page tables; this is done with `kmap()`. In practice, this means that the kernel must call `kmap()` (or one of its variants) before accessing any page in memory, and call `kunmap()` when that access is complete. In cases where the target page is in low memory (that would be all pages on 64-bit systems), those calls do nothing, but they must still be present. 

Thus, `kmap()` would seem to be an ideal interface for adjusting PKS restrictions. When the kernel needs to access a page, it will call `kmap()`, which can suspend any PKS protections for the page in question on the local CPU; the following `kunmap()` call can then restore those protections. The only problem is that [high memory is going away](<https://lwn.net/Articles/813201/>) sooner or later, and the plan is for `kmap()` to be removed at the same time. We live in a 64-bit world now, Weiny said. There are still some Arm CPUs that need high memory now, he added, but the writing is on the wall for high memory in the longer term. 

It would thus be good to find an alternative to using `kmap()`. One apparent option is the `page_address()` macro, but that will not work due to the lack of an unmap operation. The problem needs to be solved, Weiny said; PKS is not the last protection scheme of this type that will come along. The kernel project needs to establish the rule that code cannot just access the direct map without making prior arrangements; he suggests simply redefining `kmap()` to fill this role. The new meaning of a `kmap()` call would be "give me a kernel-accessible address for this page". 

An alternative would be to improve `[vmap()](<https://elixir.bootlin.com/linux/v5.17.6/source/mm/vmalloc.c#L2736>)`, which creates a new kernel-space mapping for a page, for this purpose, though that would require changing a lot of `kmap()` calls. Matthew Wilcox said that it should be possible to make `vmap()` more efficient; there just has never been a driving need to optimize it so far. Weiny said that could make long-term mappings, for which `kmap()` is not intended, work better. Or, he said, the kernel could just eliminate the direct map entirely and always require memory to be mapped explicitly, but that approach probably would not perform well. 

Wilcox raised the issue of the ["Capability Hardware Enhanced RISC Instructions" (CHERI) architecture](<https://www.cl.cam.ac.uk/research/security/ctsrd/cheri/>), which applies capabilities to all of memory. It has a 128-bit address type that provides a lot of space for access keys and more. Only FreeBSD supports this architecture currently, but it is, he said, something that the Linux community should be thinking about; CHERI-like mechanisms seem likely to show up in other processors over time. Supporting PKS can be seen as a small step in preparing for that world. 

Josef Bacik said that the Btrfs code is currently a heavy user of `kmap()`, but he does not really care about the API to access kernel memory. "Just tell me what to use", he added. Chris Mason said that Btrfs developers have a debugging patch that makes pages read-only so that they can look at the resulting crashes and see who is modifying pages when they shouldn't be. PKS would be a useful way to implement this functionality. 

Wilcox suggested that there should be an interface that can change the protections on multiple pages. Some sort of `kmap_local_range()` function would be useful. Bacik agreed, saying that Btrfs often has to map 16KB metadata blocks. It would be easy, he said, to change over to a new API that did the job better. At that point time ran out and the session came to a close.
