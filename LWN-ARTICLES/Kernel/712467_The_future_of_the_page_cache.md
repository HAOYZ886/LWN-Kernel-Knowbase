---
title: The future of the page cache
url: https://lwn.net/Articles/712467/
date: "January 25, 2017"
category: "Memory management-Page cache"
author: "By Jonathan Corbet January 25, 2017 linux.conf.au 2017"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
January 25, 2017

* * *

[linux.conf.au 2017](<https://lwn.net/Archives/ConferenceByYear/#2017-linux.conf.au>)

The promise of large-scale persistent memory has forced a number of changes in the kernel and has raised questions about whether the kernel's page cache will be needed at all in the future. In his linux.conf.au 2017 talk, Matthew Wilcox asserted that not only do we still need the page cache, but that its role should be increased. First, though, there is the small matter of correcting a mistake made by a certain Mr. Wilcox a couple of years ago. 

This was, he started, his first talk ever as a Microsoft employee — something he thought he would never find himself saying. He then launched into his topic by saying that computing is all about caching. His new laptop can execute 10 billion instructions per second, but only as long as it doesn't take a cache miss. Memory on that system can only deliver 530 million cache lines per second, so it doesn't take many cache misses to severely impact its performance. Things get even worse if the data you want isn't cached in main memory and has to be read from a storage device, even a fast solid-state device. 

It has always been that way; a PDP-11 was also significantly slowed by cache misses. But the problem is getting worse. CPU speeds have increased more than memory speeds, which, in turn, have increased more than storage speeds. The cost of not caching your data properly is thus going up. 

#### The page cache

Unix systems have had a buffer cache, which sits between the filesystem and the disk for the purpose of caching disk blocks in memory, for a long time. While preparing the talk, he went back to look at Sixth-edition Unix (released in 1975) and found a buffer cache there. Linux has had a buffer cache since the beginning. In the 1.3.50 release in 1995, Linus Torvalds added a significant innovation in the form of the page cache. This cache differs from the buffer cache in that it sits between the virtual filesystem (VFS) layer and the filesystem itself. With the page cache, there is no need to call into filesystem code at all if the desired page is present already. Initially, the page and buffer caches were entirely separate, but Ingo Molnar unified them in 1999. Now, the buffer cache still exists, but its entries point into the page cache. 

The page cache has a great deal of functionality built into it. There are some obvious functions, like finding a page at a given index; if the page doesn't exist, it can be created and optionally filled from disk. Dirty pages can be pushed back to disk. Pages can be locked, unlocked, and removed [![\[Matthew Wilcox\]](https://static.lwn.net/images/conf/2017/lca/MatthewWilcox-sm.jpg)](<https://lwn.net/Articles/712504/>) from the cache. Threads can wait for changes in a page's state, and there are interfaces to search for pages in a given state. The page cache is also able to keep track of errors associated with persistent storage. 

Locking for the page cache is handled internally. There tends to be disagreement in the kernel community over the level at which locking should be handled; in this case it has been settled in favor of internal locking. There is a spinlock to control access when changes are being made to the page cache, but lookups are handled using the lockless read-copy-update (RCU) mechanism. 

Caching is the art of predicting the future, he said. When the cache grows too large, various heuristics come into play to decide which pages should be removed. Pages used only once are likely to not be used again, so those are kept in the "inactive" list and pushed out relatively quickly. A second use will promote a page from the inactive list to the active list. Unused pages eventually age off the active list and are put back onto the inactive list. Exceptional ["shadow" entries](<https://lwn.net/Articles/495543/>) are used to track pages that have fallen off the end of the inactive list and have been reclaimed; these entries have the effect of lengthening the kernel's memory about pages that were used in the relatively distant past. 

Huge pages have been a challenge for the page cache for a while. The kernel's [transparent huge page feature](<https://lwn.net/Articles/423584/>) initially only worked with anonymous (non file-backed) memory. There are good reasons for using huge pages in the page cache, though. Initial work in this area simply adds a large set of single-page entries to the page cache to correspond to a single huge page. Wilcox concluded that this approach was "silly"; he [enhanced the radix tree code](<https://lwn.net/Articles/684864/>), used to track pages in the page cache, to be able to handle huge-page entries directly. Pending patches will cause the page cache to use a single entry for huge pages. 

#### Do we still need the page cache?

Recently, Dave Chinner [asserted](<https://lwn.net/Articles/704487/>) that there was no longer a need for a page cache. He noted that the DAX subsystem, initially created by Wilcox to provide direct access to file data stored in persistent memory, bypasses the page cache entirely. "There is nothing like having your colleagues question your entire motivation", Wilcox said. There are people who disagree with Chinner, though, including Torvalds, who [popped up](<http://www.realworldtech.com/forum/?threadid=162139&curpostid=162588>) in a separate forum saying that the page cache is important because good things don't come from having low-level filesystem code in the critical path for data access. 

With that last statement in mind, Wilcox delved into how an I/O request using DAX works now. He designed the original DAX code and, in so doing, concluded that there was no need to use the page cache. That decision, he said, was wrong. 

In current kernels, when an application makes a system call like `read()` to read some data from a file stored in persistent memory, DAX gets involved. Since the requested data is not present in the page cache, the VFS layer calls the filesystem-specific `read_iter()` function. That, in turn, calls into the DAX code, which will call _back_ into the filesystem to turn the file offset into a block number. Then the block layer is queried to get the location of that block in persistent memory (mapping it into the kernel's address space if need be) so that the block's contents can be copied back to the application. 

That is "not awful", but it should work differently, he said. The initial steps would be the same, in that the `read_iter()` function would still be called, and it would call into the DAX code. But, rather than calling back into the filesystem, DAX should call into the page cache to get the physical address associated with the desired offset in the file. The data is then copied back to user space from that address. This all assumes that the information is already present in the page cache but, when that is the case, the low-level filesystem code need not get involved at all. The filesystem had already done the work, and the page cache had cached the result. 

When Torvalds wrote the above-mentioned post about the page cache, he said: 

It's also a major disaster from a locking standpoint: trust me, if you think your filesystem can do fine-grained locking right when it comes to things like concurrent lookup of pathnames, you're living in a dream world. 

This, Wilcox said, was "so right"; the locking in DAX has indeed been disastrous. He originally thought it would be possible to get away with relatively simple locking, but complexity crept in with each new edge case that was discovered. DAX locking is now "really ugly" and he is sorry that he made the mistake of thinking that he could bypass the page cache. Now, he said, he has to fix it. 

#### Future work

He concluded with a number of enhancements he would like to see made around DAX and the page cache. The improved huge-page support mentioned above is one of them; that is already sitting in the -mm tree and should be done soon. The use of page-frame numbers instead of `page` structures has been [under discussion](<https://lwn.net/Articles/672457/>) for a while since there is little desire to make the kernel store vast numbers of `page` structures for large persistent memory arrays. 

He would like to revisit the idea of filesystems with a block size larger than the system's page size. That is something that people have wanted for many years; now that the page cache can handle more than one page size, it should be possible. "A simple matter of coding", he said. He is looking for other interested developers to work with on this project. 

Huge swap entries are also an area of interest. We have huge anonymous pages in memory but, when it comes time to swap them out, they get broken up into regular pages. "That is probably the wrong answer". There is work in improving swap performance, but it needs to be reoriented toward keeping huge pages together. That might help with the associated idea of swapping to persistent memory. Data in a persistent-memory swap space can still be accessed, so it may well make sense to just leave it there, especially if it is not being heavily modified. 

[The video of this talk](<https://www.youtube.com/watch?v=xxWaa-lPR-8>), including a bonus section on page-cache locking, is available. 

[Your editor would like to thank linux.conf.au and the Linux Foundation for assisting with his travel to the event.]
