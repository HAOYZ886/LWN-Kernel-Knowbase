---
title: Persistent memory
url: https://lwn.net/Articles/591779/
date: "March 26, 2014"
category: "Memory management-Nonvolatile memory"
author: "By Jake Edge March 26, 2014 2014 LSFMM Summit"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
March 26, 2014

* * *

[2014 LSFMM Summit](<https://lwn.net/Articles/LSFMM2014/>)

Matthew Wilcox and Kent Overstreet talked about support for persistent memory in the kernel on the first day of the 2014 Linux Storage, Filesystem, and Memory Management Summit held in Napa, California. There have been, well, persistent rumors of the imminent availability of persistent memory for some time, but Wilcox said you can actually buy some devices now. He wanted to report on some progress he had made on supporting these devices as well as to ask the assembled developers for their thoughts on some unresolved issues. 

Persistent memory is supposed to be as fast as DRAM, but to retain its contents even without power. To support these devices, Wilcox has written a ["direct access" block layer that is called DAX](<https://lwn.net/Articles/591646/>) ("it has an 'X', which is cool", he said—it also is a three-letter acronym that is not used by anything else in the kernel). The idea behind DAX came from the execute-in-place (XIP) code in the kernel, not because the data accessed from persistent memory will be executed, necessarily, but because it should avoid the page cache. XIP originally came from IBM, which wanted to share executables and libraries between virtual machines, but it has also been used in the embedded world to execute directly from ROM or flash. 

Since persistent memory is as fast as RAM, it doesn't make sense to put another copy into memory as a page cache entry. XIP seemed like a logical starting point, since it avoided the page cache, but it required a lot of work to make it suitable for persistent memory. So Wilcox rewrote it and renamed it. Filesystems will make calls to the `direct_access()` block device operation in a DAX driver to access data from the device without it ending up in the page cache. Wilcox would like to see DAX merged, so he was encouraging people in the room to look at the code and comment. 

But there are a few problem areas still. Currently, calling `msync()` to flush a range of memory to persistent storage will actually sync the entire file and metadata. That is not required by POSIX and Wilcox would like to change the behavior to just sync the range in question. Obviously that has a much further reach than just affecting DAX, and Peter Zijlstra cautioned that changing sync behavior can surprise user space, pointing to "`fsync()` wars from a few years back" as an example. User space often doesn't care what is supposed to be done, instead it depends on the existing semantics, he said. 

Wilcox said that kernel developers "suck at implementing [syncing], user space sucks at using it" and concluded that "syncing sucks". The consensus seemed to be that any application that was syncing a range, but depending on the whole file being synced, is broken. Furthermore, Chris Mason was all in favor of fixing `msync()` for ranges as it would "make filesystem guys look good". 

Another problem area is with the `MAP_FIXED` flag for `mmap()`. It has two meanings, one of which is not very well known, he said. `MAP_FIXED` means to map the pages at the address specified, which is expected. But it also means to unmap any pages that are in the way of that mapping, which is surprising. Someone must have wanted that behavior at one time, but no one wants it any more, he said. He has proposed a `MAP_WEAK` flag that would only map the memory if nothing else is occupying the address range. 

The `get_user_pages()` function cannot be used with persistent memory, because there are no `struct page` entries created for it. There could be a lot of pages in a persistent memory device, so wasting 64 bytes per page for a mostly unused `struct page` is not desirable. The call to `get_user_pages()` is generally for I/O, so Dave Hansen has been working on a `get_user_sg()` that create a scatter-gather list for doing I/O. The crypto subsystem also wants this capability, Wilcox said. 

There is a problem, though. A `truncate()` operation could remove blocks out from under `get_user_sg()`, which would leave a mess behind. Wilcox wondered if file truncation could just be blocked until the pages are no longer pinned by the I/O operation. That did not seem popular, but Overstreet had another idea. 

Overstreet has been working on a direct I/O rewrite for some time and, in many ways, doing a DAX mapping and a direct I/O look similar, he said. His rewrite would create a new `struct bio` that would be the container for the I/O. It would get rid of the `get_block()` callback, which is, he said, a horrible interface. For one thing, it may have to read the mapping from disk, which should be asynchronous, but `get_block()` isn't. Moving to `struct bio` would allow the usual block-layer filesystem locking to avoid the `truncate()`. 

There were some complaints that making I/O be `bio`-based was problematic for filesystems like NFS and CIFS that don't use the `bio` structure. Overstreet said that we may get to a point where buffered I/O lives atop direct I/O, which would help that problem. In addition, Mason did not think that a `bio`-based interface would really be that big of a problem for NFS and others. A `bio` is just a container of pages, Overstreet said. 

In the end, no really clear conclusions were drawn. It would seem that folks need to review the DAX code (and, eventually, Overstreet's direct I/O rewrite) before reaching those conclusions. 

[ Thanks to the Linux Foundation for travel support to attend LSFMM. ]
