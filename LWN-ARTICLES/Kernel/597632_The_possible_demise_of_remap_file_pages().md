---
title: The possible demise of remap_file_pages()
url: https://lwn.net/Articles/597632/
date: "May 7, 2014"
category: remap file pages
author: "By Jonathan Corbet May 7, 2014"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
May 7, 2014

The [`remap_file_pages()`](<http://man7.org/linux/man-pages/man2/remap_file_pages.2.html>) system call is a bit of a strange beast; it allows a process to create a complicated, non-linear mapping between its address space and an underlying file. Such mappings can also be created with multiple `mmap()` calls, but the in-kernel cost is higher: each `mmap()` call creates a separate virtual memory area (VMA) in the kernel, while `remap_file_pages()` can get by with just one. If the mapping has a large number of discontinuities, the difference on the kernel side can be significant. 

That said, there are few users of `remap_file_pages()` out there. So few that Kirill Shutemov has posted [a patch set](<https://lwn.net/Articles/597631/>) to remove it entirely, saying ""Nonlinear mappings are pain to support and it seems there's no legitimate use-cases nowadays since 64-bit systems are widely available."" The patch is not something he is proposing for merging yet; it's more of a proof of concept at this point. 

It is easy to see the appeal of this change; it removes 600+ lines of tricky code from the kernel. But that removal will go nowhere if it constitutes an ABI break. Some kernel developers clearly believe that no users will notice if `remap_file_pages()` goes away, but going from that belief to potentially breaking applications is a big step. So there is talk of adding a warning to the kernel; Peter Zijlstra [suggested](<https://lwn.net/Articles/597635/>) going a step further and require setting a sysctl knob to make the system call active. But it would also help if current users of `remap_file_pages()` would make themselves known; speaking now could save some trouble in the future.
