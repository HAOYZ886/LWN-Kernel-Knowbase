---
title: No hardware memory isolation for BPF programs
url: https://lwn.net/Articles/1059218/
date: "February 25, 2026"
category: Memory protection keys
author: "By Daroc Alden February 25, 2026"
---

By **Daroc Alden**  
February 25, 2026

On February 12, Yeoreum Yun posted a [suggestion](<https://lwn.net/ml/all/aY3+Raf8eZqipCd6@e129823.arm.com/>) for an improvement to the security of the kernel's BPF implementation: use [ memory protection keys](<https://lwn.net/Articles/643797/>) to prevent unauthorized access to memory by BPF programs. Yun wanted to put the topic on the list for discussion at the Linux Storage, Filesystem, Memory Management, and BPF Summit in May, but the lack of engagement makes that unlikely. They also have a patch set implementing some of the proposed changes, but has not yet shared that with the mailing list. Yun's proposal does not seem likely to be accepted in its current form, but the kernel has [ added hardware-based hardening options](<https://lwn.net/Articles/517475/>) in the past, sometimes after substantial discussion. 

When a modern CPU needs to turn a virtual address into a physical address, it does so by consulting a [ page table](<https://en.wikipedia.org/wiki/Page_table>). This table also dictates whether the memory in question is readable, writable, executable, accessible by user space, etc. Page tables have a [ multi-level structure](<https://lwn.net/Articles/717293/>), requiring several pointer indirections to find the actual entry for a page of memory. To avoid the overhead of following these indirections on every memory access, the CPU keeps a cache of recently accessed entries called the [ translation lookaside buffer](<https://en.wikipedia.org/wiki/Translation_lookaside_buffer>) (TLB). 

When the kernel wants to change the access permissions of a given area of memory, it needs to update the page table and then flush the TLB (causing an inevitable performance hit as it refills). Worse, if the area of memory is large, it may need to update many page-table entries. Even just keeping track of which page-table entries need to be updated and iterating through them all can be a time-consuming operation. Memory protection keys are a hardware feature that helps avoid the overhead of changing large sections of the page table, making it practical to change the permissions of memory as part of a routine operation. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Memory protection keys use four bits in the page table to associate each page in memory with one of sixteen keys; there is a special CPU register that associates each key with read and write permissions. This avoids the need to actually change individual page-table entries: just change the permission bits for the corresponding key and the memory becomes inaccessible, without the need to walk the page table or flush the TLB. 

The kernel has had support for memory protection keys since 2016, but that support has only extended to user space. The [related system calls](<https://www.man7.org/linux/man-pages/man7/pkeys.7.html>) allow user-space applications to use memory protection keys to implement a faster version of [`mprotect()`](<https://www.man7.org/linux/man-pages/man2/mprotect.2.html>). There is no reason, in theory, that memory protection keys couldn't be used within the kernel as well. In practice, there have been [a number of attempts](<https://lwn.net/Kernel/Index/#Memory_protection_keys>) to integrate them into the kernel that have not come to fruition. 

Yun suggested adding a new set of `kmalloc_pkey()` and `vmalloc_pkey()` functions to allocate kernel objects in parts of memory protected by a key. That would make it practical to give BPF programs access to only a subset of kernel memory. Specifically, memory that is owned by BPF programs, or that a subsystem specifically intends to share with a BPF program, could be allocated with a separate memory protection key from other kernel allocations. Then, when entering BPF code, access to general kernel memory could be swiftly disabled. Yun's message described how this would work in some depth, but did not include any of the actual code to implement it — even though they intend to share their work-in-progress code soon — so it's hard to judge how invasive a change this would be. 

Dave Hansen [thought](<https://lwn.net/ml/all/d6d37293-592f-4109-90e9-20b77b023476@intel.com/>) that plan might be feasible for subsystems such as the scheduler that have a relatively limited amount of writable data, but that other areas of the kernel would have more problems: 

> Networking isn't my strong suit, but packet memory seems rather dynamically allocated and also needs to be written to by eBPF programs. I suspect anything that slows packet allocation down by even a few cycles is a non-starter. 
> 
> IMNHO, _any_ approach to solving this problem that starts with: we just need a new allocator or modification to existing kernel allocators to track a new memory type makes it a dead end. Or, best case, a very surgical, targeted solution. 

Alexei Starovoitov, on the other hand, [ did not just think the suggestion would be difficult to pull off](<https://lwn.net/ml/all/CAADnVQ+D7v+sGFFyTfSxFSEEcnGV111NeSaVtASYHfP-mDnV4w@mail.gmail.com/>), but also pointless. Yun had listed several CVEs from between 2020 and 2023 as a way of showing that the BPF verifier alone was not enough to ensure security. Starovoitov disagreed: ""None of them are security issues. They're just bugs."" 

Yun [agreed](<https://lwn.net/ml/all/aY4Vipl+4rGa+u6D@e129823.arm.com/>) that they were bugs, but disagreed that they have no security implications. Yun is of the opinion that the existence of bugs in the BPF verifier that have led to memory corruption in the past is enough justification to put another barrier between running a BPF program and having an exploitable vulnerability. 

Considering the previous unsuccessful attempts to use memory protection keys in the kernel and the difficulty of implementation, Yun's proposal to introduce memory protection keys to the BPF subsystem seems unlikely to go anywhere. On the other hand, the kernel has slowly been adding hardening measures such as [per-call-site slab caches](<https://lwn.net/Articles/986174/>) — perhaps associating these caches with memory protection keys is a logical next step. Only time will tell whether memory protection keys are a useful addition to the kernel's various hardening tools, or whether they're a cumbersome distraction. Either way, they will likely have to make their way into the kernel via some other subsystem.
