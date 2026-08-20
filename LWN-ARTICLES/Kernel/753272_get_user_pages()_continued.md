---
title: get_user_pages() continued
url: https://lwn.net/Articles/753272/
date: "May 4, 2018"
category: "Memory management-get user pages"
author: "By Jonathan Corbet May 4, 2018 LSFMM"
---

By **Jonathan Corbet**  
May 4, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

At [a plenary session](<https://lwn.net/Articles/753027/>) held relatively early during the 2018 Linux Storage, Filesystem, and Memory-Management Summit, the developers discussed a number of problems with the kernel's `get_user_pages()` interface. During the waning hours of LSFMM, a tired (but dedicated) set of developers convened again in the memory-management track to continue the discussion and try to push it toward a real solution. 

Jan Kara and Dan Williams scheduled the session to try to settle on a way to deal with the issues associated with `get_user_pages()` — in particular, the fact that code that has pinned pages in this way can modify those pages in ways that will surprise other users, such as filesystems. During the first session, Jérôme Glisse had suggested using the MMU notifier mechanism as a way to solve these problems. Rather than pin pages with `get_user_pages()`, kernel code could leave the pages unpinned and respond to notifications when the status of those pages changes. Kara said he had thought about the idea, and it seemed to make some sense. 

His current thinking is to audit all existing `get_user_pages()` callers and see which of those could be changed to use notifiers instead. Changing away from `get_user_pages()` would not be mandatory for device drivers (or other code) that couldn't handle that mode of operation. That leaves open the question of how to solve the problems for code that cannot be converted; in the worst case, operations on affected pages might just have to hang until all references to the pages in question are dropped. 

The problem there is it's not always easy to know whether there are references to a page created by `get_user_pages()` or not. With memory accessed via DAX, life is relatively simple, and one can just wait until the reference count drops to one. For page-cache pages it's harder; it would be necessary to compare the reference and map counts for each page of interest. Glisse suggested just forcing `get_user_pages()` to lock the pages as it pins them. That would "be mean" to `get_user_pages()` callers, he said, but he thought that was a fine idea. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Hugh Dickins worried that this change would result in reduced performance and the introduction of kernel regressions. But Glisse said that only "legacy" code would be affected, and perhaps that is not a problem. An alternative might be to try to find some bits in `struct page` that could be used to track these uses, but there is not a lot of space available. Another possibility might be to create a special type of virtual memory area (VMA) for use with `get_user_pages()`. 

One potential problem is interference with `get_user_pages_fast()`, which attempts to pin the pages without taking locks. Adding those locks to avoid contention with the MMU notifiers would cause it to not be fast anymore. Glisse, after trying a couple of suggestions, conceded that MMU notifiers are not going to work with `get_user_pages_fast()`; he said that he was "running out of bad ideas". Dave Hansen suggested creating some sort of mechanism based on read-copy-update for `get_user_pages_fast()` users, but agreed that the idea "sounds terrifying". 

In the end, the apparent conclusion was that Kara will start by experimenting with page locks and, maybe, RCU. Patches should be forthcoming.
