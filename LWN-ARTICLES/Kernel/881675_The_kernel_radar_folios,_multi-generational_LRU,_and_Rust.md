---
title: "The kernel radar: folios, multi-generational LRU, and Rust"
url: https://lwn.net/Articles/881675/
date: "January 20, 2022"
category: "Development tools-Rust; Memory management-Folios; Memory management-Page replacement algorithms"
author: "By Jonathan Corbet January 20, 2022"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
January 20, 2022

The kernel community is a busy place, so it is not even remotely possible to write full-length articles about everything that is going on. Other topics may be of interest, but not require a longer treatment. The answer is a collection of short topics covering developments that are on the radar; the selection this time around includes folios, the multi-generational LRU, and Rust in the kernel. 

#### A folio update

Folios have been an active topic since they were first [covered here](<https://lwn.net/Articles/849538/>) less than one year ago. A folio, recall, is just a container for a `struct page` that is guaranteed not to be a tail page. It can thus be used to refer to memory, in units of a single page or larger, in a way that is more type-safe and requiring fewer run-time checks than when working directly with `page` structures. After some [extensive discussion](<https://lwn.net/Articles/868598/>), the first set of folio patches was merged for the 5.16 kernel. 

A large change of that nature to the memory-management subsystem naturally leads to fears of regressions, but the work in 5.16 appears to have been relatively problem-free. So 5.17 saw another round of folio-related changes, mostly focused on the page cache (which caches file data). In current kernels, the page cache holds, unsurprisingly, pages, but the 4KB page size used on most systems is often far too small to be efficiently managed. When dealing with files of anything but the smallest size, there is value in caching larger chunks at a time. The 5.17 conversion of the page cache to use folios is intended, among other things, to allow the use of "large folios" (a name chosen because the more descriptive "multi-page folios" was a little too long). Large folios might be huge pages, but they don't have to be limited to the huge-page sizes supported by the CPU; the plan is to support any folio size, as long as it is a power of two. 

The 5.17 work adds the machinery to support large folios in the page cache, the low-level filesystem-support code, and in the XFS filesystem, but does not actually start using them yet. As Matthew Wilcox said in [his pull request](<https://lwn.net/ml/linux-mm/YdyuuBCe4EPmr3k2@casper.infradead.org/>): ""there may still be places I've overlooked which still have page size assumptions"". So the coming development cycle will, presumably, focus on finding any such places so that the transition can happen in 5.18. Meanwhile, the more adventurous among us can [enable large folios in 5.17](<https://lwn.net/ml/linux-kernel/20220116121822.1727633-1-willy@infradead.org/>) and help find the remaining sharp edges. 

#### The multi-generational LRU

Another significant memory-management change that has been under development over the last year is the [multi-generational LRU](<https://lwn.net/Articles/856931/>), which reworks how the kernel decides which pages to evict when memory is tight. Current kernels use a two-queue system, one each for pages deemed "active" and "inactive". Pages move between the queues based on accesses; when memory is needed, pages are reclaimed off the end of the inactive queue. The multi-generational work generalizes this setup into a larger number of queues, a change that seemingly improves the kernel's ability to identify the pages that are unlikely to be needed in the near future. 

When Yu Zhao [posted](<https://lwn.net/ml/linux-kernel/20220104202227.2903605-1-yuzhao@google.com/>) the sixth version of this patch set in early January, he added [a request](<https://lwn.net/ml/linux-kernel/YdSuSHa/Vjl6bPkg@google.com/>) for review and a verdict as to whether it could be merged for 5.17. That sparked a long discussion on the state of this work. As part of that discussion, Michal Hocko (who also did a lot of detailed review of the patches) [repeated](<https://lwn.net/ml/linux-kernel/YdxTR4+FL08XyFuO@dhcp22.suse.cz/>) a theme that has been heard with previous postings: that it would be better to see this work as a series of incremental changes rather than a big addition of new reclaim mechanism: 

> Changes in the reclaim path are paved with failures and reverts and fine tuning on top of existing fine tuning. The difference from your patchset is that they tend to be much much smaller and go incremental and therefore easier to review. 

Jesse Barnes [responded](<https://lwn.net/ml/linux-kernel/CAJmaN=n=kn9-gC8if5wp8Gfj7uN+QVrX0ex=9JPXC7rPvGf1Qg@mail.gmail.com/>) that an incremental series might be worse in this case: 

> I understand the desire for an "incremental approach that gets us from A->B". In the abstract it sounds great. However, with a change like this one, I think it's highly likely that such a path would be littered with regressions both large and small, and would probably be more difficult to reason about than the relatively clean design of MGLRU. On top of that, I don't think we'll get the kind of user feedback we need for something like this *without* merging it. 

Linus Torvalds [responded](<https://lwn.net/ml/linux-kernel/CAHk-=whAOM+xQm9vMe33qf8X1LX2QQZ7RBgKe_KH2C3LB=6Jdw@mail.gmail.com/>) to Barnes, saying that this work ""is worth going with"". Hocko [didn't disagree](<https://lwn.net/ml/linux-kernel/Yd1et2VFOX4xxgly@dhcp22.suse.cz/>) with Barnes, but did note that there are a lot of things needing fixing before the code could be merged in any case. 

Zhao, meanwhile, has been actively trying to get supporters of this work to post to the list in favor of its inclusion. Those who responded include [Holger Hoffstätte](<https://lwn.net/ml/linux-kernel/ca3a7d27-150d-ed06-c8f8-0c74c4f00667@applied-asynchrony.com/>), [Shuang Zhai](<https://lwn.net/ml/linux-kernel/20220111160455.2773-1-szhai2@cs.rochester.edu/>) (""the performance improvement is fabulous""), [Suleiman Souhlal](<https://lwn.net/ml/linux-kernel/CABCjUKBiECMEgnMaCrfApVbWPgw3UuwNH+-Y9ME+2ogJSm+p-Q@mail.gmail.com/>) (""Android on ChromeOS has been using MGLRU for a while now, with great results""), [Sofia Trinh](<https://lwn.net/ml/linux-kernel/CAMwLHrZi0ZGFsuc74Yj3DErT1zyG2DEudS3ZgPEsexj0XfZJyA@mail.gmail.com/>), [Donald Carr](<https://lwn.net/ml/linux-kernel/e2db335efc02c61f4a4035dd2270e256@chaos-reins.com/>), and [Oleksandr Natalenko](<https://lwn.net/ml/linux-kernel/11900935.O9o76ZdvQC@natalenko.name/>). 

There is clearly some interest in getting this work merged; it is just as clearly not in the cards for 5.17, though. Normally one would expect that a change this fundamental could take a long time yet to get in; given the pressure and the approval from Torvalds, though, it could happen a bit more quickly this time. Merging for 5.18 still seems optimistic, but sometime in 2022 could be a real possibility. 

#### Rust for Linux

The project to make it possible to develop kernel modules in the Rust programming language continues to move forward; the [third version](<https://lwn.net/ml/linux-kernel/20220117053349.6804-1-ojeda@kernel.org/>) of the Rust-support patch set was posted on January 17\. A number of changes had been made to keep up with the Rust community and to get this work closer to ready for inclusion. 

This version of the patch set supports (and thus requires) the recent [1.58 release](<https://blog.rust-lang.org/2022/01/13/Rust-1.58.0.html>) of the compiler. The build system is now able to determine automatically whether a suitable Rust toolchain is available for building and, if something is missing, it will tell the developer what is needed. The cover letter notes that a couple of the unstable Rust features required for kernel work are becoming stable in near-future compiler releases. There is, however, still [a discouragingly long list](<https://github.com/Rust-for-Linux/linux/issues/2>) of required unstable features. 

The series itself starts by increasing the maximum length of symbols that can be managed in the "kallsyms" mechanism. It seems that the name-mangling used by Rust can expand names considerably, to the point that 255 characters is not enough to store some names. Developers will not normally need to see the mangled names, but they will show up in kallsyms and may be surprising. Another preliminary step is to add C helper functions for a long list of things that already look like functions in the kernel — `readb()` or `kmap()`, for example — that are actually macros or are inlined. Those cannot be called directly from Rust, so they need to be turned into proper functions first. 

Most of the Rust code itself currently appears in two crates. The first, called [`alloc`](<https://lwn.net/ml/linux-kernel/20220117053349.6804-7-ojeda@kernel.org/>), deals with memory allocation. The Rust language wasn't built with the idea that code might need to continue when a memory allocation fails; instead, the normal result is an immediate crash. Since crashing in kernel code is considered to be impolite, a modified allocator that can handle failures is required. As a Rust developer would expect, it returns a `Result` object that contains either a pointer to the allocated memory or an error indication, depending on what happened. Evidently the work to support fallible allocations is meant to go into the upstream Rust library, so the kernel's version of this crate may eventually be able to go away. 

The other big crate is called [`kernel`](<https://lwn.net/ml/linux-kernel/20220117053349.6804-10-ojeda@kernel.org/>); it contains the rest of the impedance-matching code that makes kernel APIs look like proper Rust interfaces. These provide interfaces for char devices, the clock framework, `file` structures, `file_operations` vectors, memory-mapped I/O functions, mutexes, spinlocks, and more. A surprising amount of code is dedicated to the implementation of generic linked lists. 

All told, it represents a lot of work toward making it possible to write kernel code in Rust. It is quite a bit of code that, at some point, is going to need to be more widely exercised if it is to progress in useful directions. That, of course, would be helped by getting this support into the mainline kernel where more developers can look at and work with it. Torvalds indicated at the [2021 Maintainers Summit](<https://lwn.net/Articles/870555/>) that he expected to merge this work, but there is no indication of when that might happen. The timing is likely to come down to Torvalds and when he thinks that the time has come to open the door to this new language.
