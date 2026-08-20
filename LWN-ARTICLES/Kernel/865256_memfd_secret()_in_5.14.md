---
title: memfd_secret() in 5.14
url: https://lwn.net/Articles/865256/
date: "August 6, 2021"
category: "Memory management-Address-space isolation; Releases-5.14; System calls-memfd secret"
author: "By Jonathan Corbet August 6, 2021"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
August 6, 2021

The `memfd_secret()` system call has, in one form or another, been covered here since February 2020. [In the beginning](<https://lwn.net/Articles/812325/>), it was a flag to [`memfd_create()`](<https://man7.org/linux/man-pages/man2/memfd_create.2.html>), but its functionality was later moved to a separate system call. There have been many changes during this feature's development, but its core purpose remains the same: allow a user-space process to create a range of memory that is inaccessible to anybody else — kernel included. That memory can be used to store cryptographic keys or any other data that must not be exposed to others. This new system call was finally merged for the upcoming 5.14 release; what follows is a look at the form this call will take in the mainline kernel. 

The prototype for `memfd_secret()` is: 

```
int memfd_secret(unsigned int flags);
```

The only allowed flag is `O_CLOEXEC`, which causes the area to be removed when the calling process calls [`execve()`](<https://man7.org/linux/man-pages/man2/execve.2.html>). The secret area _will_ be available to children after a fork, though. The return value from `memfd_secret()` will be a file descriptor attached to the newly created secret memory area. 

At this point, the process can't actually access that memory, which doesn't even have a size yet. A call must be made to [`ftruncate()`](<https://man7.org/linux/man-pages/man2/truncate.2.html>) to set the size of the area, then [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>) is used to map that "file" into the owning process's address space. The pages allocated to populate that mapping will be removed from the kernel's direct map, and specially marked to prevent them from being mapped back in by mistake. Thereafter, the memory is accessible to that process, but to nobody else, not even the kernel. The memory is thus about as well protected as it can get, which is good, but there are some consequences as well. Pointers to the secret-memory region cannot be used in system calls, for example; this memory is also inaccessible to DMA operations. 

[The first public posting of this work](<https://lwn.net/ml/linux-kernel/1572171452-7958-1-git-send-email-rppt@kernel.org/>) happened in October 2019. A number of changes have been made over the nearly two years that this system call has been under development — beyond the shift to a separate system call ([in July 2020](<https://lwn.net/ml/linux-kernel/20200720092435.17469-1-rppt@kernel.org/>)), which was done because this functionality was deemed to have little in common with ordinary memfds. For example, the ability to reserve a dedicated range of memory for `memfd_secret()` was added in the [version-2 posting](<https://lwn.net/ml/linux-kernel/20200727162935.31714-1-rppt@kernel.org/>) later in July 2020, only to be removed in [version 5](<https://lwn.net/ml/linux-kernel/20200916073539.3552-1-rppt@kernel.org/>) in September. 

Early versions of the patch set included a flag to require the removal of the pages in the secret area from the kernel's direct map. Doing so has the advantage of making the memory inaccessible to the kernel (and to anybody who might be able to compromise the kernel), but there were fears that breaking up the 1GB pages used for the direct mapping would degrade performance. Those fears have subsided over time, though; performance with 2MB pages is not much different than with 1GB pages. So that option disappeared in [version 4](<https://lwn.net/ml/linux-kernel/20200818141554.13945-1-rppt@kernel.org/>) in August 2020, and direct-map removal became the rule. 

[Version 8](<https://lwn.net/ml/linux-kernel/20201110151444.20662-1-rppt@kernel.org/>), posted in November, added a couple of changes. Rather than allocating arbitrary kernel memory, [CMA](<https://lwn.net/Articles/486301/>) was used as the memory pool for `memfd_secret()`. Another change, which created a bit of controversy over the life of the patch, disables hibernation when a secret memory area is active. The purpose is to prevent the secret data from being written to persistent storage, but some users may become disgruntled if they find that they can no longer hibernate their systems. That notwithstanding, this behavior was part of the version that went into 5.14. 

Since the beginning, `memfd_secret()` has supported a flag requesting uncached mappings — memory that bypasses the memory caches — for the secret area. This feature makes the area even more secure by eliminating copies in the caches (which could be exfiltrated via Spectre vulnerabilities), but setting memory to be uncached will drastically reduce performance. The caches exist for a reason, after all. Andy Lutomirski repeatedly [opposed](<https://lwn.net/ml/linux-kernel/CALCETrXr-9ABs7rzXcCrh1VXn-15AfpwjA6bQA7aU9Ta7DR+bw@mail.gmail.com/>) making uncached mappings available, objecting to the performance cost and more; Rapoport finally [agreed to remove it](<https://lwn.net/ml/linux-kernel/20201124092919.GI8537@kernel.org/>). [Version 11](<https://lwn.net/ml/linux-kernel/20201124092556.12009-1-rppt@kernel.org/>) removed that feature, leading to the current state where there are no flags that are specific to this system call. 

In [version 17](<https://lwn.net/ml/linux-kernel/20210208084920.2884-1-rppt@kernel.org/>) (February 2021), `memfd_secret()` was disabled by default and a command-line option (`secretmem_enable=`) was added to enable it at boot time. This decision was made out of fears that system performance could be degraded by breaking up the direct map and locking secret-area memory in RAM, so the feature is unavailable unless the system administrator turns it on. This version also ended the use of CMA for memory allocations. 

And that leads to what is essentially the current state of `memfd_secret()`. It took 23 versions to get there, which seems like a lot, but that is often the nature of memory-management changes. Once 5.14 comes out, we'll see how big the user community is for this feature, and what changes will be deemed necessary. For now, though, this work would appear to have finally reached a successful conclusion. For the curious, [this draft man page](<https://lwn.net/ml/linux-mm/20210729082900.1581359-1-rppt@kernel.org/>) has a few more details.
