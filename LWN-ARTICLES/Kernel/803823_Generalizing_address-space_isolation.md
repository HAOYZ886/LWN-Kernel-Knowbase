---
title: "Generalizing address-space isolation"
url: https://lwn.net/Articles/803823/
date: "November 5, 2019"
category: "Memory management-Address-space isolation; Security-Meltdown and Spectre"
author: "By Jonathan Corbet November 5, 2019 OSS EU"
---

By **Jonathan Corbet**  
November 5, 2019

* * *

[OSS EU](<https://lwn.net/Archives/ConferenceByYear/#2019-Open_Source_Summit_Europe>)

Linux systems have traditionally run with a single address space that is shared by user and kernel space. That changed with the advent of the Meltdown vulnerability, which forced the merging of [kernel page-table isolation](<https://lwn.net/Articles/741878/>) (KPTI) at the end of 2017\. But, Mike Rapoport said during his [2019 Open Source Summit Europe](<https://events19.linuxfoundation.org/events/open-source-summit-europe-2019/>) talk, that may not be the end of the story for address-space isolation. There is a good case to be made for increasing the separation of address spaces, but implementing that may require some fundamental changes in how kernel memory management works. 

Currently, Linux systems still use a single address space, at least when they are running in kernel mode. It is efficient and convenient to have everything visible, but there are security benefits to be had from splitting the address space apart. Memory that is not actually mapped is a lot harder for an attacker to get at. The first step in that direction was KPTI. It has performance costs, especially around transitions between user and kernel space, but there was no other option that would address the Meltdown problem. For many, that's all the address-space isolation they would like to see, but that hasn't stopped Rapoport from working to expand its use. 

#### Beyond KPTI

Recently, he tried to extend this idea with [system-call address-space isolation](<https://lwn.net/ml/linux-kernel/1556228754-12996-1-git-send-email-rppt@linux.ibm.com/>), which implemented a restricted address space for system calls. When a system call is invoked, most of the code and data space within the kernel is initially unmapped and inaccessible; any access to that space will generate a page fault. The kernel can then check to determine whether it thinks the access is safe; if so, the address in question is mapped, otherwise the calling process will be killed. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

There are potentially a few use cases for this kind of protection, but the immediate objective was to defend against [return-oriented programming (ROP)](<https://en.wikipedia.org/wiki/Return-oriented_programming>) attacks. If the target of a jump matches a known [![\[Mike Rapoport\]](https://static.lwn.net/images/conf/2019/osse/MikeRapoport-sm.jpg)](<https://lwn.net/Articles/803830/>) symbol in the kernel, the jump can be considered safe and allowed to proceed; the page containing that address will be mapped for the duration of the call. ROP attacks work by jumping into code that is _not_ associated with a kernel symbol, so most of them would be blocked by this mechanism. Mapping the occasional page for safe references will make some code available to ROP attacks again, but it will still be a fraction of the entire kernel text (which is available in current kernels). 

These patches have not been merged, though, for a number of reasons. One is that nobody really knows how to check data accesses for safety; the known-symbol test used for text is not applicable to data. A system call with invalid parameters can still result in mapping a fair amount of code, making ROP gadgets available to an attacker. This patch also slowed execution considerably, which always makes acceptance harder. 

The [L1TF](<https://lwn.net/Articles/762570/>) and [MDS](<https://lwn.net/Articles/788522/>) speculative-execution vulnerabilities bring some challenges of their own. In particular, they allow a host system to be attacked from guests, and are thus frightening to cloud providers. The defense, for now, is to disable hyperthreading, but that can have a significant performance cost. A better solution, Rapoport said, might be another form of address-space isolation; in this case, it would be a special kernel mapping used whenever control passes into the kernel from a guest system. This "KVM isolation" mechanism was [posted by Alexandre Chartre](<https://lwn.net/ml/linux-kernel/1557758315-12667-1-git-send-email-alexandre.chartre@oracle.com/>) in May, but has not been merged. 

Other address-space isolation ideas are also circulating. One of these would be to map specific kernel data only for the process that needs to access it. That would be done by setting up a private range in the kernel page tables. Kernel code could allocate memory in this space with new functions like `kmalloc_proclocal()`. For extra isolation, memory allocated in this way would be removed from the kernel's "direct map", which is a linear mapping of all of the system's physical memory. Taking pages out of the direct map has its own performance issues, though, since it requires breaking up huge pages into small pages. 

Then, there are user-exclusive mappings — user-space mappings that are only visible to the owning process. These could be used to store secrets (cryptographic keys, for example) in memory where they could not be (easily) accessed from outside. Once again, this memory would be removed from the direct map; it would also not be swappable. The [`MAP_EXCLUSIVE`](<https://lwn.net/ml/linux-kernel/1572171452-7958-1-git-send-email-rppt@kernel.org/>) patch series implementing this behavior was posted recently. 

Finally, Rapoport also mentioned namespace-based isolation: kernel memory that is tied to a specific namespace and which is only visible to processes running within that namespace. This turns out to get tricky when dealing with network namespaces, though. The `sk_buff` structures used to represent packets would be obvious candidates for isolation, but they also often cross namespace boundaries. 

#### Generalizing address-space isolation

While each of the address-space isolation mechanisms described above is different, there are some common factors between all of them. They are all concerned with creating a restricted address space from existing memory, then making this space available when entering the proper execution context. So Rapoport is working on an in-kernel API to support address-space isolation mechanisms in general. That is going to require some interesting changes, though. 

The kernel's memory-management code currently assumes that the [`mm_struct`](<https://elixir.bootlin.com/linux/v5.4-rc5/source/include/linux/mm_types.h#L365>) structure attached to a process is equivalent to that process's page tables, but that connection will need to be broken. A new `pg_table` structure will need to be created to represent page tables; there will also be an associated API to manipulate these page tables. A particular challenge will be the creation of a mechanism that can safely free kernel page tables. 

Creating the restricted contexts is, instead, relatively easy. Some, like KPTI, are set up at boot time; others will need to be established at the right time: process creation, association with a namespace, etc. The context-switch code will need to be able to switch between restricted address spaces; again, switching the kernel's page tables is likely to be tricky. There will need to be code to free these restricted address spaces as well, with appropriate care taken to avoid the inconvenience that would result from, say, freeing the main kernel page tables. 

Once the infrastructure is in place, the kernel will need to gain support for private memory allocations. Functions like `alloc_page()` and `kmalloc()` will need to gain awareness of the context into which memory is being allocated; there will be a new `__GFP_EXCLUSIVE` flag to request an allocation into a restricted context. Once again, pages so allocated will need to be removed from the kernel's direct mapping (and return once they are freed). Extra care will need to be taken with objects that need to cross context boundaries. 

Finally, the slab caches will also need to be enhanced to support this behavior. Some of the necessary mechanism is already there in the form of the caching used by the memory controller. Slab memory is often freed from a context other than the one in which it was allocated, though, leading to a number of potential traps. 

Rapoport concluded by stating that address-space isolation needs to be investigated; it offers a way of significantly reducing the kernel's attack surface, even in the presence of hardware bugs. Whether the security gained this way justifies the extra complexity required to implement it is something that will have to be evaluated as the patches take shape. Expect to see some interesting patches on the mailing lists in the near future as this work is developed. 

[Thanks to the Linux Foundation, LWN's travel sponsor, for supporting your editor's travel to the event.]
