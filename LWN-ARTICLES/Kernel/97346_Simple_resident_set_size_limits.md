---
title: Simple resident set size limits
url: https://lwn.net/Articles/97346/
date: "August 10, 2004"
category: "Memory management-Page replacement algorithms; Resident set-Size limits"
author: 
---

One of the problems which can afflict any virtual memory system is a process which expands to fill all of memory. All it takes is, say, a quick OpenOffice session, and everything else running on the system finds itself shoved into a corner of memory and pushed out onto swap. Avoiding this problem is a simple matter of limiting the amount of physical memory that any given process can occupy, but Linux lacks such limits. 

Rik van Riel seems to have started off on a series of relatively simple patches which address immediate VM issues. His latest [patch](<https://lwn.net/Articles/96859/>) implements resident set size limits for Linux processes. Once this patch is applied, a bit of appropriate limit setting could do a lot to keep those memory hog processes in their place. 

The core of the patch comes down to two lines: 

```
if (mm->rss > mm->rlimit_rss)
    	referenced = 0;
```

This code appears in the function `page_referenced_one()`, which tries to decide whether a process has actually made use of one of its in-core pages. If the page has not been referenced, it goes directly onto the list of pages to reclaim. All that this particular patch is doing is pretending that a process which has exceeded its maximum resident set size has not actually used any of its pages; as a result, the memory hog's pages will be the first ones to be reclaimed. 

This patch applies on top of the token-based mechanism discussed [last week](<https://lwn.net/Articles/95591/>). It modifies that code by depriving a process of the swap token once it goes over its memory limit. 

Many systems in the past have chosen to implement hard resident set size limits. On such systems, a process which incurs a page fault will, if it's at its memory limit, immediately surrender one other page back to the memory management system. Rik's patch works differently, in that there are no hard limits. If there is no particular memory pressure, a process can grow to any size. The limit is only applied when the system starts looking for pages to reclaim for other users. This approach is simple, which is always good; it also allows the system to make full use of its memory when there is not a lot of contention.
