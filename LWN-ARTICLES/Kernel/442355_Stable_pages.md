---
title: Stable pages
url: https://lwn.net/Articles/442355/
date: "May 11, 2011"
category: Data integrity; Stable pages
author: "By Jonathan Corbet May 11, 2011"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 11, 2011

When a process writes to a file-backed page in memory (through either a memory mapping or with the `write()` system call), that page is marked dirty and must eventually be written to its backing store. The writeback code, when it gets around to that page, will mark the page read-only, set the "under writeback" page flag, and queue the I/O operation. The write-protection of the page is not there to prevent changes to the page; its purpose is to detect further writes which would require that another writeback be done. Current kernels will, in most situations, allow a process to modify a page while the writeback operation is in progress. 

Most of the time, that works just fine. In the worst case, the second write to the page will happen before the first writeback I/O operation begins; in that case, the more recently written data will also be written to disk in the first I/O operation and a second, redundant disk write will be queued later. Either way, the data gets to its backing store, which is the real intent. 

There are cases where modifying a page that is under writeback is a bad idea, though. Some devices can perform [integrity checking](<https://lwn.net/Articles/290141/>), meaning that the data written to disk is checksummed by the hardware and compared against a pre-write checksum provided by the kernel. If the data changes after the kernel calculates its checksum, that check will fail, causing a spurious write error. Software RAID implementations can be tripped up by changing data as well. As a result of problems like this, developers working in the filesystem area have been convinced for a while that the kernel needs to support "stable pages" which are guaranteed not to change while they are under writeback. 

When LWN [looked at stable pages](<https://lwn.net/Articles/429295/>) in February, Darrick Wong had just posted a patch aimed at solving this problem. In situations where integrity checking was in use, the kernel would make a copy of each page before beginning a writeback operation. Since nobody in user space knew about the copy, it was guaranteed to remain unmolested for the duration of the write operation. This patch solved the problem for the integrity checking case, but all of those copy operations were expensive. Given that providing stable pages in all situations was seen as desirable, that cost was considered to be too high. 

So Darrick has come back with [a new patch set](<https://lwn.net/Articles/442156/>) which takes a different - and simpler - approach. In short, with this patch, any attempt to write to a page which is under writeback will simply wait until the writeback completes. There is no need to copy pages or engage in other tricks, but there may be a cost to this approach as well. 

As noted above, a page will be marked read-only when it is written back; there is also a page flag which indicates that writeback is in progress. So all of the pieces are there to trap writes to pages under writeback. To make it even easier, the VFS layer already has a callback (`page_mkwrite()`) to notify filesystems that a read-only page is being made writable; all Darrick really needed to do was to change how those `page_mkwrite()` callbacks operate in presence of writeback. 

Some filesystems do not provide `page_mkwrite()` at all; for those, Darrick created a generic `empty_page_mkwrite()` function which locks the page, waits for any writeback to complete, then returns the locked page. More complicated filesystems do have `page_mkwrite()` handlers, though, so Darrick had to add similar functionality for ext2, ext4, and FAT. Btrfs has implemented stable pages internally for some time, so no changes were required there. Ext3 turns out to have some complicated interactions with the journal layer which make a stable page implementation hard; since invasive changes to ext3 are not welcomed at this point, that filesystem may never get stable page support. 

There have been concerns expressed that this approach could slow down applications which repeatedly write to the same part of a file. Before this change, writeback would not slow down subsequent writes; afterward, those writes will wait for writeback to complete. Darrick ran some benchmarks to test this case and found a performance degradation of up to 12%. This slowdown is unwelcome, but there also seems to be a consensus that there are very few applications which would actually run into this problem. Repetitively rewriting data is a relatively rare pattern; indeed, the developers involved are [saying](<https://lwn.net/Articles/442369/>) that they don't even know of a real-world case they can test. 

Lack of awareness of applications which would be adversely affected by this change does not mean that they don't exist, of course. This is the kind of change which can create real problems a few years down the line when the code is finally shipped by distributors and deployed by users; by then, it's far too late to go back. If there are applications which would react poorly to this change, it would be good to get the word out now. Otherwise the benefits of stable pages are likely to cause them to be adopted in most settings.
