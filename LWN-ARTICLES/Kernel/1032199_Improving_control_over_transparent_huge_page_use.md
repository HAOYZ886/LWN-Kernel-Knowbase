---
title: Improving control over transparent huge page use
url: https://lwn.net/Articles/1032199/
date: "August 5, 2025"
category: "Huge pages; Memory management-Huge pages"
author: "By Jonathan Corbet August 5, 2025"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
August 5, 2025

The use of huge pages can significantly increase the performance of many workloads by reducing both memory-management overhead in the kernel and pressure on the system's translation lookaside buffer (TLB). The addition of transparent huge pages (THP) for the 2.6.38 kernel release in 2011 caused the kernel to allocate huge pages automatically to make their benefits available to all workloads without any effort needed on the user-space side. But it turns out that use of huge pages can make some workloads slower as the result of internal memory fragmentation, so the THP feature is often disabled. Two patch sets aimed at better targeting the use of transparent huge pages are currently working their way through the review process. 

Over the years, the kernel has evolved a number of ways to control the use of THP; they are described in [`Documentation/admin-guide/mm/transhuge.rst`](<https://docs.kernel.org/admin-guide/mm/transhuge.html>). At the global level, the `/sys/kernel/mm/transparent_hugepage/enabled` knob controls behavior system-wide. It can be set to "`always`" or "`never`" with obvious results. This knob also supports the "`madvise`" setting, which only enables THP for processes that explicitly opt in for specific memory regions with a call to [`madvise()`](<https://man7.org/linux/man-pages/man2/madvise.2.html>). The kernel, in other words, allows for the imposition of a system-wide policy, with the possibility of restricting THP usage to places where applications have explicitly enabled it. 

#### Tweaking `prctl()`

There are more control points for THP usage, though, including a whole set of knobs for the `khugepaged` kernel thread (which builds huge pages out of base pages in the background) and a set of kernel command-line options. There is also the [`PR_SET_THP_DISABLE`](<https://man7.org/linux/man-pages/man2/PR_SET_THP_DISABLE.2const.html>) option to [`prctl()`](<https://man7.org/linux/man-pages/man2/prctl.2.html>), which lets a process disable the THP usage for its entire address space regardless of the overall system policy. This option is used in programs where the developer feels that THP allocations will only harm performance; for example, [MariaDB](<https://sources.debian.org/src/mariadb/1:11.8.2-1/sql/mysqld.cc/?hl=5683#L5683>) uses this feature. 

The implementation of `PR_SET_THP_DISABLE` is absolute; it will override any `madvise()` calls that the process may subsequently make to enable THP for an address range. Disabling THP process-wide in this way is a big hammer. There are use cases where it would be useful to be able to disable THP generally (on systems where it is enabled globally), while still being able to enable THP for specific memory regions, but the kernel does not currently offer that option. There is, in other words, no way for a process to request "do not use THP unless otherwise requested". 

As described in [this patch from David Hildenbrand](<https://lwn.net/ml/all/20250804154317.1648084-2-usamaarif642@gmail.com>) (which has been made part of [this series](<https://lwn.net/ml/all/20250804154317.1648084-1-usamaarif642@gmail.com>) by Usama Arif), one could consider letting `madvise()` override `PR_SET_THP_DISABLE` but, as Hildenbrand put it, ""this would change the documented semantics quite a bit"". There may also be cases where the current semantics are what is actually wanted, so this does not seem like a good option. 

Instead, Hildenbrand's patch adds a new option to `PR_SET_THP_DISABLE`, called `PR_THP_DISABLE_EXCEPT_ADVISED`, that provides the new semantics. When this option is provided (as the third parameter to `prctl()`), THP will be disabled process-wide except for regions where `madvise()` has been used to request an exception, addressing the above-mentioned use case. This series has been through a couple of revisions and appears to be stabilizing. 

This new option may solve an immediate problem, but it is one more knob added to a complex and interacting pile of them. It also does not address another aspect of the THP problem. Once upon a time, there was only one size of THP: the "PMD size", which is 2MB on many systems. The advent of multi-size THP, which allows many different sizes of huge pages, has complicated the situation. One result of that can be seen by looking in the `/sys/kernel/mm/transparent_hugepage` directory, which now includes subdirectories for a range of huge-page sizes. The system administrator can use this large array of knobs to control the allocation of specific THP sizes system-wide, but there is currently no way to tune the policy with size granularity for a specific process's needs. The idea of adding more complexity to the `prctl()` and `madvise()` interfaces to provide that flexibility is looking increasingly unappetizing. 

#### Using BPF

At the conclusion of his changelog, Hildenbrand acknowledges that the new `prctl()` option does not solve the whole problem: 

> Likely, the future will use bpf or something similar to implement better policies, in particular to also make better decisions about THP sizes to use, but this will certainly take a while as that work just started. 

The current form of that work can be seen in [this patch set](<https://lwn.net/ml/all/20250729091807.84310-1-laoar.shao@gmail.com>) from Yafang Shao. It introduces a new [struct-ops](<https://docs.ebpf.io/linux/program-type/BPF_PROG_TYPE_STRUCT_OPS/>) hook with this BPF callback: 

```
int (*get_suggested_order)(struct mm_struct *mm, unsigned long tva_flags,
                                   int order);
```

If a BPF program is attached to this hook, it will be called from the memory-management subsystem when page-allocation decisions are being made. The `mm` argument will point to the [`mm_struct`](<https://elixir.bootlin.com/linux/v6.16/source/include/linux/mm_types.h#L933>) structure describing the address space, `tva_flags` describes the current context (and, specifically, whether a page fault is being handled), and `order` is the largest size that is being considered for this allocation. The function should return an order number indicating the largest allocation that should actually be performed; returning zero will disable huge pages entirely in this case. 

To make this decision, the BPF program will likely want to know a bit about the context in which the allocation is being made. The patch series adds a couple of new kfuncs, `bpf_mm_get_mem_cgroup()` and `bpf_mm_get_task()`, to give that program access to the relevant control-group and task information. Shao notes in the series that there may be a need for other helpers in the future to provide information about the memory pressure that the system is currently experiencing. 

The patch series, currently in its fourth revision, has not yet reached the point where it is ready to go upstream. One significant change was [requested](<https://lwn.net/ml/all/08D7155B-84F0-4575-B192-96901CFE690A@nvidia.com>) by Zi Yan, who would like to give the callback access to the relevant virtual memory area (VMA) as well. Shao [answered](<https://lwn.net/ml/all/CALOAHbDRBs8bdXB_LJjx-7gALOCLvmMxFD+c7MbHAiQ3htXawA@mail.gmail.com>) that ""mm is sufficient for our use cases"", but Hildenbrand [said](<https://lwn.net/ml/all/65e9ee9a-3b64-4efc-ade0-83990de7af91@redhat.com>) that an approach based on virtual memory areas would be required. 

A separate concern, raised in regard to [the previous revision](<https://lwn.net/ml/all/20250608073516.22415-1-laoar.shao@gmail.com>), was that any BPF callbacks could quickly find themselves cast in stone as part of the kernel's ABI. That is especially undesirable given that the problem of how to best control the use of THP is clearly not fully understood at this point. As Hildenbrand [put it](<https://lwn.net/ml/all/87a54cdb-1e13-4f6f-9603-14fb1210ae8a@redhat.com>): 

> Every MM person I talked to about this was like "as soon as it's actively used out there (e.g., a distro supports it), there is no way you can easily change these callbacks ever again - it will just silently become stable." 
> 
> That is actually the biggest concern from the MM side: being stuck with an interface that was promised to be "unstable" but suddenly it's not-so-unstable anymore, and we have to support something that is very likely to be changed in the future. 

As a result of these concerns, the current patch set includes warnings that the interface could change or even be removed in future kernel versions. That matches the general understanding around BPF programs; they are not normally deemed to be a part of the kernel's stable ABI. That said, if a given BPF interface becomes sufficiently deeply integrated into how systems are managed, it will become increasingly hard to change or remove. Memory-management developers could end up having to support it for years, even if it gets in the way of needed changes. 

So the addition of BPF control for core memory-management decisions is likely to be approached with a great deal of caution, even by developers who feel that it is the most logical way to solve the problems around control of THP usage. There will almost certainly be a way to control THP allocation with BPF in a future kernel, but that may not happen as quickly as some might like.
