---
title: System calls for memory protection keys
url: https://lwn.net/Articles/689395/
date: "June 2, 2016"
category: Memory protection keys
author: "By Jonathan Corbet June 2, 2016"
---

By **Jonathan Corbet**  
June 2, 2016

"Memory protection keys" are an Intel processor feature that is making its first appearance in Skylake server CPUs. They are a user-controllable, coarse-grained protection mechanism, allowing a program to deny certain types of access to ranges of memory. LWN last [looked at kernel support](<https://lwn.net/Articles/667156/>) for memory protection keys (or "pkeys") at the end of 2015. The system-call interface is now deemed to be in its final form, and there is a push to stage it for merging during the 4.8 development cycle. So the time seems right for a look at how this feature will be used on Linux systems. 

A pkey is a four-bit value (in the current Intel implementation) that can be stored in the page-table entry for each page in a process's address space. Pages can thus be arbitrarily assigned to one of sixteen key values; each address space has its own set of keys. For each of those keys, the process can configure the CPU to deny either write operations or all access entirely. Pkeys will override the regular protections assigned to each page but, since they can only deny operations, their effect will always be to restrict access more strictly than the page protections do. There are a number of intended use cases, including the implementation of execute-only memory or the protection of sensitive data (cryptographic keys, for example) when it is not in active use. 

Most pkey operations are unprivileged and thus could be left to user space to handle without kernel involvement; the one exception is storing the key values in the page-table entries. There is value in having the kernel take an overall role in coordinating the use of pkeys, though, so that library code can use them without interfering with the rest of the application. The kernel can also make good use of pkeys if it knows it has exclusive access to them. To make all this possible, [five system calls](<https://lwn.net/Articles/689178/>) have been defined for working with pkeys in applications. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

#### The proposed pkey API

To avoid conflicts over the use of any specific key, pkeys should be allocated prior to use. The allocation system calls are: 

```
int pkey_alloc(unsigned long flags, unsigned long initial_rights);
        int pkey_free(int key);
```

A new protection key may be obtained with `pkey_alloc()`. In the current implementation, the `flags` argument must be zero, while `initial_rights` is a bitmask setting the key's initial access restrictions. The available access bits are `PKEY_DISABLE_WRITE` (disabling write access) or `PKEY_DISABLE_ACCESS` (which disables all access). It is worth noting that these flags refer to _data_ accesses; memory with a `PKEY_DISABLE_ACCESS` pkey can still be read by the processor for execute access. 

The return value from `pkey_alloc()` is an integer index indicating which key was allocated, or `ENOSPC` if no keys are available. Keys which are no longer in use may be freed with `pkey_free()`. Freeing a key does not, however, remove that key value from page-table entries or remove any restrictions that had been applied to that key. So surprising things could happen if an application frees a key that is still applied to pages within its address space and the key is later reallocated to another use. 

The assigning of keys to pages is done with a new variant of the `mprotect()` system call: 

```
int pkey_mprotect(void *start, size_t len, int prot, int pkey);
```

This call behaves like `mprotect()` in that it will set the (regular) protection bits described by `prot` on the pages containing `len` bytes beginning at `start`. It will also assign the given `pkey` (which must have been allocated with `pkey_alloc()`) to those pages. A call to `pkey_mprotect()` will succeed on systems that do not support pkeys, but only if `pkey` is passed as zero. 

If an application wants to ensure that a given memory range will never be accessible without the desired pkey restrictions, it can create that range by passing `PROT_NONE` to `mmap()`, making the memory initially inaccessible. A subsequent `pkey_mprotect()` call will then atomically change the protections and assign the pkey, ensuring that there is never a window where the restrictions are not as desired. 

An application can query the current restrictions associated with a pkey using the `RDPKRU` instruction, and change them with `WRPKRU`, so there is not strictly a need for the kernel to support these operations. The kernel provides a couple of system calls for manipulating pkey restrictions anyway: 

```
unsigned long pkey_get(int pkey);
        int pkey_set(int pkey, unsigned long access_rights);
```

These functions eliminate the need to use special assembly instructions in application code; they can also verify that the given `pkey` has been allocated. 

#### Execute-only interactions

There can be some security benefits from designating memory that contains code as execute-only, so that its contents cannot be read for other purposes. As it happens, though, setting the page protections to `PROT_EXEC` does not have that effect — the affected pages are still readable. So, on current processors, true execute-only protections are not easily achievable. But, as mentioned above, the `PKEY_DISABLE_ACCESS` restriction does not block execute access. It can thus be used, in conjunction with `PROT_EXEC`, to create execute-only memory ranges. 

While the system-call API is still out-of-tree, the core support for pkeys has been in the mainline kernel since the 4.6 release. If the kernel sees an `mprotect()` call setting `PROT_EXEC` permissions on a range of memory, it will automatically use a pkey to create true execute-only permissions. This is one of the reasons why it is useful to have the kernel in control of key allocation. 

There is an interesting question that comes up, though: what if a process sets a pkey of its own with `pkey_mprotect()`, then uses a regular `mprotect()` call to set the page permissions to `PROT_EXEC`? In this case, the kernel could either change the restrictions for the assigned pkey, or it could change the affected pages to use its own reserved pkey. Either approach could lead to results that the application developer finds surprising. 

To avoid such surprises, one pkey (number zero) has been set aside as the default key for all pages. This key will never be allocated with `pkey_alloc()`, and its restrictions cannot be changed with `pkey_set()`. As of 4.8 (assuming these patches are merged), the kernel will only assign the execute-only pkey to pages that are currently controlled by the default key. 

The memory protection keys patches have been circulating for some time, and have evolved considerably in response to reviewer comments. At this point, they would appear to have reached a stable point where the developers who are paying attention are happy with them. So the chances are good that the 4.8 kernel will include these system calls making the full functionality available to applications. How soon the requisite hardware will be widely available is yet to be seen, though.
