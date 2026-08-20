---
title: "Featherstitch: Killing fsync() softly"
url: https://lwn.net/Articles/354861/
date: "September 30, 2009"
category: Filesystems
author: "September 30, 2009 This article was contributed by Valerie Aurora"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

September 30, 2009

This article was contributed by Valerie Aurora

[Soft updates](<http://lwn.net/Articles/339337/>), a method of maintaining on-disk file system consistency through carefully ordering writes to disk, have only been implemented once in a production operating system (FreeBSD). You can argue about exactly why they have not been implemented elsewhere, and in Linux in particular, but my theory is that not enough file systems geniuses exist in the world to write and maintain more than one instance of soft updates. [Chris Frost](<http://www.frostnet.net/chris/>), a graduate student at UCLA, agrees with the too-complicated-for-mere-mortals theory. That's why in 2006, he and several co-conspirators at UCLA wrote the [Featherstitch](<http://featherstitch.cs.ucla.edu/>) system for keeping on-disk data consistent. 

Featherstitch is a generalization of the soft updates system of write dependencies and rollback data. The resulting system is general enough that most (possibly all) other file system consistency strategies (e.g., journaling) can be efficiently implemented on top of the Featherstitch interface. What makes Featherstitch unique among file systems consistency techniques is that it exports a safe, efficient, non-blocking mechanism to userland applications that lets them group and order writes without using `fsync()` or relying on file system-specific behavior (like ext3 `data=ordered` mode). 

#### Featherstitch basics: patches, dependencies, and undo data

What is Featherstitch, other than something file system aficionados throw in your face whenever you complain about soft updates being too complicated? Featherstitch grew out of soft updates and has a lot in common with that approach architecturally. The main difference between Featherstitch and soft updates is that the latter implements each file system operation individually with a different specialized set of data structures specific to the FFS file system, while Featherstitch generalizes the concept of a set of updates to different blocks and creates one data structure and write-handling mechanism shared by all file system operations. As a result, Featherstitch is easier to understand and implement than soft updates. 

Featherstitch records all changes to the file system in "patches" (the dearth of original terminology in software development strikes again). A patch includes the block number, a linked list of patches that this patch depends on, and the "undo data." The undo data is a byte-level diff of the changes made to this block by this patch, including the offset, length, and contents of the range of bytes overwritten by this change. Another version of a patch is optimized for bit-flip changes, like those made to block bitmaps. The rule for writing patches out to storage is simple: If any of the patches this patch depends on - its dependencies - aren't confirmed to be written to disk, this patch can't be written yet. 

In other words, patches and dependencies look a lot like a generic directed acyclic graph (DAG), with patches as the circles and dependencies as the arrows. If you are a programmer, you've probably drawn hundreds or thousands of these pictures in your life. Just imagine a little diff hanging off each circle and you've got a good mental model for thinking about Featherstitch. The interesting bits are around reducing the number of little circles - in the first implementation, the memory used by Featherstitch undo data was often twice that of the actual changes written to disk. For example, untarring a 220MB kernel source tree allocated about 460MB of undo data. 

The acyclic-ness of Featherstitch patch dependencies deserves a little more attention. It is the caller's responsibility to avoid creating circular patch dependencies in the first place; Featherstitch doesn't detect or attempt to fix them. (The simplified interface exported to userspace makes cycles impossible to create in the first place, more about that later.) However, lack of circular dependencies among patches does not imply a lack of circular dependencies between blocks. Patches are a record of a change to a block and each block can have multiple outstanding patches against it. Imagine a patch dependency, patch A depends on patch B, which depends on patch C. That is, `A->B->C`, where "`->`" reads as "depends on." If patch A applies to block 1, and patch B applies to block 2, and patch C applies to block 1, then viewing the blocks and their outstanding patches as a whole, you have a circular dependency where block 1 must be written before block 2, but block 2 must also be written before block 1. This is called a "block-level cycle" and it causes most of the headaches in a system based on write ordering. 

The way both soft updates and Featherstitch resolve block level cycles is by keeping enough information about each change to roll it back. When it comes time to write a block, any applied patches which can't be written yet (because their dependencies haven't been written yet) are rolled back using their undo data. In our example, with `A->B->C` and A and C both applied to block 1, we would roll back A on block 1, write block 1 with patch C applied, write B's block, and then write block 1 a second time with both patch A and C applied. 

