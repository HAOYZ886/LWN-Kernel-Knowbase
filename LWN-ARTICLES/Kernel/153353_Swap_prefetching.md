---
title: Swap prefetching
url: https://lwn.net/Articles/153353/
date: "September 27, 2005"
category: "Memory management-Swapping"
author: 
---

It's a common occurrence: some large application runs briefly and pushes all kinds of useful memory out to swap space. Examples include large `ld` runs, backups, `slocate`, and others. Once the program is done, the Linux system is left with a great deal of free memory, and a substantial amount of useful application data stuck in swap space. When the user tries to use a running application, everything stops while it populates that free memory with its pages. Wouldn't it be nice if the system could restore swapped out pages when the memory becomes available and avoid making the user wait later on? 

A number of attempts have been made at prefetching swapped data in the past. It has proved hard, however, to repopulate memory from swap in a way which does not adversely affect the performance of the system as a whole. A well-intended interactivity optimization can easily turn into a performance hit in real use. Con Kolivas has been making another try at it, however, with a series of prefetch patches based on code originally written by Thomas Schlichter. [Version 11](<https://lwn.net/Articles/152935/>) of the swap prefetch patch was posted on September 23\. 

This patch creates two new data structures to track pages which have been evicted to swap. Each swapped page is represented by a `swapped_entry_t` structure; this structure is added to a linked list and a radix tree. The list enables the prefetch code to find the most recently swapped pages, with the idea that those pages are more likely to be useful in the near future than others which have been languishing in swap for longer. The radix tree, instead, allows the quick removal of entries without having to search the entire (possibly very long) list to find them. 

Whenever a page is pushed out to swap, it is also added to the list and radix tree. There is a limit on how many pages will be remembered; it is currently set to a relatively high value which keeps the swapped page entries from occupying more than 5% of RAM. If that limit is exceeded, an older entry will be recycled. The `add_to_swapped_list()` code also refuses to wait for any locks; if there is a conflict with another processor, it will simply forget a page rather than spin on the lock. The consequence of forgetting a page (it will never be prefetched) is relatively small, so holding up the swap process for contention is not worth it in this case. 

The code which actually performs prefetching is even more timid; every effort has been made to make the process of swap prefetching as close to free as possible. The prefetch code only runs once every five seconds - and that gets pushed back any time there is VM activity. The number of available free pages must be substantially above the minimum desired number, or prefetching will not happen. The code also checks that no writeback is happening, that the number of dirty pages in the system is relatively small, that the number of mapped pages is not too high, that the swap cache is not too large, and that the available pages are outside of the DMA zone. When all of those conditions are met, a few pages will be read from swap into the swap cache; they remain on the swap device so that they can be immediately reclaimed should a sudden shortage of memory develop. 

Con claims that the end result is worthwhile: 

In testing on modern pc hardware this results in wall-clock time activation of the firefox browser to speed up 5 fold after a worst case complete swap-out of the browser on an static web page. 

That seems like a benefit worth having, if the cost of the prefetch code is truly low. Discussion on the list has been limited, suggesting that developers are unconcerned about the impacts of prefetching - or simply uninterested at this point.
