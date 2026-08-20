---
title: An ioctl() call to detect memory writes
url: https://lwn.net/Articles/940704/
date: "August 10, 2023"
category: "Memory management; Releases-6.7; userfaultfd"
author: "By Jonathan Corbet August 10, 2023"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
August 10, 2023

It is the kernel's business to know when a process's memory has been written to; among other things, this knowledge is needed to determine which pages can be immediately reclaimed or to properly write dirty pages to backing store. Sometimes, though, user space also needs access to this information in a reliable and fast manner. [This patch series](<https://lwn.net/ml/linux-mm/20230808104309.357852-1-usama.anjum@collabora.com/>) from Muhammad Usama Anjum adds a new `ioctl()` call for this purpose; using it requires repurposing an existing system call in an unusual way, though. 

The driving purpose for this feature, it seems, is to enable an efficient emulation of the Windows [`GetWriteWatch()`](<https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-getwritewatch>) system call, which is evidently useful for game developers who want to defend against certain kinds of cheating. A game player who is able to access (and modify) a game's memory can enhance that game's functionality in ways that are not welcomed by the developers — or by other players. Using `GetWriteWatch()`, the game is able to detect when crucial data structures have been modified by an external actor, put up the modern equivalent of a "Tilt" indicator, and bring the gaming session to a halt. 

Linux actually provides this functionality now by way of the [`pagemap` file](<https://docs.kernel.org/admin-guide/mm/soft-dirty.html>) in `/proc`. The current dirty state of a range of pages can be read from this file, and writing the associated `clear_refs` file will reset the dirty state (useful, for example, after the game itself has written to the memory of interest). Accessing this file from user space is slow, though, which runs counter to the needs of most games. The new `ioctl()` call is meant to implement this feature more efficiently. The [Checkpoint/Restore In Userspace](<https://criu.org/Main_Page>) (CRIU) project would also be able to make use of a more efficient mechanism to detect writes; in this case, the purpose is to identify pages that have been modified after the checkpoint process has begun. 

#### Soft dirty deemed insufficient

The kernel's "soft dirty" mechanism, which provides the `pagemap` file, would seem to be the appropriate base on which to build a feature like this. All that should be needed is a more efficient mechanism to query the data and to reset the soft-dirty information for a specific range of pages. According to the cover letter, though, this approach ended up not working well. There are various other operations, such as virtual memory area merging or [`mprotect()`](<https://man7.org/linux/man-pages/man2/mprotect.2.html>) calls, that can cause pages to be reported as dirty even though they have not been written to. That, in turn, can lead to a game concluding, incorrectly, that its memory has been tampered with. 

That can lead to undesirable results. One rarely sees the level of anger that can be reached by a game player who has been told, at the crux point of a quest, that cheating has been detected and the game is over. 

Fixing this false-positive problem, evidently, is not an option, so the decision was made to work around it instead via an unexpected path. The [`userfaultfd()`](<https://man7.org/linux/man-pages/man2/userfaultfd.2.html>) system call allows a process to take charge of its own page-fault handling for a given range of memory. The patch set adds a new operation (`UFFD_FEATURE_WP_ASYNC`) to `userfaultfd()` that changes how write-protect faults are handled. Rather than pass such faults to user space, the kernel will simply restore write permission to the page in question and allow the faulting process to continue. 

Thus, `userfaultfd()`, which was designed to allow the handling of faults in user space, is now being used to modify the handling of faults directly within the kernel with no user-space involvement. This approach does have some advantages for the use case in question, though: it allows specific ranges of memory to be targeted, and the use of write protection to trap write operations provides for more reliable reporting of writes to memory. To see which pages have been written, it is sufficient to query the write-protect status; if the page has been made writable, it has been written to. 

#### The `ioctl()` interface

With that piece in place, it is possible to create an interface to query the results. That is done by opening the `pagemap` file for the process in question, then issuing the new `PAGEMAP_SCAN` `ioctl()` call. This call takes a relatively complex structure as an argument: 

```
struct pm_scan_arg {
    	__u64 size;
    	__u64 flags;
    	__u64 start;
    	__u64 end;
    	__u64 walk_end;
    	__u64 vec;
    	__u64 vec_len;
    	__u64 max_pages;
    	__u64 category_inverted;
    	__u64 category_mask;
    	__u64 category_anyof_mask;
    	__u64 return_mask;
        };
```

The `size` argument contains the size of the structure itself; it is there to enable the backward-compatible addition of more fields later. There are two `flags` values that will be described below. The address range to be queried is specified by `start` and `end`; the `walk_end` field will be updated by the kernel to indicate where the page scan actually ended. `vec` points to an array holding `vec_len` structures (described below) to be filled in with the desired information. 

The final four fields describe the information the caller is looking for. There are six "categories" of pages that can be reported on: 

  * `PAGE_IS_WPALLOWED`: the page protections allow writing. 
  * `PAGE_IS_WRITTEN`: the page has been written. 
  * `PAGE_IS_FILE`: the page is backed by a file. 
  * `PAGE_IS_PRESENT`: the page is present in RAM. 
  * `PAGE_IS_SWAPPED`: the (anonymous) page has been written to swap. 
  * `PAGE_IS_PFNZERO`: the page-table entry points to the zero page. 

Every page will belong to some combination categories; a calling program will want to learn about the pages in the range of interest that are (or are not) in a subset of those categories. The usage of the masks is described, to a point, in [this patch](<https://lwn.net/ml/linux-mm/20230808104309.357852-6-usama.anjum@collabora.com/>), though it makes for difficult reading. The code itself does the following to determine if a given page, described by `categories`, is interesting: 

```
categories ^= p->arg.category_inverted;
        if ((categories & p->arg.category_mask) != p->arg.category_mask)
    	return false;
        if (p->arg.category_anyof_mask && !(categories & p->arg.category_anyof_mask))
    	return false;
        return true;
```

Or, in English: first, the `category_inverted` mask is used to flip the sense of any of the selected categories; this is a way of selecting for pages that do _not_ have a given category set. Then, the result must have all of the categories described by `category_mask` set and, if `category_anyof_mask` is not zero, at least one of those categories must be set as well. If all of those tests succeed, the page is of interest; otherwise it will be skipped. 

After the scan, those pages are reported back to user space in `vec`, which is an array of these structures: 

```
struct page_region {
    	__u64 start;
    	__u64 end;
    	__u64 categories;
        };
```

The `return_mask` field of the request is used to collapse the pages of interest down to regions with the same categories set; for each such region, the return structure describes the address range covered and the actual categories set. 

Finally, to return to the `flags` argument, there are two possibilities. If `PM_SCAN_WP_MATCHING` is set, the call will write-protect all of the selected pages after noting their status; this is meant to allow checking for pages that have been written and resetting the status for the next check. If `PM_SCAN_CHECK_WPASYNC` is set, the whole operation will be aborted if the memory region has not been set up with `userfaultfd()` as described above. 

#### Checking for cheaters

To achieve the initial goal of determining whether pages in a given range have been modified, the first step is to invoke `userfaultfd()` to set up the write-protect handling. Then, occasionally, the application can invoke this new `ioctl()` call with both flags described above set, and with both `category_mask` and `return_mask` set to `PAGE_IS_WRITTEN`. If no pages have been written, no results will be returned; otherwise the returned structures will point to where the modifications have taken place. Meanwhile, the pages that were written to will have their write-protect status reset, ready for the next scan. 

This series is in an impressive ~~27th~~ [28th](<https://lwn.net/ml/linux-kernel/20230809061603.1969154-1-usama.anjum@collabora.com/>) revision as of this writing. It was initially [proposed](<https://lwn.net/ml/linux-kernel/20220726161854.276359-1-usama.anjum@collabora.com/>) in mid-2022 as a new system call named `process_memwatch()` before eventually becoming an `ioctl()` call instead. There is clearly some motivation to get this feature merged, but the level of interest in the memory-management community is not entirely clear. The work has not landed in linux-next, and recent posts have not generated a lot of comments. There do appear to be use cases for the feature, though, so a decision will need to be made at some point.
