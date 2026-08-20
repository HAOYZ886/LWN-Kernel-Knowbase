---
title: Toward improved page replacement
url: https://lwn.net/Articles/226756/
date: "March 20, 2007"
category: "Memory management-Page replacement algorithms"
author: 
---

When memory gets tight (a situation which usually comes about shortly after starting an application like tomboy), the kernel must find a way to free up some pages. To an extent, the kernel can free memory by cleaning up its own internal data structures - reducing the size of the inode and dentry caches, for example. But, on most systems, the bulk of memory will be occupied by user pages - that is what the system is there for in the first place, after all. So the kernel, in order to accommodate current demands for user pages, must find some existing pages to toss out. 

To help in the choice of pages to remove, the kernel maintains two big linked lists for each memory zone. The "active" list contains pages which have been recently accessed, while the "inactive" list has those which have not been used in the recent past. When the kernel looks for pages to evict, it will scan through the inactive list, in the theory that the pages least likely to be needed soon are to be found there. 

There is an additional complication, though: there are two fundamental types of pages to be found on these lists. "Anonymous" pages are those which are not associated with any files on disk; they are process memory pages. "Page cache" pages, instead, are an in-memory representation of (portions of) files on the disks. A proper balance between anonymous and page cache pages must be maintained, or the system will not perform well. If either type of page is allowed to predominate at the expense of the other, thrashing will result. 

The kernel offers a knob called [swappiness](<http://lwn.net/Articles/83588/>) which controls how this balance is struck. If the system administrator sets a higher value of swappiness, the kernel will allow the page cache to occupy a larger portion of memory. Setting swappiness to a very low value is a way to tell the kernel to keep anonymous pages around at the expense of the page cache. In general, the system can be expected to perform better if page cache pages are reclaimed first; they can often be reclaimed without needing to be written back to disk, and their layout on the disk can make recovery faster should they be needed again. For this reason, the default value for swappiness favors the eviction of page cache pages; anonymous pages will only be targeted when memory pressure becomes relatively severe. 

Swappiness clearly affects how the process of scanning pages for eviction candidates is done. If swappiness is low, anonymous pages will simply be passed over. As it turns out, this behavior can lead to performance problems; there may be a lot of anonymous pages which must be scanned over before the kernel finds any page cache pages, which are the ones it was looking for in the first place. It would be nice to avoid all of that extra work, especially since it comes at a time when the system is already under stress. 

Rik van Riel has posted [a patch](<http://lwn.net/Articles/226658/>) which tries to improve this situation. The approach taken is quite simple: the active and inactive lists are each split into two new lists: one pair (active and inactive) for anonymous pages and one pair for page cache pages. With separate lists for the page cache, the kernel can go after those pages without having to iterate over a bunch of uninteresting anonymous pages on the way. The result should be better scalability on larger systems. 

The idea is simple, but the patch is reasonably large. Any code which puts pages onto one of the lists must be changed to specify which list is to be used; that requires a number of small changes throughout the memory management and filesystem code. Beyond that, the current patch does not really change how the page reclamation code works, though Rik does note: 

For now the swappiness parameter can be used to tweak swap aggressiveness up and down as desired, but in the long run we may want to simply measure IO cost of page cache and anonymous memory and auto-adjust. 

There tends to be a lot of sympathy for changes which remove tuning knobs in favor of automatic adaptation within the kernel itself. So if this approach could be made to work, it might well be adopted. Getting system tuning right is hard; it's often better if the computer can figure it out by itself. 

Meanwhile, the list-splitting patch, so far, lacks widespread testing or benchmarking. So, at this point, it is difficult to say when (or in what form) this patch will find its way into the mainline.
