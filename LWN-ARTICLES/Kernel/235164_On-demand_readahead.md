---
title: "On-demand readahead"
url: https://lwn.net/Articles/235164/
date: "May 21, 2007"
category: "Memory management-Readahead; Readahead"
author: 
---

"Readahead" is the act of speculatively reading a portion of a file's contents into memory in the expectation that a process working with that file will soon want that data. When readahead works well, a data-consuming process will find that the information it needs is available to it when it asks, and that waiting for disk I/O is not necessary. The Linux kernel has done readahead for a long time, but that does not mean that it cannot be done better. To that end, Fengguang Wu has been working on a set of "adaptive readahead" patches for a couple of years. 

Adaptive readahead was [covered here](<http://lwn.net/Articles/155510/>) in 2005. The patches have been languishing in the -mm tree for one simple reason: their complexity is at such a level that few people are able to review them in any useful way. The new [on-demand readahead patch](<http://lwn.net/Articles/234784/>) is a response to a request from Andrew Morton for a simpler patch to help get the merge process going. The new code is indeed simpler, having dispensed with much of the logic found in the full adaptive readahead mechanism. 

To a great extent, the on-demand patch reimplements what Linux readahead does currently, but in a simpler and more flexible way. Like the current code, the on-demand patch maintains a "readahead window" consisting of a portion of the file starting with the application's last read. Pages inside the readahead window should already be in the page cache - or, at least, under I/O to get there as soon as possible. The window moves forward as the application reads data from the file. 

The current code actually implements two windows, being the "current window" (a set of in-cache pages which includes the application's current position) and the "ahead window," being the pages most recently read in by the kernel. Once the application's position crosses from the current window into the ahead window, a new I/O operation is started to make a new ahead window. In this way, the kernel tries to always keep sufficiently far ahead of the application that the file data will be available when requested. 

The on-demand patch, instead, has a single readahead window. Rather than maintain a separate "ahead window," the new readahead code marks a page (using a flag in the `page` structure) as being at the "lookahead index." When an application reads its way into the marked page, the readahead window is extended and a new I/O operation is started. There is some resistance to the idea of using a page flag, since those bits are perennially in short supply. Andrew Morton has [suggested](<https://lwn.net/Articles/235175/>) using some more approximate heuristics instead. That approach might occasionally make the wrong decision, but the penalty is low and does not affect the correctness of the system's operation as a whole. 

While the on-demand patch appears to do relatively little, it does have the advantage of removing a bunch of complexity from the current readahead code. It is able to make its decisions without the overhead of trying to track events like an attempted readahead of pages which are already in the cache. The checks for sequential access are made less strict as well, causing readahead to stay active in situations where the current code would turn it off. The result, according to [some benchmarks](<https://lwn.net/Articles/235181/>) posted with the patch, is improvements in application speed between 0.1% and 8% or so - with some performance regressions in some cases. Interestingly, some of the best results come with a benchmark running on a MySQL database, which is not where one would normally expect to see a lot of sequential activity. 

This patch set is clearly simple enough to be reviewed; in the absence of any strong objections, it could conceivably be ready for 2.6.23. Then, perhaps, Fengguang can start working on adding some of the more complex logic which makes up the full adaptive readahead mechanism.
