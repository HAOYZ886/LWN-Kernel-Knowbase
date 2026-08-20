---
title: Memory protection keys for the kernel
url: https://lwn.net/Articles/826554/
date: "July 21, 2020"
category: "Memory protection keys; Security-Kernel hardening"
author: "By Jonathan Corbet July 21, 2020"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
July 21, 2020

The [memory protection keys](<https://lwn.net/Articles/695355/>) feature was added to the 4.6 kernel in 2016; it allows user space to group pages into "protection domains" that can have their access restricted independently of the normal page protections. There is no equivalent feature for kernel space; access to memory in the kernel's portion of the address space is controlled exclusively by the page protections. That situation may be about to change, though, as a result of the [protection keys supervisor (PKS) patch set](<https://lwn.net/ml/linux-kernel/20200717072056.73134-1-ira.weiny@intel.com/>) posted by Ira Weiny (with many patches written by Fenghua Yu). 

Virtual-memory systems maintain a set of protection bits in their page tables; those bits specify the types of accesses (read, write, or execute) that are allowed for a given processor mode. These protections are implemented by the hardware, and even the kernel cannot get around them without changing them first. On the face of it, the normal page protections would appear to be sufficient for the task of keeping the kernel away from pages that, for whatever reason, it should not be accessing. Those protections do indeed do the job in a number of places; for example, page protections prevent the kernel from writing to its own code. 

Page protections work less well, though, in situations where the kernel should be kept away from some memory _most_ of the time, but where occasional access must be allowed. Changing page protections is a relatively expensive operation involving tasks like [translation lookaside buffer](<https://en.wikipedia.org/wiki/Translation_lookaside_buffer>) invalidations; doing so frequently would hurt the performance of the kernel. Given that protecting memory from the kernel is usually done as a way of protecting against kernel bugs that, one hopes, do not normally exist anyway, that performance hit is one that few users are willing to pay. 

If memory could be protected from the kernel in a way that efficiently allows occasional access, though, there is likely to be more interest. That is the purpose of the PKS feature, which will be supported in future Intel CPUs. PKS associates a four-bit protection key with each page in the kernel's address space, thus allowing each of those pages to be independently assigned to one of sixteen zones. Each of those zones can be set to disallow write access from the kernel, or to disallow all access altogether. Changing those restrictions can be done much more quickly than changing the protections on those pages. 

The patch set adds a few new functions for management of protection keys, starting with the allocation and deallocation routines: 

```
int pks_key_alloc(const char * const pkey_user);
        void pks_key_free(int pkey);
```

A protection key is allocated with `pks_key_alloc()`; the `pkey_user` string only appears in an associated debugfs file. The return value will either be the key that has been allocated, or a negative error code if the allocation fails. A previously allocated key can be freed with `pks_key_free()`. 

Code using protection keys must be prepared for `pks_key_alloc()` to fail. This feature will not be available at all on most running systems for some time, so there may be no keys to allocate. Even when the hardware is available, there are only fifteen keys available for the entire kernel to use (since key zero is reserved as the default for all pages). If there is contention for keys, not every subsystem will succeed in allocating one. The good news is that failure to allocate a key just leaves the affected subsystem in the situation it's in today; everything still works, but the additional protection will not be available. 

Putting a specific page under the control of a given key is done by setting its (regular) protections in the usual ways and using the `PAGE_KERNEL_PKEY()` macro to set the appropriate bits in the protection field. Once the key has been set, there is no further need to modify the page's protections. When a key is first allocated, it will be set to disallow all access to any pages associated with that key. Changing the restrictions is done with: 

```
void pks_update_protection(int pkey, unsigned long protection);
```

Where `protection` is zero (to enable all access allowed by the ordinary page protections), `PKEY_DISABLE_WRITE`, or `PKEY_DISABLE_ACCESS`. This operation is relatively fast; one reason for that is that it only applies to the current thread. One thread running in the kernel can thus enable access to a specific zone without opening it up to kernel code running in other threads. 

One can think of a number of areas where this feature might be useful within the kernel. Protecting memory containing cryptographic keys from all access will raise the bar for any attacker trying to get at those keys. The initial focus for this patch set, though, is protecting device memory from stray writes. The kernel accesses memory found on peripheral devices by mapping that memory into its own address space; that makes access quick, but it also opens up a whole new range of potential problems should the kernel accidentally write to the wrong place. 

Kernel developers famously experienced this eventuality back in 2008, when writes to the wrong place [destroyed numerous e1000e network interface cards](<https://lwn.net/Articles/304105/>) before being tracked down. That problem was at least highly evident; users tend to notice when they can no longer connect to the net. The advent of [persistent memory](<https://en.wikipedia.org/wiki/Non-volatile_random-access_memory>), though, has raised the stakes on this kind of problem. This memory holds important user data; a stray write will corrupt that data in ways that may not be discovered for some time. Persistent memory can occupy terabytes of address space — something that a network interface is unlikely to do — so the target for misdirected writes is significantly larger. It would be undesirable for Linux to gain a reputation as the sort of system that occasionally trashes data in persistent memory, so an additional level of protection seems like a useful thing to have. 

The PKS patch set provides this protection by allocating a single protection key for all device memory. Device drivers wishing to opt into the protection provided by this key can do so by setting a new flag in the [`dev_pagemap` structure](<https://elixir.bootlin.com/linux/v5.7.8/source/include/linux/memremap.h#L94>) associated with the memory in question. This memory will be set up with writes disabled (but reads allowed); whenever the kernel needs to write to that memory, it will need to adjust the restrictions first. That can be done with a couple of helper functions: 

```
void dev_access_enable();
        void dev_access_disable()
```

Most of the time, though, those calls are not necessary. It is already a bug to access device memory without having first called `kmap()` (or one of its variants); the patch set enhances those functions to enable write access whenever a mapping obtained from `kmap()` exists for a protected region. Naturally, that means that, if a driver marks memory as being protected, calls `kmap()` on the memory, then keeps the mapping around forever, the extra access protection for all device memory using this mechanism will vanish. Hopefully, all users are calling `kmap_atomic()`, which requires mappings to be short-lived and is more efficient as well. 

While there seems to be a consensus that this feature is worth supporting in the kernel, there is still some ongoing discussion about various details of how it is implemented. It thus seems unlikely to be ready to be merged when the 5.9 merge window opens next month. PKS may well find its way into the kernel in a subsequent development cycle, though, making the kernel that much less likely to overwrite a persistent-memory device by mistake.