#### Optimization

The first version of Featherstitch was elegant, general purpose, easy to understand, and extraordinarily inefficient. On several benchmarks, the original implementation allocated over twice as much memory for patches and undo data as needed for the actual new data itself. The system became CPU-bound with as few as 128 blocks in the buffer cache. 

The first goal was to reduce the number of patches needed to complete an operation. In many cases, a patch will never be reverted - for example, if we write to a file's data block when no other writes are outstanding on the file system, then there is no reason we'd ever have to roll back to the old version of the block. In this case, Featherstitch creates a "hard patch" - a patch that doesn't keep any undo data. The next optimization is to merge patches when they can always be written together without violating any dependencies. A third optimization merges overlapping patches in some cases. All of these patch reduction techniques hinge on the Featherstitch rules for creating patches and dependencies, in particular that a patch's dependencies must be specified at creation time. Some opportunities for merging can be detected at patch creation time, others when a patch commits and is removed from the queue. 

The second major goal was to efficiently find patches ready for writing. A normal buffer cache holds several hundred thousand blocks, so any per-block data structures and algorithms must be extremely efficient. Normally, the buffer cache just has to, in essence, walk a list of dirty blocks and issue writes on them in some reasonably optimal manner. With Featherstitch, it can find a dirty block, but then it has to walk its list of patches checking to see if there is a subset whose dependencies have been satisfied and are therefore ready for writing. This list can be long, and it can turn out that none of the patches are ready, in which case it has to give up and go on to the next patch. Rather than randomly searching in the cache, Featherstitch instead keeps a list of patches that are ready to be written. When a patch has committed, the list of patches that depended on it is traversed and newly ready patches added to the list. 

With these optimizations, the memory overhead of Featherstitch dropped from 200+% to 4-18% in the set of benchmarks used for evaluation - still high, but in the realm of practicality. The optimizations described above were only partially implemented in some cases, leaving more room for improvement without any further insight. 

#### Performance

For head-to-head performance comparisons, the authors implemented several versions of file system consistency using the Featherstitch patch interface and compared them to the ext2 and ext3 equivalents. Using ext2 as the on-disk file system format, they re-implemented soft updates, metadata-only journaling, and full data/metadata journaling. Metadata-only journaling corresponds to ext3's `data=writeback` mode (file data is written without regard to the state of file system metadata that refers to it) and full journaling corresponds to ext3's `data=full` mode (all file data is written directly to the journal along with file system metadata). 

The benchmarks used were extraction of a ~200MB tar file (the kernel source code, natch), deletion of the results of previous, a Postmark run, and a modified Andrew file system benchmark - in other words, the usually motley crew of terrible, incomplete, unrepresentative file system benchmarks we always run because there's nothing better available. The deficiency shows: under this workload, ext3 performed about the same in `data=writeback` and `data=ordered` mode (not usually the case in real-world systems), which is one of the reasons the authors didn't implement ordered mode for Featherstitch. The overall performance result was that the Featherstitch implementations were at par or somewhat better with the comparable ext3 version for elapsed time, but used significantly more CPU time. 

#### Patchgroups: Featherstitch for userspace

So, you can use Featherstitch to re-implement all kinds of file system consistency schemes - soft updates, copy-on-write, journaling of all flavors - and it will go about as fast the old version while using up more of your CPU. When you have big new features like checksums and snapshots in btrfs, it's hard to get excited about an under-the-covers re-implementation of file system internals. It's cool, but no one but file systems developers will care, right? 

