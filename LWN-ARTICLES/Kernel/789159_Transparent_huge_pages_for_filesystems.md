---
title: Transparent huge pages for filesystems
url: https://lwn.net/Articles/789159/
date: "May 22, 2019"
category: Huge pages
author: "By Jake Edge May 22, 2019 LSFMM"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
May 22, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

One thing that is known about using transparent huge pages (THPs) for filesystems is that it is a hard problem to solve, but is there a solid first step that could be taken toward that goal? That is the question Song Liu asked to open his combined filesystem and memory-management session at the 2019 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM). His employer, Facebook, has a solid use case for using THPs on files in the page cache, which may provide a starting point. 

THPs reduce translation lookaside buffer (TLB) misses so they provide better performance. Facebook is trying to reduce misses on the TLB for instructions by putting hot functions into huge pages. It is using the [Binary Optimization and Layout Tool](<https://github.com/facebookincubator/BOLT>) (BOLT) to profile its code in order to identify the hot functions. Those functions are collected up into an 8MB region in the generated executable. 

[ ![\[Song Liu\]](https://static.lwn.net/images/2019/lsf-liu-sm.jpg) ](<https://lwn.net/Articles/789161/>)

At run time, the application creates an 8MB temporary buffer and the hot section of the executable memory is copied to it. The 8MB region in the executable memory is then converted to a huge page (by way of an `mmap()` to anonymous pages and an `madvise()` to create a huge page), the data is copied back to it, and it is made executable again using `mprotect()`. 

This results in a 5-10% performance boost without requiring any kernel changes to support it. But it breaks the symbol addresses and uprobe targets in the THP region because the kernel has no idea this region is part of the application any more. If there were support for THPs in the filesystem, that whole dance could be eliminated; a simple `madvise()` could be used. 

Liu calls making it work without the copy a "baby step" toward THP support for filesystems. He is working on it, but there are lots of limitations and simplifications in his approach. For example, there is no support for writing to the THP, thus no writeback is required. That would prove to be a sticking point. 

An attendee asked why writing was not supported; is it a performance issue or a complexity problem? Johannes Weiner said that adding write support would mean touching all of the places where a page is expected. In particular, all of the filesystems would need to change to support write. Instead of a "massive patch" that would update everything at the same time, this is simply a first step, not a permanent solution. 

There were concerns expressed by Kirill Shutemov and Matthew Wilcox about merging something that did not support writing. For the Facebook use case, writing is not needed at all, Chris Mason said, but that is not a general enough solution, Wilcox said. Rik van Riel said that everyone needed to keep in mind that it will be impossible to merge all of the feature at once—it will be too big of a patch set. So there is a need to identify the first steps to take. But Shutemov and Wilcox were adamant that nothing should be added to kernel unless writing to the THP was supported. 

Wilcox said that some of the [changes he is working on](<https://lwn.net/Articles/787388/>) for the page cache may help simplify the problem for filesystems. In particular, eliminating functions that return tail pages for [compound pages](<https://lwn.net/Articles/619514/>), so that the filesystem code only needs to deal with head pages, will help. He suggested waiting for that work to get finished before proceeding further down the THP for filesystems path; Shutemov agreed with that approach. That may not have been quite what Liu was looking for from the session, but Facebook will presumably keep using its approach in the interim.
