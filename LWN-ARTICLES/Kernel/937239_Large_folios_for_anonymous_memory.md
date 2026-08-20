---
title: Large folios for anonymous memory
url: https://lwn.net/Articles/937239/
date: "July 6, 2023"
category: "Memory management-Folios"
author: "By Jonathan Corbet July 6, 2023"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
July 6, 2023

The transition to [folios](<https://lwn.net/Articles/849538/>) has transformed the memory-management subsystem in a number of ways, but has also resulted in a lot of code churn that has not been welcomed by all developers. As this work proceeds, though, some of the benefits from it are beginning to become clear. One example may well be in the handling of anonymous memory, as can be seen in a pair of patch sets from Ryan Roberts. 

The initial Linux kernel release used 4KB pages on systems whose total memory size was measured in megabytes — and a rather small number of megabytes at that. Since then, installed-memory sizes have grown by a few orders of magnitude or so, but the 4KB page size remains mostly unchanged. So the kernel has to manage far more pages than it once did; that leads to more memory used for tracking, longer lists to scan, and more page faults to handle. In many ways, a 4KB page size is far too small for contemporary systems. 

Some architectures support running with larger page sizes, and any system could emulate larger pages by clustering the existing base pages. But a larger page size has its own problem: internal fragmentation that can waste a significant amount of memory. In practice, this problem has been severe enough to keep 4KB pages around, despite their drawbacks. 

One of the key advantages that folios bring is that they make the system's base page size less important; a folio can have any size (as long as it is a power of two) and kernel code will do the right thing with it. That allows different sizes to be used in different settings, as appropriate. 

#### Anonymous folios

Roberts's [large folios for anonymous memory](<https://lwn.net/ml/linux-kernel/20230703135330.1865927-1-ryan.roberts@arm.com/>) patch set takes advantage of this flexibility to improve the management of anonymous pages — pages associated with program data and not backed by a file on disk. At its core, the change is simple; whenever the kernel is called upon to map a page of anonymous memory for a process, it tries to allocate and map a larger folio instead. By default, the code will try to allocate and map a 64KB folio, dropping down to smaller sizes if that cannot be done; there is a hook that allows architecture-specific code to specify a different default. 

Since anonymous memory starts out filled with zeroes, mapping it in larger chunks is not particularly hard; there is no extra I/O that must be done. The biggest advantage for the kernel is that mapping large folios can significantly reduce the number of page faults that must be handled. If a single fault results in the mapping of a 64KB folio, that memory range can be accessed with just that one fault, rather than the 16 that would otherwise be required when mapping 4KB base pages. 

Of course, it is not always possible to map a larger folio in that way. If a physically contiguous chunk of memory with a suitable size and alignment is not available, then the attempt will fail. It is also not possible to map a folio that extends beyond the virtual memory area (VMA) that contains it. If there are pages already mapped in a part of the address range that a folio would cover then, once again, that folio cannot be used. The ability to transparently drop down to smaller sizes means that the kernel can use an allocation that is suited to the conditions it finds at the time. Among other things, that helps to avoid internal fragmentation with small mappings. 

Benchmarks running the most important workload of all — compiling the kernel — show an approximately 5% reduction in the time needed, with a reduction in kernel time of about 40%. That, alone, suggests that this work may be a good idea, but there are more gains to be had on current hardware. 

#### Reducing TLB pressure

Virtual-address translation is a complicated process; it involves stepping through three to five levels of page tables, perhaps incurring cache misses at each step. The CPU tries to avoid this expense whenever possible by maintaining a cache of recent translations in the translation lookaside buffer (TLB). To a surprising extent, an application's performance will be determined by how well it fits into the TLB; a lot of TLB misses will slow things considerably. Unfortunately, TLB memory is expensive, so the cache is never as big as one might like it to be. 

One important technique for stretching the TLB is the use of huge pages, which can allow 2MB (or even 1GB) of memory to be covered by a single TLB entry. Huge pages are, however, huge; they can be difficult to allocate on a busy system and can create huge internal-fragmentation problems of their own. The smaller folios used by Roberts's patch are much easier to manage, but they don't provide the same TLB benefits that huge pages do. 

Or, at least, that was once the case. More recent CPUs have started adding a bit to their page-table entries to indicate that a small range of pages has been placed in physically contiguous memory. The processor can use that information to collapse the TLB entries referring to those pages into a single entry; the benefit is not as large as with a full huge page, but it is also much easier to obtain. This benefit will only be enjoyed, though, if the kernel sets the "contiguous PTE" bit where the mapping is truly contiguous. 

The [second patch set from Roberts](<https://lwn.net/ml/linux-kernel/20230622144210.2623299-1-ryan.roberts@arm.com/>) does exactly that, for the arm64 architecture at least. In an amazing coincidence, arm64 systems can map a contiguous range of up to 64KB — which just happens to be the default folio size set for arm64 in the first patch series — into a single TLB entry. With this series applied, contiguous ranges of pages are detected automatically, and the appropriate page-table bits will be set. That results in another 2% gain for the kernel-compilation benchmark. 

#### Discussion

These gains will only happen if this code is merged into the mainline kernel, though. That seems likely to happen, but there will be some changes needed first. For example, Yu Zhao has [complained](<https://lwn.net/ml/linux-kernel/CAOUHufa_xFJvFFvmw1Tkdc9cXaZ1GPA1dVSauH+J9zGX-sO1UA@mail.gmail.com/>) about the architecture-specific function to set the default folio size. That function takes the faulting VMA as a parameter; Zhao feels that the result is a mixture of architecture-specific decision making with policy that should be managed by the core memory-management code. Roberts has [indicated](<https://lwn.net/ml/linux-kernel/eea2b36d-9c6d-64ca-4e21-57cfd5a93d57@arm.com/>) that he is willing to change that interface. 

Zhao also [dislikes](<https://lwn.net/ml/linux-kernel/CAOUHufaK82K8Sa35T7z3=gkm4GB0cWD3aqeZF6mYx82v7cOTeA@mail.gmail.com/>) the practice of trying intermediate sizes if the desired folio size cannot be used. The work, he said, would ""be a lot easier to sell"" if it fell back immediately to the base-page size. As was explained in the anonymous-folio cover letter, Zhao has recommended this change in the past, and Roberts tried it; the result was worse performance on some benchmarks. So he seems less willing to give on this point. When asked, Zhao [gave three reasons](<https://lwn.net/ml/linux-kernel/CAOUHufZWONm+5QTo9F-2iyKdAHC+Nz2NPkWuJ1QsE6d4QjXgrA@mail.gmail.com/>) for his dislike of the intermediate fallback, with the most significant being that it may cause system-wide fragmentation: 

> The third one is why I really don't like this best-fit policy. By falling back to smaller orders, we can waste a limited number of physically contiguous pages on wrong vmas (small vmas only), leading to failures to serve large vmas which otherwise would have a higher overall ROI. 

A [possible compromise](<https://lwn.net/ml/linux-kernel/CAOUHufZY-zN8jjQz+iMzwmqMb2VCh7=N+YxfXobgF7i1zUmTNA@mail.gmail.com/>) would be to attempt a single fallback to the size known as [`PAGE_ALLOC_COSTLY_ORDER`](<https://elixir.bootlin.com/linux/v6.4.1/source/include/linux/mmzone.h#L37>), which is 32KB by default, before giving up and mapping base pages. In other words, this policy would avoid allocating relatively small (but still larger than single-page) folios that might break up larger, physically contiguous ranges of memory. 

Another concern is that this work — and the benchmarking that comes with it — is all specific to the arm64 architecture. Support for physically contiguous page-table entries is coming to x86 processors as well, so this feature will eventually need to work beyond arm64. That suggests that a favorable review from the x86 community will be a necessary precondition to getting this work merged. Intel developer Yin Fengwei has been participating in the discussion and has indicated that some, but not all, of the patches seem ready. 

The biggest stumbling block, though, may be that large anonymous folios are not yet fully integrated into the rest of the memory-management subsystem. As mentioned in [one changelog](<https://lwn.net/ml/linux-kernel/20230703135330.1865927-5-ryan.roberts@arm.com/>) in the series: 

> The new behaviour is hidden behind the new FLEXIBLE_THP Kconfig, which defaults to disabled for now; there is a long list of todos to make FLEXIBLE_THP robust with existing features (e.g. compaction, mlock, some madvise ops, etc). These items will be tackled in subsequent patches. 

Roberts has posted [a more detailed list](<https://lwn.net/ml/linux-kernel/4d4c45a2-0037-71de-b182-f516fee07e67@arm.com/>) of things that need to be fixed and indicated that he would prefer to merge the feature, disabled by default, and deal with the remaining problems afterward. But, as Matthew Wilcox [pointed out](<https://lwn.net/ml/linux-kernel/ZKVdUDuwNWDUCWc5@casper.infradead.org/>), there will be little desire to merge a patch set that still has that kind of outstanding issue, so these problems will almost certainly need to be worked out before this feature can be considered ready for the mainline. 

This work suggests that the debate over whether the kernel's page size should be increased is over; the answer is to use the size that works best in each situation rather than using a single page size everywhere. The folio work has given the kernel some of the flexibility needed to adopt a policy like that. There is a gap, though, between the ability to implement such a feature and creating a feature that can be deployed in production kernels. Future kernels will almost certainly be capable of mapping variably sized anonymous folios, but getting to that point may take a while yet.
