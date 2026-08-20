---
title: Memory protection keys v5
url: https://lwn.net/Articles/667156/
date: "December 9, 2015"
category: "Memory protection keys; Security-Security technologies"
author: "By Jonathan Corbet December 9, 2015"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
December 9, 2015

The first [memory protection keys patch set](<https://lwn.net/Articles/643797/>) showed up in May; it adds support for an upcoming feature in high-end Intel processors. This mechanism allows applications to assign an integer key value to each of their pages; each key has associated with it a protection mask that can deny access regardless of what the regular protection bits say. The feature is useful as a way to quickly change the restrictions applying to large ranges of memory without having to change each page's protection bits independently. Protection keys can support "read-mostly" memory, for example, or make memory containing sensitive information (cryptographic keys) entirely inaccessible most of the time. 

The [fifth version of the memory protection keys patch set](<https://lwn.net/Articles/666882/>) has been posted. The API that is proposed for this feature has shifted considerably since May, so it merits another look. 

Intel's memory protection keys feature works by making use of four page-table bits to assign one of sixteen key values to each page. A separate register then allows the assertion of "write-disable" and "access-disable" bits for each key value. Setting the write-disable bit for key seven, for example, will cause all pages marked with that key as being read-only, even if the protection bits on those pages would otherwise allow write access. The write- and access-disable bits are local to each thread, and they can be modified without privilege. Since keys are assigned to pages in the page-table entries, only the kernel can change those. 

The original patch set allowed processes to assign keys to pages with any system call that changed page permissions — `mprotect()` and `mmap()`, for example. Four new "permissions" bits were defined, corresponding to the four bits of the key value. This API ran into some difficulties in the review process, though; it was criticized as being too closely tied to one specific implementation of memory protection keys. It might not extend well even to future changes in Intel's mechanism, and might not fit equivalent mechanisms on other processors at all. So a rethink of the API was called for. 

In the middle of the discussion of this feature, Ingo Molnar [came up with](<https://lwn.net/Articles/667174/>) an interesting use case. The access-disable bit applies to data access, not execution access; as a result, it can be used, in conjunction with the regular "execute access" bit, to create regions of memory that can be executed by the processor, but which cannot be read by the executing process. That, he said, could be used to frustrate attacks against address-space layout randomization that read the executable text in order to try to locate a specific data structure or chunk of code. There could be security advantages to protecting library code, at least, in this manner. 

This idea seemed popular among the security-oriented developers in the discussion. Like anything else, this protection would not be absolute, since the access-disable bit can be turned off. But it adds another barrier that must be overcome by an attacker; in many cases, it may be enough to thwart an attack. 

Fully implementing this feature could be challenging for a number of reasons, not the least of which being that it's common to mix executable and read-only data in an executable image. Most of the work to implement this feature would have to be done in user space, and is thus beyond the immediate reach of the kernel community, but, as Ingo [said](<https://lwn.net/Articles/667177/>): ""That does not mean we can not try!"" As part of getting there, he suggested that, rather than just giving user space total control over the protection keys, the kernel should manage them and allocate them on request. That would, among other things, allow the kernel to reserve some keys for its own use in the future. 

That suggestion was implemented in the fourth revision of the patch set in the form of two new system calls: 

```
int pkey_alloc(unsigned long flags, unsigned long init_access_rights)
        int pkey_free(int pkey);
```

The `flags` value to `pkey_alloc()` is currently unused and must be zero. The initial access restrictions associated with the allocated key are provided in `init_access_rights`; either of the `PKEY_DENY_WRITE` and `PKEY_DENY_ACCESS` bits can be set. If a key is available for allocation, the kernel will allocate it and return the associated key number as the return value from `pkey_alloc()`. 

If an application is done with a particular key, that key can be returned to the system with `pkey_free()`. The code does not check whether any pages have that key value assigned to them; applications will want to be careful there or surprising things might happen. 

Assigning a key to a specific set of pages is done with the new `mprotect_key()` system call: 

```
int mprotect_key(void *start, size_t len, unsigned long prot, int pkey);
```

This system call will set both the page protections and the protection key for the pages starting at `start` and extending for `len` bytes. The given `pkey` must have been allocated to the process using `pkey_alloc()` or the call will fail. For what it's worth, this system call is called `mprotect_pkey()` and `pkey_mprotect()` in other parts of the patch set, so the final name may not yet be set in stone. 

Comments on the patch set this time around have been relatively subdued; it would seem that most developers have had their say and are happy with the direction that this work is taking. So it may find its way into the kernel in a near-future development cycle. What may take a bit longer, though, is actual availability of hardware that supports this feature, which is slated to first show up in [Skylake](<https://en.wikipedia.org/wiki/Skylake_%28microarchitecture%29>) server chips.
