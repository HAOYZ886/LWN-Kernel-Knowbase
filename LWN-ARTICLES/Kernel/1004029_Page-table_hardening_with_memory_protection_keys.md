---
title: "Page-table hardening with memory protection keys"
url: https://lwn.net/Articles/1004029/
date: "January 9, 2025"
category: Memory protection keys
author: "By Jonathan Corbet January 9, 2025"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
January 9, 2025

Attacks on the kernel can take many forms; one popular exploitation path is to find a way to overwrite some memory with attacker-supplied data. If the right memory can be targeted, one well-targeted stray write is all that is needed to take control of the system. Since the system's page tables regulate access to memory, they are an attractive target for this type of attack. [This patch set](<https://lwn.net/ml/all/20250108103250.3188419-1-kevin.brodsky@arm.com>) from Kevin Brodsky is an attempt to protect page tables (and, eventually, other data structures) using the "memory protection keys" feature provided by a number of CPU architectures. 

Memory protection keys are an additional access-permission mechanism that is layered on top of the permissions implemented in the page tables. Memory can be partitioned into a relatively small number (eight or 16, typically) of domains (or "keys"). A key, in the sense used here, is simply a small integer value that has a set of memory-access permissions associated with it. Each page has an assigned key that can be used to impose additional access restrictions. Memory that is nominally writable cannot be written if its key denies that access. The permissions associated with a key can be changed quickly and affect all pages marked with that key; as a result, large swaths of memory can be quickly made accessible or inaccessible at any time. 

Changing the permissions associated with a key is an unprivileged operation. Memory protection keys, thus, cannot protect against attackers who are able to execute arbitrary code. They _can_ , though, be useful to protect against unintended access. Critical data can be write-protected using a key, with that key's permissions being briefly changed only when that data must be written. An attacker attempting to overwrite the same data, perhaps through exploitation of a use-after-free vulnerability, will be blocked, making the system that much harder to compromise. Similarly, memory containing sensitive data (cryptographic keys, for example) can be assigned a key that, most of the time, allows no access at all, reducing the likelihood that this data will be leaked to an attacker. 

Linux first gained [support for memory protection keys](<https://docs.kernel.org/core-api/protection-keys.html>) with the 4.6 kernel release in 2016. That support is available for 64-bit Arm and x86 systems, but only for user space. Some attempts over the years notwithstanding, memory protection keys have never been used to protect memory in kernel space, despite the fact that the CPUs support that functionality. 

Brodsky's patch set is an attempt to change that situation by using memory protection keys to regulate access to page tables on 64-bit Arm systems. Page tables were chosen for protection because of their value as a target, but also because access to them is already well confined to a set of helper functions, making it relatively easy to add the necessary hooks to change the key protections for a brief period when page tables need to be modified. 

A recurring concern with memory protection keys is their relatively small number; it is generally expected that there will be demand for more keys than the hardware can provide, though there has been little evidence of that happening so far. The user-space interface added a set of system calls, including [`pkey_alloc()`](<https://man7.org/linux/man-pages/man2/pkey_free.2.html>), which is used to allocate a new key. On the kernel side, though, there may be no need for a general allocation mechanism; the kernel's code is all present in the repository, so keys can be assigned statically, at least for now. 

The patch set does add a bit of structure, though, in the form of a concept called "kpkeys levels". Each level allows access to specific regions of memory. The intent would appear to be that access grows monotonically as the level increases, but there is nothing in the code that implements a hierarchy of levels; each level can be independent of the others. Since this is the first use of this mechanism, there are only two levels implemented: `KPKEYS_LVL_DEFAULT`, which provides access to kernel-space memory that is not further protected, and `KPKEYS_LVL_PGTABLES`, which enables write access to page tables. 

This abstraction might seem like more than is really needed in this case, where one could simply assign a key for page-table pages and be done with it. Brodsky appears to be looking forward to future applications where more complex combinations of permissions are needed. Separating levels from specific keys also makes it possible for multiple levels to use the same key, which could be useful if the available keys are oversubscribed someday. 

The interface to kpkeys levels, from the point of view of most kernel code, is fairly simple; there are two new functions: 

```
u64 kpkeys_set_level(int level);
        void kpkeys_restore_pkey_reg(u64 pkey_reg);
```

A call to `kpkeys_set_level()` will set the current kpkeys level, enabling whatever access that level provides. The return value is an architecture-specific representation of the state of key permissions prior to the change (not the previous level, since other code may be using some of the keys outside of the kpkeys levels mechanism). The previous protections can be restored by passing that returned value to `kpkeys_restore_pkey_reg()`. 

The page-table protection API is layered on top of the kpkeys levels machinery. It causes page-table pages to be assigned to the memory protection key set aside for page tables; by default, the associated protections do not allow writing to those pages. Any code that must modify a page-table page should first enable access with a call to `kpkeys_set_level()` setting the level to `KPKEYS_LVL_PGTABLES`, then use `kpkeys_restore_pkreg_reg()` to remove that access afterward. The easier and safer way, though, is to use a [scope-based guard](<https://lwn.net/Articles/934679/>): 

```
guard(kpkeys_hardened_pgtables)();
```

This bit of magic will make page tables writable and ensure that the change is undone as soon as the current function returns, making it impossible to forget to restore the page-table protections regardless of which code path is taken. 

The current implementation, Brodsky said, ""should be considered a proof of concept only"". It includes just enough support for Arm's kernel-space memory protection keys feature ("Permission Overlay Extension" or POE) to make the rest work; it is not intended to be a complete kernel-space POE implementation at this point. There are also no benchmark results showing what the impact of this mechanism is on performance; developers will want to see those measurements eventually. 

As it happens, this is not the first attempt to protect page-table pages using memory protection keys; Rick Edgecombe posted [an x86 patch set](<https://lwn.net/ml/all/20210830235927.6443-1-rick.p.edgecombe@intel.com/>) back in 2021. There was also [an attempt](<https://lwn.net/Articles/883352/>) to use memory protection keys to prevent stray writes to persistent memory by Ira Weiny in 2022. Neither series progressed to the point of being merged into the mainline, and Edgecombe eventually [set the page-table work aside](<https://lwn.net/ml/all/3b3c941f1fb69d67706457a30cecc96bfde57353.camel@intel.com/>) in favor of other projects. 

It seems clear, though, that there is interest in providing this sort of protection for page-table pages. To be successful, a patch set will almost certainly need to incorporate elements from both the Arm and x86 work to show that it is, indeed, applicable to more than one architecture. If that barrier can be overcome, the kernel might eventually have hardening of page-table access. Thereafter, it may make sense to extend this protection to other critical data structures within the kernel (Brodsky suggests task credentials and SELinux state, among other things). First, though, there needs to be agreement on the core infrastructure, and that discussion has barely begun.
