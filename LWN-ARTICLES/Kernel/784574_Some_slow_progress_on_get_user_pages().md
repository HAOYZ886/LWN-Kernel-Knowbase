---
title: Some slow progress on get_user_pages()
url: https://lwn.net/Articles/784574/
date: "April 2, 2019"
category: "Memory management-get user pages"
author: "By Jonathan Corbet April 2, 2019"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
April 2, 2019

One of the surest signs that the Linux Storage, Filesystem, and Memory-Management (LSFMM) Summit is approaching is the seasonal migration of memory-management developers toward the `get_user_pages()` problem. This core kernel primitive is necessary for high-performance I/O to user-space memory, but its interactions with filesystems have never been reliable — or even fully specified. There are currently a couple of patch sets in circulation that are attempting to improve the situation, though a full solution still seems distant. 

`get_user_pages()` is a way to map user-space memory into the kernel's address space; it will ensure that all of the requested pages have been faulted into RAM (and locked there) and provide a kernel mapping that, in turn, can be used for direct access by the kernel or (more often) to set up zero-copy I/O operations. There are a number of variants of `get_user_pages()`, most notably `get_user_pages_fast()`, which trades off some flexibility for the ability to avoid acquiring the contended `mmap_sem` lock before doing its work. The ability to avoid copying data between kernel and user space makes `get_user_pages()` the key to high-performance I/O. 

If `get_user_pages()` is used on anonymous memory, few problems result. Things are different when, as is often the case, file-backed memory is mapped in this way. Filesystems are generally responsible for the state of file-backed pages in memory; they ensure that changes to those pages are written back to permanent storage, and they make sure that the right thing happens when a file's layout on that storage changes. Filesystems are not designed to expect that file-backed pages can be written to without their knowledge, but that is exactly what can happen when those pages are mapped with `get_user_pages()`. 

Most of the time, things happen to work anyway. But if an I/O operation writes to a page while the filesystem is, itself, trying to write back changes to that page, data corruption can result. In some cases, having a page unexpectedly become dirty can cause filesystem code to crash. And there is a whole new range of problems that can turn up for filesystems on nonvolatile memory devices, where writing to a page directly modifies the underlying storage. Filesystems implementing this sort of direct access (a mode called "DAX") can avoid some problems by being careful to not move file pages around while references to them exist, but that leads to different kinds of problems if pages mapped with `get_user_pages()` remain mapped for long periods of time. Naturally, certain subsystems (notably the RDMA layer) [do exactly that](<https://lwn.net/Articles/774411/>). 

#### Tracking get_user_pages() mappings

When memory is mapped into kernel space with `get_user_pages()` the reference count for each page is incremented; among other things, that prevents those pages from being evicted from RAM for as long as the mapping persists. When the kernel is done with those pages, the references are released by calling `put_page()` on each page; `put_page()` is a generic function that is used to release any reference to a page. There is currently no infrastructure for tracking references resulting specifically from `get_user_pages()` calls, so there is no way for any other kernel subsystem to know when such references exist. 

John Hubbard is trying to change that situation with [a simple patch](<https://lwn.net/ml/linux-kernel/20190308213633.28978-1-jhubbard@nvidia.com/>) adding `put_user_page()`, which is intended to be called instead of `put_page()` when releasing references created by `get_user_pages()`. In this patch set, the new function is defined as: 

```
static inline void put_user_page(struct page *page)
        {
    	put_page(page);
        }
```

In other words, `put_user_page()` simply turns into a call to `put_page()`, with no other changes. It clearly is not solving any problems in its own right, but it is a first step in a larger strategy. 

The next step is to locate all `get_user_pages()` callers in the kernel and convert them to use `put_user_page()`; there are quite a few of those, so this process is expected to take a while. Once that has been done, though, those functions can be changed to allow for separate tracking of references created by `get_user_pages()`. [According to Jérôme Glisse](<https://lwn.net/ml/linux-kernel/20190319220654.GC3096@redhat.com/>), the plan is to increment the page reference counts by a rather higher number (called `GUP_BIAS`) rather than by one. Any page with a reference count greater than `GUP_BIAS` can then be assumed to have references created by `get_user_pages()`, meaning that it might be written to, without warning, by some peripheral device on the system. 

The next step appears to be a little fuzzier; Glisse describes it as ""decide what to do for GUPed page"". The thoughts seem to include keeping such pages in a dirty state at all times; writeback by filesystems would also be performed using bounce pages in an attempt to avoid corruption problems. Keeping pages dirty would disable a lot of filesystem features (such as copy-on-write) but, Glisse said, ""it seems to be the best thing we can do"". Another idea, [suggested](<https://lwn.net/ml/linux-kernel/c854b2d6-5ec1-a8b5-e366-fbefdd9fdd10@nvidia.com/>) by Hubbard, is to introduce a "file lease" mechanism that would allow for coordination between user space and the kernel when filesystem code needs to shuffle pages around. 

This patch has found its way into the -mm tree, and thus into linux-next, so it seems likely to be merged for 5.2. 

#### `FOLL_LONGTERM`

When `get_user_pages()` was first added to the kernel, it was assumed that pages would be kept mapped for short periods of time. Over the years, that assumption has become increasingly invalid; now mappings from subsystems like RDMA can literally last for days, and the new [`io_uring()`](<https://lwn.net/Articles/776703/>) system call can also create mappings with an indefinite lifetime. Such long-lived mappings can be a stress for any filesystem implementation, but they are especially problematic for those that implement DAX. A file page that is referenced will block a number of important operations, from copy-on-write to basic housekeeping when a file is deleted. 

In fact, long-lived references have been deemed to be fundamentally incompatible with DAX filesystems. As a result, a variant called `get_user_pages_longterm()` was merged for the 4.15 kernel release; it functions like `get_user_pages()` with the exception that it will refuse to create a mapping on filesystems where DAX is enabled. Creators of long-lived mappings can use this function to avoid causing problems for filesystems on nonvolatile-memory devices. This helps to head off one problem with long-lived mappings, but creates another: even though these mappings can last a long time, users like RDMA would still rather use `get_user_pages_fast()` to create them efficiently — and there is no `get_user_pages_fast_longterm()`. Creators of long-lived mappings are thus stuck using the slower interface. 

Ira Weiny is trying to address this limitation with [a patch set](<https://lwn.net/ml/linux-kernel/20190328084422.29911-1-ira.weiny@intel.com/>) adding a new `FOLL_LONGTERM` flag to `get_user_pages()`. This flag requests the same functionality as a call to `get_user_pages_longterm()` does now; indeed, that function is reimplemented to use the new flag as part of the patch set. But making it available in the core of `get_user_pages()` means that this flag can now be passed to `get_user_pages_fast()`; that, in turn, means that creators of long-lived mappings can do so more quickly. 

This patch set, too, is currently present in linux-next, and is thus likely to be in the 5.2 release. 

While both patch sets improve the situation, they are both just nibbling at a big problem that has been vexing memory-management and filesystem developers for years. There will doubtless be a lot of discussion on the topic at [the upcoming LSFMM Summit](<https://events.linuxfoundation.org/events/linux-storage-filesystem-mm-summit-2019/>) and afterward as well. `get_user_pages()` may make things fast, but the process of making it play well with filesystems in all settings is not.
