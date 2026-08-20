---
title: "Two address-space-isolation patches get closer"
url: https://lwn.net/Articles/835342/
date: "October 27, 2020"
category: "Memory management-Address-space isolation; Memory management-Virtualization; System calls-memfd secret"
author: "By Jonathan Corbet October 27, 2020"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
October 27, 2020

Address-space isolation is the technique of removing a range of memory from one or more address spaces as a way of preventing accidental or malicious access to that memory. Since the disclosure of the Meltdown and Spectre vulnerabilities, the kernel has used [one form of address-space isolation](<https://lwn.net/Articles/741878/>) to make kernel memory completely inaccessible to user-space processes, for example. There has been a steady level of interest in using similar techniques to protect memory in other contexts; two patches implementing new isolation mechanisms are getting closer to being ready for merging into the mainline kernel. 

#### `memfd_secret()`

The first of these is the [`memfd_secret()`](<https://lwn.net/ml/linux-kernel/20201026083752.13267-1-rppt@kernel.org/>) patch set from Mike Rapoport, which has been [covered here before](<https://lwn.net/Articles/812325/>), so this overview will be relatively brief; see that article for more background. The purpose of this work is to allow a user-space process to create a "secret" memory area that is as inaccessible as possible outside of the process. Intended users include cryptographic libraries, which can use a secret area to hold cryptographic keys and keep them safe from prying eyes. 

This functionality has, in recent revisions of the patch set, been moved into a separate system call: 

```
int memfd_secret(unsigned long flags);
```

The return value will be a file descriptor that can then be passed to [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>) to map an actual range of memory. For the most part, that memory will look (to the mapping process) like any other memory area, but there will be a couple of differences: 

  * Pages of memory in this range will be removed from the kernel's direct map — the portion of the address space that lets the kernel access (almost) any physical page in the system. This makes it much harder for the kernel to access this memory, either intentionally or by way of an exploit. 
  * If `flags` includes `SECRETMEM_UNCACHED`, then the memory will be mapped uncached if the underlying architecture supports it. Uncached memory will be far slower to access, but it is also immune to disclosure via many speculative-execution vulnerabilities. 

Memory in a secret area is locked into RAM and unable to be swapped. As such, it is counted against the owning process's locked-memory limit. 

One ongoing problem with features like this is that removal of pages from the kernel's direct map is an expensive operation. The direct map uses huge pages, minimizing its impact on the system's translation lookaside buffer (TLB). Removing random pages from the map breaks up those huge pages, significantly increasing TLB pressure. In order to minimize this impact, the `memfd_secret()` patch set maintains a separate cache of physically contiguous pages to use for this purpose. 

The rate of change for this patch set has been slowing for some time, so it may be close to being ready for inclusion. One never knows for sure with memory-management patches, though, until the patches are actually applied. 

#### KVM protected memory

While memory-disclosure vulnerabilities are unwanted on any system, the stakes are often higher on systems that are running virtualized guests. Such machines may be running workloads from unrelated groups that are unwilling to share their secrets with each other in ordinary circumstances; the possibility of sharing a physical system with a guest that is under the control of an attacker makes memory protection an even more urgent problem. As a way of hardening these systems, CPU vendors have been adding memory-encryption mechanisms that make guest memory inaccessible to the kernel and to other guests. These features have their own cost, though, and support in hardware is far from universal at this point. 

Kirill Shutemov has drawn an interesting conclusion from these technologies, though: the fact that systems using them still work means that access to that memory from the kernel or the hypervisor is not actually needed most of the time. So he has put together [a patch set](<https://lwn.net/ml/linux-kernel/20201020061859.18385-1-kirill.shutemov@linux.intel.com/>) that takes a fully software-based approach. Rather than encrypt guest memory, systems running this code just unmap it. Using this feature requires support on the part of both the kernel and the guest. 

Specifically, a KVM hypercall is added that allows guests to request that their memory be made inaccessible. The host kernel will respond by removing any memory allocated to the guest from the direct map, taking away its own ability to access that memory. In user space the approach is a bit different: any memory belonging to the guest remains mapped but is marked with `PROT_NONE` protections, again making it inaccessible. This will affect processes like the QEMU emulator, which will lose direct access to guest memory. The lack of mappings will naturally impede attacks coming from other guests as well. Within the guest, the guest kernel controls memory permissions as usual. 

The resulting isolation protects guest memory from unwanted access by way of vulnerabilities in components like the kernel or QEMU. It is not a complete protection, though; if the host kernel is compromised to the level of arbitrary code execution, it can remap the pages and pillage them at leisure. For the wide range of vulnerabilities that depend on getting the kernel to access a stray pointer — or speculative-execution vulnerabilities — though, this unmapping should significantly raise the bar for any exploit attempt. 

Of course, there are times when the kernel must access memory within guests to perform normal kernel functions. A second hypercall has been added for guests to indicate which memory they need to open up to the host kernel; those ranges will be mapped back into the host kernel's address space. DMA buffers for virtualized devices are one example of the type of memory that a guest would want to share with the host kernel in this way. 

This work looks interesting, but there are a number of loose ends that need to be tied down before it can be considered ready. Unlike `memfd_secret()`, this work has no mechanism for avoiding direct-map fragmentation as pages are removed; since the amount of memory involved is rather larger in this case, the fragmentation problems are likely to be that much more severe. Unmapped guest memory cannot be migrated, which defeats the kernel's mechanisms for defragmenting memory. That is likely to cause all sorts of problems over time; Shutemov has [acknowledged](<https://lwn.net/ml/linux-kernel/20201020125504.xadmnhpf3pu4uva7@black.fi.intel.com/>) that this problem will need to be fixed before the patches can be merged. It is also currently not possible to reboot a guest with protected memory; Shutemov has suggested that this case could just be declared "unsupported", an idea that has already [drawn complaints](<https://lwn.net/ml/linux-kernel/87ft6949x8.fsf@vitty.brq.redhat.com/>) in the discussion. 

The length of this list of issues implies that the KVM protected memory work is not something that will be seen in the mainline kernel in the near future. Both of these patch sets are a likely indicator of the direction things are going, though. Sharing as much as possible may improve performance, but it seems increasingly clear that the associated security problems are anything but easy to address. Separating address spaces as much as possible looks like a relatively straightforward way to sidestep many of those problems.
