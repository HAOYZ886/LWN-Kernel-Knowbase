---
title: "How to cope with hardware-poisoned page-cache pages"
url: https://lwn.net/Articles/893565/
date: "May 5, 2022"
category: Fault tolerance; HWPOISON
author: "By Jonathan Corbet May 5, 2022 LSFMM"
---

By **Jonathan Corbet**  
May 5, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

"Hardware poisoning" is a mechanism for detecting and handling memory errors in a running system. When a particular range of memory ceases to remember correctly, it is "poisoned" and further accesses to it will generate errors. The kernel has had [support for hardware poisoning](<https://lwn.net/Articles/348886/>) for over a decade, but that doesn't mean it can't be improved. At the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>), Yang Shi discussed the challenges of dealing with hardware poisoning when it affects memory used for the page cache. 

The page cache, of course, holds copies of pages from files in secondary storage. A page-cache page that is generating errors will no longer accurately reflect the data that is (or should be) in the file, and thus should not be used. If that page has not been modified since having been read from the backing store, the solution is easy: discard it and read the data again into memory that actually works. If the page is dirty (having been written to by the CPU), though, the situation is harder to deal with. Currently, Shi said, the page is dropped from the page cache and any data that was in it is lost. Processes will not be notified unless they have the affected page mapped into their address space. 

This behavior, Shi said, leads to silent data loss. Subsequent accesses to the page will yield incorrect data, with no indication to the user that there is a problem. That leads to problems that can be difficult to debug. 

To solve the problem, he continued, the kernel should keep the poisoned page in the page cache rather than dropping it. The filesystem that owns the page will need to be informed of the problem and must not try to write the page back to secondary store. Some operations, such as truncation or [![\[Yang Shi\]](https://static.lwn.net/images/conf/2022/lsfmm/YangShi-sm.png)](<https://lwn.net/Articles/893730/>) hole-punching, can be allowed to work normally since the end result will be correct. But if the page is accessed in other ways, an error must be returned. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

There are a few ways in which this behavior could be implemented. One would be to check the hardware-poison flag on every path that accesses a page-cache page; that would require a lot of code changes. An alternative would be to return `NULL` when looking up the page in the cache. The advantage here is that callers already have to be able to handle `NULL` return values, so there should be few surprises — except that the error returned to user space will be `ENOMEM`, which may be surprising or misleading. Finally, page-cache lookups could, instead, return `EIO`, which better indicates the nature of the real problem. That would be much more invasive, though, since callers will not be prepared for that return status. 

Matthew Wilcox jumped in to say that only the first alternative was actually viable. Poisoning is tracked per page, but the higher-level interfaces are being converted to folios, which can contain multiple pages. The uncorrupted parts of a folio should still be accessible, so page-cache lookups need to still work. Dan Williams said that, in the DAX code (which implements direct access to files in persistent memory), the approach taken is to inform the filesystem of the error and still remove the page from the page cache. That makes it possible to return errors to the user, he said; this might also be a good approach for errors in regular memory as well. 

Ted Ts'o expressed his agreement with Williams, saying that if the information about a corrupt page exists only in memory, a crash will erase any knowledge of the problem; that, too, leads to silent data corruption. The proposed solution does a lot of work, he said, to return `EIO` only until the next reboot happens. Asking the filesystem to maintain this information is more work, but may be the better approach in the end. One way to make it easier, he said, would be to not worry about tracking corrupted pages individually; instead, the file could just be marked as having been corrupted somewhere. 

Shi argued that memory failures are not particularly rare in large data-center environments, and that any of his approaches would be better than doing nothing. Also, he said, users may well care about _which_ page in a file has been damaged, so just marking the file as a whole may not be sufficient. 

Kent Overstreet said that, beyond returning an indication of the problem to the user, the point of this work is to avoid writing garbage back to the disk. Then, if the system crashes, "the slate is wiped clean" and the corrupted memory no longer exists. A crash, he said, might be seen as the best case. Wilcox replied that this "best case" still involves data loss. 

Josef Bacik said that storing corruption information made the most sense to him; the implementation could mostly go into the generic filesystem code. When notified of problems, the filesystem code should mark the affected pages, refuse to return data from them, and take care to avoid writing them to backing store. But he suggested that a per-file flag might suffice; developers — in both user and kernel space — are not good at dealing with error cases, so this mechanism should be kept simple, especially at the beginning. Developers can "try to be fancy" later if it seems warranted. 

David Hildenbrand objected that a per-file flag could get in the way of virtual machines running from images stored in a single file. A single error would prevent the whole thing from being used, essentially killing the virtual machine. Tracking individual pages is probably better for that use case. But Bacik reiterated that the community was destined to make mistakes, so the simple case should be done first. 

As time ran out, Wilcox pointed out that filesystems could handle the case of writing to a corrupted page — if the entire page is being overwritten. In that case, the damaged data is gone and the file is, once again, in the state that the user intended. Goldwyn Rodrigues pointed out, though, that the situation is not so simple for copy-on-write filesystems, which may still have the damaged pages sitting around. Bacik said this case is exactly why he opposes fancy solutions.
