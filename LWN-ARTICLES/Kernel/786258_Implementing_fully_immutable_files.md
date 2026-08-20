---
title: Implementing fully immutable files
url: https://lwn.net/Articles/786258/
date: "April 19, 2019"
category: Filesystems
author: "By Jonathan Corbet April 19, 2019"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
April 19, 2019

Like all Unix-like systems, Linux implements the traditional protection bits controlling who can access files in a filesystem (and what access they have). Fewer users, perhaps, are aware of a set of additional permission bits hidden away behind the [`chattr`](<http://man7.org/linux/man-pages/man1/chattr.1.html>) and [`lsattr`](<http://man7.org/linux/man-pages/man1/lsattr.1.html>) commands. Among other things, these bits can make a file append-only, mark a file to be excluded from backups, cause a file's data to be automatically overwritten on deletion, or make a file immutable. The implementation of many of these features is incomplete at best, so perhaps it's not surprising that immutable files can still be changed in certain limited circumstances. Darrick Wong has posted [a patch set](<https://lwn.net/ml/linux-fsdevel/155552786671.20411.6442426840435740050.stgit@magnolia/>) changing this behavior, implementing a user-visible behavioral change that he describes as ""an extraordinary way to destroy everything"". 

The `chattr` man page is clear on what happens when the immutable bit is set: 

A file with the 'i' attribute cannot be modified: it cannot be deleted or renamed, no link can be created to this file, most of the file's metadata can not be modified, and the file can not be opened in write mode. 

This description is true for the most part (at least on filesystems supporting this bit), but Wong noticed an important exception: a process that opens a file for writing prior to the setting of the immutable bit will still be able to write to the file after the bit is set for as long as it holds the file descriptor open. So while many operations on an immutable file are blocked, modifying the data in the file using open file descriptors is still allowed. The file is not yet, in other words, fully immutable. 

That behavior is both inconsistent and surprising; Wong set out to change it by making the system actually behave the way the man page says it will. Doing that requires two types of changes, the first of which is easier than the second. Whenever a process attempts to write to a file descriptor, a call is made to [`generic_write_checks()`](<https://elixir.bootlin.com/linux/v5.0/source/mm/filemap.c#L2931>) to ensure that the operation can be allowed. Adding a check for immutability to that function will cause `write()` calls to fail immediately once a file has been marked immutable. A similar check needs to be added to [`do_mmap()`](<https://elixir.bootlin.com/linux/v5.0/source/mm/mmap.c#L1380>) to prevent the creation of a writable memory mapping from a file descriptor. Those changes close off the most obvious ways to change an immutable file. 

The remaining problem is that writable memory mappings of the file may already exist, and those, too, can be used to modify a file that has since been marked immutable. User-space code need not make any system calls to write to a memory-mapped region, so there isn't a single, simple place to add a check like there is with `write()` and `mmap()`. The good news is that most of the needed machinery to prevent such writes is already in place, thanks to how filesystems manage writable mappings now. 

The problem that a filesystem implementation has to solve is that it, too, will get no notification when a process writes to a region of memory that has a file mapped into it; user space simply dereferences a pointer and stores data there. But the filesystem needs to know when that happens so it can ensure that the newly written data eventually finds its way to persistent storage. The trick that is used here is to write-protect the pages in memory. When user space attempts to write to one of those pages, a page fault will result; the kernel can then make the page writable and notify the filesystem that the page has been written to. When the filesystem code eventually writes the modified page(s) back to disk, it can once again write-protect those pages to mark them as being clean and to catch any subsequent modifications. 

One obvious place, then, to prevent modification of an immutable file is when that page fault occurs; rather than allow the modification to proceed, the kernel can fail the operation and deliver a `SIGBUS` signal to user space. But even that doesn't catch the case where pages have already been marked as being writable; user space could continue to make changes to those. 

Closing that last hole requires making changes to every filesystem that is to support the new behavior; that is the object of the bulk of the patches in Wong's set. Filesystem implementations already have an `ioctl()` handler that will be called when the immutable bit is set, so that is the logical place to respond when the status of a file is changed. A number of things need to happen at that point. If there are direct I/O operations outstanding, they must be allowed to conclude before marking the file immutable. Then, any pages that are currently dirty need to be flushed out to permanent storage; those changes were made prior to the file being marked immutable, so they should persist. Finally, all pages mapped from the file in question can be marked read-only, at which point they cannot be modified further. In most filesystems, flushing dirty pages and write-protecting them is already implemented as a single operation, so it's just a matter of calling the right function. 

The end result is that, once a file is marked immutable, it truly cannot be changed further — at least, until a privileged user clears the immutable bit. This is, of course, a change in the kernel's behavior; any application that relies on the ability to write to an open file descriptor for an immutable file will break. [Hyrum's law](<http://www.hyrumslaw.com/>) says that this is certain to happen somewhere; that would likely lead to the reversion of this patch set. In practice, though, it seems entirely possible that nobody actually depends on this obscure behavior, so Wong's patch set will fail to destroy everything as advertised.
