---
title: "A nasty file corruption bug - fixed"
url: https://lwn.net/Articles/215868/
date: "December 31, 2006"
category: "Block layer-Buffer heads; Debugging; Filesystems-ext3"
author: 
---

The December 20 LWN Kernel Page contained [an article](<http://lwn.net/Articles/215113/>) about a file corruption bug generally (but not exclusively) seen with ext3 filesystems. Certain applications which have unusual patterns of access to memory-mapped files could, at times, see gaps where data had not made it all the way to the disk. The rtorrent tool was one such application; other test cases were found (and developed) as the hunt for this problem intensified. The problem is now solved, but it offers some interesting lessons on how this kind of subtle bug can come about - and how to get it fixed. 

![\[Cheezy diagram\]](https://static.lwn.net/images/ns/kernel/DirtyPageBug1.png) In an attempt to explain what was going on, your editor will once again employ his rather dubious artistic skills. To that end, readers are kindly requested to look at the diagram to the right and suspend enough disbelief to imagine that it represents a page in memory - a page containing interesting data, and which represents an equivalent set of blocks found within a file on the disk. The distinction between the page and its component blocks is important, which is why the dotted lines divide up the page. A 4096-byte page in memory is likely represented by eight 512-byte disk blocks (which are, most likely, merged back together by the drive, but we'll pretend that isn't happening). 

There are a couple of different kernel data structures which contain information about this page, making the diagram a bit more complicated: 

> ![\[Second diagram\]](https://static.lwn.net/images/ns/kernel/DirtyPageBug2.png)

The page may be mapped into one or more process address spaces. For each such mapping, there will be a page table entry (PTE) which performs the translation between the user-space virtual address and the physical address where the page actually lives. There is also some other information in the PTE, including a "dirty" bit. When the application modifies the page, the processor will set the dirty bit, allowing the operating system to respond by (for example) writing the page back to its backing store. Note that, if there are multiple PTEs pointing to a single page, they may well disagree on whether the page is dirty or not. The only way to know for sure is to scan all existing PTEs and see if any of them are marked dirty. 

The kernel maintains a separate data structure known as the system memory map; it contains one `struct page` for every physical page known to exist. This structure contains a number of interesting bits of information, including a pointer to the page's backing store (if any), a data structure allowing the associated PTEs to be found relatively easily, and a set of page flags. One of those flags is a dirty bit - another flag which notes that the page is in need of writing to its backing store. (For those following closely, it may be worth pointing out that the red arrow pointing to the page does not actually exist as a pointer field; it is implicit in the structure's position within the memory map). 

Finally, there is another set of structures which may be associated with this page: 

> ![\[Third diagram\]](https://static.lwn.net/images/ns/kernel/DirtyPageBug3.png)

The "buffer head" (or "bh") goes back to the earliest days of Linux. It can be thought of as a mapping between a disk block and its copy in memory. The bh is not central to Linux memory management in the way it once was, but a number of filesystems still use it to handle their disk I/O tracking. Note that there is not necessarily a bh structure for every block found within a page; if a filesystem has reason to believe that only some blocks need writing, it does not need to create bh structures for the rest. Among other things, the bh structure contains yet another dirty flag. 

With all of these different flags representing what is essentially the same information, it is not entirely surprising that some confusion eventually came about. The maintenance of redundant data structures can be a challenge in any setting, and the kernel environment adds difficulties of its own. 

Deep within the kernel, there is a function called `set_page_dirty()`; it is used by the memory management code when it notices (via a PTE or a direct application operation) that a page is in need of writeback. Among other things, it copies the dirty bit from the page table entries into the `page` structure. If the page is part of a file, `set_page_dirty()` will call back into the relevant filesystem \- but only if said filesystem has provided the appropriate method. Many filesystems do not provide `set_page_dirty()` callback, however; for these filesystems, the kernel will, instead, traverse the list of associated bh structures and mark each of them dirty. 

And that is where the problem comes in. The filesystem may well have noticed that a block represented by a given bh was dirty and started I/O on it before the `set_page_dirty()` call. When the I/O is complete, the filesystem clears the dirty flag in the bh. If the `set_page_dirty()` call comes while the I/O on the block is active, the filesystem will not notice the fact that the block's data may have changed after it was written. Instead, the block will be marked clean, even though what was written does not correspond to what is currently in memory. File corruption results. 

Linus's [fix](<https://lwn.net/Articles/215870/>) is simple. When the virtual memory subsystem decides that it is time to write a page, a new call to `set_page_dirty()` is made. That ensures that all buffer heads will be marked dirty at the time the filesystem's `writepage()` method is called. That change ensures that all blocks of the page will be written; testers have confirmed that it makes the file corruption problems go away. The patch has gone into the mainline git repository; it should show up in the next 2.6.19 stable update as well. 

The longer-term solution is to continue pushing buffer heads out of the kernel's I/O paths. As Linus [puts it](<https://lwn.net/Articles/215871/>): 

The buffer head has been purely an "IO entity" for the last several years now, and it's not a cache entity. Anybody who does writeback by buffer heads is basically bypassing the real cache (the page cache), and that's why all the problems happen. 

I think ext3 is terminally crap by now. It still uses buffer heads in places where it really really shouldn't, and as a result, things like directory accesses are simply slower than they should be. Sadly, I don't think ext4 is going to fix any of this, either. 

Ted Ts'o [responds](<https://lwn.net/Articles/215872/>) that a fix for ext4 could yet happen, but it involves other filesystems as well. The ext3 filesystem is probably going to stay with buffer heads, however, meaning that the kernel will have to continue to work with them indefinitely. 

Finally, this story illustrates just how hard it can be to track down and fix certain kinds of kernel bugs. Early in the process it was hard for the interested developers to reproduce the problem, so they had to rely on the initial reporters to try out various patches. Those reporters stuck with the process, building and testing a _lot_ of kernels before the problem was flushed out. They deserve much of the credit for the resolution of this problem.
