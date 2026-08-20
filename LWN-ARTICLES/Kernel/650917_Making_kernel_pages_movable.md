---
title: Making kernel pages movable
url: https://lwn.net/Articles/650917/
date: "July 14, 2015"
category: "Memory management-Large allocations"
author: "By Jonathan Corbet July 14, 2015"
---

By **Jonathan Corbet**  
July 14, 2015

A longtime aspect of the kernel's memory-management subsystem is that it tends to fragment memory over time. After a system has been running for a while, finding groups of physically contiguous pages can be difficult; that is why kernel code will often go to great lengths to avoid the need for contiguous blocks of pages. But there are times when larger blocks are needed; among other things, the [transparent huge pages feature](<https://lwn.net/Articles/423584/>) requires them. Memory can be defragmented on the fly with the kernel's [memory-compaction mechanism](<https://lwn.net/Articles/368869/>), but compaction is easily thwarted by kernel-allocated pages that cannot be moved out of the way. 

User-space pages are easily migrated; they are accessed via the page tables, so relocating a page is just a matter of changing the appropriate page-table entries. Pages in the system's page cache are also accessed via a lookup, so they can be migrated as well. Pages allocated by a random kernel subsystem or driver, though, are not so easy to move. They are accessed directly using kernel-space pointers and cannot be moved without changing all of those pointers. Because kernel pages are so hard to move, the memory-management subsystem tries to separate them from pages that can be moved, but that separation can be hard to maintain, especially when memory is in short supply in general. A single unmovable page can foil compaction for a large block of memory. 

Solving this problem in any given subsystem will require getting that subsystem's cooperation in the compaction process; that is just what [Gioh Kim's driver-page migration patch series](<https://lwn.net/Articles/650864/>) sets out to do. It builds on some special-case code (introduced in 2012) that makes balloon-driver pages movable; the patches generalize that code so that it may be used in other subsystems as well. 

To make a driver (or other kernel subsystem) support page migration (and, thus, compaction), the first step is to allocate an anonymous inode to represent those pages: 

```
#include <linux/anon_inodes.h>
        
        struct inode *anon_inode_new(void);
```

The only real purpose of this inode appears to be to hold a pointer to an `address_space_operations` structure containing a few migration-related callbacks. The relevant methods are: 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

```
bool (*isolatepage) (struct page *page, isolate_mode_t mode);
        void (*putbackpage) (struct page *page);
        int (*migratepage) (struct address_space *space, struct page *page,
        			struct page *newpage, enum migrate_mode mode);
```

`migratepage()` has been in the kernel (in various forms) since 2.6.16; the other two are new with Gioh's patch. To support compaction of its pages, a kernel subsystem should provide all three of these operations. Once the anonymous inode has been allocated, its `i_mapping->a_ops` field should be set to point to the `address_space_operations` structure containing the above methods. 

Needless to say, only whole pages can be supported in the page-compaction system; memory allocated from slab caches will remain immobile. To make a page movable by the compaction code, a kernel subsystem needs to (1) mark the page as being "mobile" and (2) set its `mapping` field to that of the anonymous inode: 

```
__SetPageMobile(page);
        page->mapping = anon_inode->mapping;
```

Once that is done, the kernel may consider that page for migration if it turns out to be in the way. The first step will be a call to `isolatepage()` to disconnect any internal mappings and ensure that the page can, indeed, be moved. The `mode` argument doesn't appear to be relevant for most code outside of the memory-management subsystem; the function should return `true` if the page can be migrated. Note that it's not necessary to cease use of the page at this point, but it is necessary to retain its ability to be moved. 

The actual migration may or may not happen, depending on whether other nearby pages turn out to be movable. If it does happen, the `migratepage()` callback will be invoked. It should do whatever work is needed to copy the page's contents, set the new page's flags properly, and update any internal pointers to the new page. It should also perform whatever locking is needed to avoid concurrent access to the pages while the migration is taking place. The return code should be `MIGRATEPAGE_SUCCESS` if the operation worked, or a negative error code otherwise. If the migration succeeds, the old page should not be touched again after `migratepage()` returns. 

The final step is a call to `putbackpage()`; its job is to replace the page in any internal lists and generally complete the migration process. If `isolatepage()` has been called on a given page, there will eventually be a `putbackpage()` call, regardless of whether the page is actually migrated in between the two calls. 

As can be seen, there is a fair amount of work required to support compaction in an arbitrary kernel subsystem. As a result, this support is likely to be confined to a relatively small number of subsystems that use substantial amounts of memory. Gioh's patch adapts the balloon driver subsystem in this way; on systems employing virtualization, balloon devices can (by their nature) use large amounts of memory, so making it movable makes some sense. Other possible use cases include long-lived I/O buffers or drivers (such as graphics drivers) that need to store large amounts of data. Fixing just a few of these drivers should go a long way toward making more large, physically contiguous regions of memory available even after the system has been up for some time.