In my opinion, the most exciting application of Featherstitch is not in the kernel, but userland. In short, Featherstitch exports an interface that applications can use to get the on-disk consistency results they want, AND keep most of the performance benefits that come with re-ordering and delaying writes. Right now, applications have only two practical choices for controlling the order of changes to the file system: Wait for all writes to a file to complete using fsync(), or rely on file system-specific implementation details, like ext3 `data=ordered` mode. Featherstitch gives you a third option: Describe the exact, minimal ordering relationship between various file system writes and then let the kernel re-order, delay, and otherwise optimize the writes as much possible within those constraints. 

The userland interface is called "patchgroups." The interface prevents the two major pitfalls that usually accompany exporting a kernel-level consistency mechanism to userspace. First, it prevents deadlocks caused by dependency cycles ("Hey, kernel! Write A depends on write B! And, oh yeah, write B depends on write A! Have a nice day!"). In the kernel, you can define misuse of the interface as a kernel bug, but if an application screws up a dependency, the whole kernel grinds to a halt. Second, it prevents applications from stalling their own or other writes by opening a transaction and holding it open indefinitely while it adds new changes to the transaction (or goes off into an infinite loop, or crashes, or otherwise fails to wrap up its changes in a neat little bow). 

The patchgroups interface simply says that all of those writes over there must be on-disk before any of these writes over here can start being written to disk. Any other writes that happen to be going on outside of these two sets can go to disk in any order they please, and the writes inside each set are not ordered with respect to each other either. Here's a pseudo-code example of using patchgroups: 

```
/* Atomic update of a file using patchgroups */
    
        /* Create a patch group to track the creation of the new copy of the file */
        copy_pg = pg_create();
    
        /* Tell it to track all our file systems changes until pg_disengage() */
        pg_engage(copy_pg);
    
        /* Open the source file, get a temporary filename, etc. */
    
        /* Create the temp file */
        temp_fd = creat();
    
        /* Copy the original file data to the temp file and make your changes */
    
        /* All changes done, now wrap up this patchgroup */
        pg_disengage(copy_pg);
```

The temp file now contains the new version of the file, and all of the related file system changes are part of the current patchgroup. Now we want to put the following `rename()` in a separate patchgroup that depends on the patchgroup containing the new version of the file. 

```
/* Start a new patchgroup for the rename() */
        rename_pg = pg_create();
        
        pg_engage(rename_pg);
    
        /*
         * MAGIC OCCURS HERE: This is where we tell the system that the
         * rename() can't hit disk until the temporary file's changes have
         * committed.  If you don't have patchgroups, this is where you would
         * fsync() instead.  fsync() can also be thought of as:
         *
         * pg_depend(all previous writes to this file, this_pg);
         * pg_sync(this_pg);
         */
    
        /* This new patchgroup, rename_pg, depends on the copy_pg patchgroup */
        pg_depend(copy_pg, rename_pg);
    
        /* This rename() becomes part of the rename_pg patchgroup */
        rename();
    
        /* All set! */
        pg_disengage(rename_pg);
    
        /* Cleanup. */
        pg_close(copy_pg);
        pg_close(rename_pg);
```

Short version: No more "[Firefox `fsync()`](<http://lwn.net/Articles/328363/>)" bug, O_PONIES for everyone who wants them and very little cost for those who don't. 

#### Conclusion

Featherstitch is a generalization and simplification of soft updates, with reasonable, but not stellar, performance and overhead. Featherstitch really shines when it comes to exporting a useful, safe write ordering interface for userspace applications. It replaces the enormous performance-destroying hammer of `fsync()` with a minimal and elegant write grouping and ordering mechanism. 

When it comes to the Featherstitch paper itself, I highly recommend reading the entire paper simply for the brief yet accurate summaries of complex storage-related issues. Sometimes I feel like I'm reading the distillation of three hours of the [Linux Storage and File Systems Workshop](<http://lwn.net/Articles/327601/>) plus another couple of weeks of mailing list discussion, all in one paragraph. For example, section 7 describes, in a most extraordinarily succinct manner, the options for correctly flushing a disk's write cache, including specific commands, both SCSI and ATA, and a brief summary of the quality of hardware support for these commands.
