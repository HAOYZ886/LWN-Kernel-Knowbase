---
title: "get_user_pages() and COW, 2022 edition"
url: https://lwn.net/Articles/895439/
date: "May 20, 2022"
category: "Memory management-get user pages"
author: "By Jonathan Corbet May 20, 2022 LSFMM"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
May 20, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

The numerous [correctness problems with the kernel's `get_user_pages()` functionality](<https://lwn.net/Kernel/Index/#Memory_management-get_user_pages>) have been a fixture at the Linux Storage, Filesystem, Memory-management and BPF Summit (LSFMM) for some years. The [2022 event](<https://events.linuxfoundation.org/lsfmm/>) did not break that tradition. The first-day discussion on page pinning was [covered here](<https://lwn.net/Articles/894390/>). On the final day, in the memory-management track, David Hildenbrand led a session on the current status of `get_user_pages()` and its interaction with copy-on-write (COW) memory. 

COW pages, he began, are used to share anonymous memory between processes. The memory is marked read-only; should a process write to a page, the write fault will be trapped by the kernel, which will make a private copy for the writing process if more than one reference to the page exists. COW is all relatively easy to implement and understand, at least until `get_user_pages()` enters the picture. That function (along with its variants) will take a reference to the indicated pages, which will then be used to access the pages themselves. There are two modes used with `get_user_pages()`, depending on whether the contents of the pages are to be accessed, or only the `page` structure describing them; not every use requests the correct mode, though. 

References taken by `get_user_pages()` are tracked in the `page_count` field of `struct page` — not in the `mapcount` field used to track mappings of the page (and to decide whether to copy a page when a COW fault happens). In general, he said, the kernel knows little about these references; they are not tracked separately from any other references to pages. 

In 2020, a security problem involving the `vmsplice()` system call was reported and became known as CVE-2020-29374. It relied on a COW page that was ostensibly only mapped once (so `mapcount` was one), but a second reference had been created with `get_user_pages()`. The full story of this vulnerability can be found in [this article](<https://lwn.net/Articles/849638/>). In short, the vulnerability was fixed with a commit that caused other problems and was quickly reverted; this happened several times. There is now a fix of sorts in place, though the hugetlbfs filesystem is still affected. But, Hildenbrand said, nobody cares much about hugetlbfs, which is not used to share data with unprivileged child processes. 

The fix that went upstream looks at `page_count` and will force a copy of a COW page if the value is not one. The `mapcount` field is no longer used for this decision. As a result, the security problem can no longer happen, but the kernel might copy pages more often than it should. There is another side effect, though: when `get_user_pages()` is called on a COW page, `page_count` will be incremented; as a result, any write to the page will force a copy to be made. The caller of `get_user_pages()` will be left with the older copy, though, and will not observe any changes made by the writing process. That can lead to the corruption or loss of data. 

Thus, Hildenbrand said, there are two potential problems with the current solution: the cost of unnecessary copies of COW pages, and the potential for data corruption when a `get_user_pages()` caller ends up with the wrong copy of a COW page. There is [a solution](<https://lwn.net/ml/linux-kernel/20220428083441.37290-1-david@redhat.com/>) being upstreamed now that relies on the new `PG_anon_exclusive` (abbreviated "PAE") page flag and `page_count` to avoid the wrong-copy problem. This flag, if present, indicates that the page is both anonymous and exclusive to a process; Hildenbrand described those pages as "PAE pages". If a page is _not_ PAE, that page _might_ be shared. The rules are that any page that is writable must be PAE, and those pages should never be copied in response to COW faults; additionally, pages can only be pinned (for access to their contents) if they are PAE. If there is a possibility that a given PAE page might be pinned, it will not be shared in settings where it otherwise would be — when a process forks, for example. 

There are various cases that need to be considered here. If the kernel seeks to pin a writable, anonymous page, all is well, but if the page is marked read-only, the kernel must trigger a write fault first. In the case of a read-only, anonymous, non-PAE page, that page must be unshared prior to pinning. "Unsharing" in this case can be thought of as "copy on read"; if the page has a single reference it can be reused, otherwise it will need to be copied. 

There are some other tricky cases, Hildenbrand continued. Transparent huge pages are "nasty", since they can be mapped as base (non-huge) pages as well. Temporary unmapping, as happens when a page is being swapped out or migrated, can create confusion. Concurrent `get_user_pages()` calls (`gup_fast()` in particular) must be handled carefully, since they don't take the page-table lock, which is used to synchronize access to the `PG_anon_exclusive` flag. Care must be taken when migrating pages to ensure that the PAE status is not lost. 

The end result, Hildenbrand said at the end of the session, is not optimal. It works well in the absence of `fork()` calls or the use of [kernel same-page merging](<https://lwn.net/Articles/330589/>) (KSM). But attempts to avoid extra copies can fail at times even if there is only one mapping, and `get_user_pages()` is not always reliable when called concurrently with a process fork. But it is all a step in the right direction; be sure to tune into the 2023 LSFMM for the inevitable update.
