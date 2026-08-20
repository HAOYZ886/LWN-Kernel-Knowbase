---
title: "Triggering huge-page collapse from user space"
url: https://lwn.net/Articles/887753/
date: "March 14, 2022"
category: "Huge pages; Memory management-Huge pages"
author: "By Jonathan Corbet March 14, 2022"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
March 14, 2022

When the kernel first gained support for huge pages, most of the work was left to user space. System administrators had to [set aside memory in the special hugetlbfs filesystem](<https://lwn.net/Articles/374424/>) for huge pages, and programs had to explicitly map memory from there. Over time, the [transparent huge pages](<https://lwn.net/Articles/423584/>) mechanism automated the task of using huge pages. That mechanism is not perfect, though, and some users feel that they have better knowledge of when huge-page use makes sense for a given process. Thus, huge pages are now coming full circle with [this patch set](<https://lwn.net/ml/linux-mm/20220308213417.1407042-1-zokeefe@google.com/>) from Zach O'Keefe returning huge pages to user-space control. 

Huge pages, of course, are the result of larger page sizes implemented by the CPU; the specific page sizes available depend on the processor model and its page-table layout. An x86 processor will normally, for example, support a "base" page size of 4KB, and huge pages of 2MB and 1GB. Huge pages dispense with the bottom layer (or layers) of the page-table hierarchy, speeding the address-translation process slightly. The biggest performance advantage that comes from huge pages, though, results from the reduced pressure on the processor's scarce translation lookaside buffer (TLB) slots. One 2MB huge page takes one TLB slot; when that memory is accessed as base pages, instead, 512 slots are needed. For some types of applications the speedup can be significant, so there is value in using huge pages when possible. 

That said, there are also costs associated with huge pages, starting with the fact that they are huge. Processes do not always need large, virtually contiguous memory ranges, so placing all process memory in huge pages would end up wasting a lot of memory. The transparent huge pages mechanism tries to find a balance by scanning process memory and finding the places where huge pages might make sense; when such a place is found, a range of base pages is "collapsed" into a single huge page without the owning process being aware that anything has changed. 

There are costs to transparent huge pages too, though. The scanning itself takes CPU time, so there are limits to how much memory the `khugepaged` kernel thread is allowed to scan each second. The limit keeps the cost of `khugepaged` within reason, but also reduces the rate at which huge pages are used, causing processes that could benefit from them to run in a more inefficient mode for longer. 

The idea behind O'Keefe's patch set is to allow user space to induce huge-page collapse to happen quickly in places where it is known (or hoped) that use of huge pages will be beneficial. The idea was first [suggested by David Rientjes](<https://lwn.net/ml/linux-mm/d098c392-273a-36a4-1a29-59731cdf5d3d@google.com/>) in early 2021, and eventually implemented by O'Keefe. Beyond allowing huge-page collapse to happen sooner, O'Keefe says, this work causes the CPU time necessary for huge-page collapse to be charged to the process that requests it, increasing fairness. 

It also allows the process to control when that work is done. Data stored in base pages will be scattered throughout physical memory; collapsing those pages into a huge page requires copying the data into a single, physically contiguous, huge page. This, in turn, requires blocking changes to those pages during the copy and uses CPU time, all of which can increase latency, so there is value in being able to control when that work happens. 

A process can request huge-page collapse for a range of memory with a new [`madvise()`](<https://man7.org/linux/man-pages/man2/madvise.2.html>) request: 

```
int madvise(void *begin, size_t length, MADV_COLLAPSE);
```

This call will attempt to collapse `length` bytes of memory beginning at `begin` into huge pages. There does not appear to be any specific alignment requirement for those parameters, even though huge pages _do_ have alignment requirements. If `begin` points to a base page in the middle of the address range that the huge page containing it will cover, then pages before `begin` will become part of the result. In other words, `begin` will be aligned backward to the proper beginning address for the containing huge page. The same is true for `length`, which will be increased if necessary to encompass a full huge page. 

There are, of course, no guarantees that this call will succeed in creating huge pages; that depends on a number of things, including the availability of free huge pages in the system. Even if the operation is successful, a vindictive kernel could split the huge page(s) apart again before the call returns. If at least some success was had, the return code will be zero; otherwise an error code will be returned. A lack of available huge pages, in particular, will yield an `EAGAIN` error code. 

Support for `MADV_COLLAPSE` is also added to [`process_madvise()`](<https://man7.org/linux/man-pages/man2/process_madvise.2.html>), allowing one process to induce huge-page collapse in another. In this case, there are a couple of flags that are available (these would be the first use of the `flags` argument to `process_madvise()`): 

  * `MADV_F_COLLAPSE_LIMITS` controls whether this operation should be bound by the limits on huge-page collapse that `khugepaged` follows; these are set via sysctl knobs in existing kernels. If the calling process lacks the `CAP_SYS_ADMIN` capability, then the presence of this flag is mandatory. It is arguably a bit strange to require an explicit flag to request the default behavior, but that's the way of it. 
  * `MADV_F_COLLAPSE_DEFRAG`, if present, allows the operation to force page compaction to create free huge pages, even if the system configuration would otherwise not allow that. This flag does not require any additional capabilities, perhaps because the cost of compaction would be borne by the affected process itself. 

The end result, O'Keefe says, is a mechanism that allows user space to take control of the use of huge pages, perhaps to the point that the kernel need no longer be involved: 

> Though not required to justify this series, hugepage management could be offloaded entirely to a sufficiently informed userspace agent, supplanting the need for khugepaged in the kernel. 

First, though, this work would need to make it into the mainline kernel. Most of the review comments thus far are focused on details, but David Hildenbrand did [take exception](<https://lwn.net/ml/linux-mm/30571216-5a6a-7a11-3b2c-77d914025f6d@redhat.com/>) to one aspect of this new operation's behavior. In the current patch series, huge pages will be created for any virtual memory area, even those that have been explicitly marked to _not_ use huge pages with an `madvise(MADV_NOHUGEPAGE)` call. That, he said, ""would break KVM horribly"" on the s390 architecture. This behavior will thus need to change. 

The current patch set only works with anonymous pages; the plan is to add support for file-backed pages at a later time. Since one of the stated justifications for this patch is to be able to quickly enable huge pages for executable text, support for file-backed pages seems important, and the developers are likely to want to see it before giving this work the go-ahead. The feature looks like it will be useful for some use cases, though, so it seems likely to find its way into the mainline sooner or later.
