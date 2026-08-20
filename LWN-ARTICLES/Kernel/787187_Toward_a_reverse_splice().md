---
title: Toward a reverse splice()
url: https://lwn.net/Articles/787187/
date: "May 1, 2019"
category: splice
author: "By Jonathan Corbet May 1, 2019 LSFMM"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 1, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

The [`splice()`](<http://man7.org/linux/man-pages/man2/splice.2.html>) system call is, at its core, a write operation; it attempts to implement zero-copy I/O by moving pages from a pipe to a file. At the 2019 Linux Storage, Filesystem, and Memory-Management Summit, Miklos Szeredi described a nascent idea for `rsplice()` — a "reverse splice" system call. There were not a lot of definitive outcomes from this discussion, but one thing was clear: `rsplice()` needs a much better description (and some code posted) before the development community can begin to form an opinion on it. 

A key aspect of `splice()` is that it works with up-to-date buffers of data, meaning that it moves pages already containing data obtained from some source. The reverse-splice operation would, instead, operate on empty buffers that need to be filled from somewhere. It would, in other words, be a read operation rather than a write. It could be used to fill buffers [![\[Miklos Szeredi\]](https://static.lwn.net/images/conf/2019/lsfmm/MiklosSzeredi-sm.jpg)](<https://lwn.net/Articles/787188/>) from a file and feed the result into a pipe, for example. One possible use case, he said, is user-space filesystems, which could use it to feed pages of file data into the kernel (and beyond) without copying the data. He thinks that the idea is "implementable", but was curious to hear what the other developers in the room thought about the idea. 

Rik van Riel worried about page-lifecycle problems. Moving a page of data into the page cache (as `rsplice()` might do) is easy if there are no other users of the page, but what if other processes already have that page in their page tables? Szeredi responded that `rsplice()` can only work if the pages involved have not yet been activated, so no other references to them can exist. 

Matthew Wilcox said that he knows of a use case from his time at Microsoft. It has to do with fetching pages of data from a remote server; there would be value in having an efficient way to place that data into the page cache. Doing this would require adding a `recvpage()` complement to the kernel's internal `sendpage()` method. He hoped that somebody within Microsoft would be able to speak more about this use case in the future. 

Hugh Dickins, instead, recalled that Linus Torvalds has expressed regret about having introduced `splice()` in the first place. Torvalds thought that it was a great idea at the time, but few users have materialized since it was implemented. Adding new system calls that lack users only serves to increase the attack surface of the kernel, he said. There are few people who truly understand `splice()`, which has needed to be "corrected" numerous times over the years. An `rsplice()` call, he said, would likely have to go through the same process before it could be trusted. 

From there the discussion wandered in various directions. There was some questioning of the value of zero-copy interfaces in general, but it does seem to offer benefits on systems with high-bandwidth adapters and huge pages. There was a fair amount of confusion about how `rsplice()` differs from `splice()`, perhaps driving home the point that not many people fully understand `splice()` in the first place. What is needed, it was agreed, was a well-defined use case for this new system call that would help to nail down what it actually does. Then, if an implementation appears shortly thereafter, it will be possible to have a more informed discussion on whether the whole thing makes sense.
