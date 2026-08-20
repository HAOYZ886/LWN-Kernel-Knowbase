---
title: "Soft updates, hard problems"
url: https://lwn.net/Articles/339337/
date: "July 1, 2009"
category: Filesystems; Soft updates
author: "July 1, 2009 This article was contributed by Valerie Aurora"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

July 1, 2009

This article was contributed by Valerie Aurora

If a file system discussion goes on long enough, someone will bring up soft updates eventually, usually in the form of, "Duhhh, why are you Linux people so stupid? Just use soft updates, like BSD!" Generally, there will be no direct reply to this comment and the conversation will flow silently around it, like a stream around an inky black boulder. Why is this? Is it pure NIH (Not Invented Here) on the part of Linux developers (and Solaris and OS X and AIX and...) or is there something deeper going on? Why are soft updates so famous and yet so seldom implemented? In this article, I will argue that soft updates are, simply put, too hard to understand, implement, and maintain to be part of the mainstream of file system development - while simultaneously attempting to explain how soft updates work. Oh, the irony! 

### Soft updates: The view from 50,000 feet

Soft updates is one of a family of techniques for maintaining on-disk file system consistency. The basic problem is that a file system doesn't always get shut down cleanly - think power outage or operating system crash - and if this happens in the middle of an update to the file system (say, deleting a file), the on-disk state of the file system may be inconsistent (corrupt). The original solution to this problem was to run `fsck` on the entire file system to find and correct inconsistencies; ext2 is an example of a file system that uses this approach. (Note that this use of `fsck` \- to recover from an unclean shutdown - is different from the use of `fsck` to check and repair a file system that has suffered corruption through some other cause.) 

The `fsck` approach has obvious drawbacks (excessive time, possible lost data), so file system developers have invented new techniques. The most popular and well-known is that of logging or journaling: before we begin writing out the changes to the file system, we write a short description of the changes we are about to make (a journal entry) to a separate area of the disk (the journal). If the system crashes in the middle of writing out the changes, we simply finish up the changes by replaying the journal entry at the next file system mount. 

Soft updates, instead, takes a two-step approach to crash recovery. First, we carefully order writes to disk so that, at the time of a crash (or any other time), the only inconsistencies are ones in which a file system structure is marked as allocated when it is actually unused. Second, at the next boot after the crash, `fsck` is run in the background on a file system snapshot (more on that later) to find and free file system structures that are wrongly marked as allocated. Basically, soft updates orders writes to the disk so that only relatively harmless inconsistencies are possible, and then fixes them in the background by checking and repairing the entire file system. The benchmark results are fairly stunning: in common workloads, performance is often within 5% of that of BSD's memory-only file system. The older version of FFS, which used synchronous writes and foreground `fsck` to provide similar reliability, often runs 20-30% slower than the in-memory file system. 

### Step 1: Update dependencies

The first part of implementing soft updates is figuring out how to order the writes to the disk so that after a crash, the only possible errors are inodes and blocks erroneously marked as allocated (when they are actually free). First, the authors lay out some rules to follow when writing changes to disk in order to accomplish this goal. From the paper: 

  1. Never point to a structure before it has been initialized (e.g., an inode must be initialized before a directory entry references it).

  2. Never re-use a resource before nullifying all previous pointers to it (e.g., an inode's pointer to a data block must be nullified before that disk block may be re-allocated for a new inode).

  3. Never reset the old pointer to a live resource before the new pointer has been set (e.g., when renaming a file, do not remove the old name for an inode until after the new name has been written).

Pairs of changes in which one change must to be written to disk before the next change can be written, according to the above rules, are called _update dependencies_. For some more examples of update dependencies, take the case of writing to the first block in a file for the first time. The first update dependency is that the block bitmap, which records which blocks are in-use, must be written to show that the block is in use before the block pointer in the inode is set. If a crash were to occur at this point, the only inconsistency would be one bit in the block bitmap showing a block is allocated when it isn't actually. This is a resource leak, and must be fixed eventually, but the file system can operate correctly with this error as long as it doesn't run out of blocks. 

The second update dependency is that the data in the block itself must be written before the block pointer in the inode can be set (along with the increase in the inode size and the associated timestamp updates). If it weren't, a crash at this point would result in garbage appearing in the file - a potential security hole, as well, if that garbage came from a previously written file. Instead, a crash would result in a leaked block (marked as allocated when it isn't) that happens to contain the data from the attempted write. As a result, the write to the bitmap and the write of the data to the block must complete (in any order) before the write that updates the inode's block pointer, size, and timestamps. 

These rules about ordering of writes aren't new for soft updates; they were originally created for writes to a "normal" FFS file system. In the original FFS code, ordering of writes is enforced with synchronous writes - that is, the ongoing file system operation (create, unlink, etc.) waits for each ordered write to hit disk before going on to the next step. While the write is in progress, the operating system buffer containing the disk block in question is locked. Any other operation needing to change that buffer has to wait its turn. As a result, many metadata operations progress at disk speed (i.e., murderously slowly). 

### Step 2: Efficiently satisfying update dependencies

So far, we have determined that synchronous writes on locked buffers are a slow, painful way of enforcing the ordering of writes to the file system. But synchronous writes are overkill for most file system operations; other than `fsync()`, we generally don't want a guarantee that the result has been written to stable storage before the system call returns, and as we've seen, the file system code itself usually only cares about the order of writes, not when they complete. What we want is a way to record changes to metadata, along with the associated ordering constraints, and then schedule the actual writes at our leisure. No problem, right? We'll just add a couple of pointers to each in-memory buffer containing metadata, linking it to the blocks it has come before and after. 

Turns out there is a problem: cyclical dependencies. We have to write to the disk in block-size units, and each block can potentially contain metadata affected by more than one metadata operation. If two different operations affect the same blocks, it can easily result in conflicting requirements: operation A requires that block 1 be written before block 2, and operation B requires that block 2 be written before block 1\. Now you can't write out any changes without violating the ordering constraints. What to do? 

Most people, at this point, decide to use journaling or copy-on-write to deal with this problem. Both techniques group related changes into transactions - a set of writes that must take effect all at once - and write them out to disk in such a manner that they take effect atomically. But if you are Greg Ganger and Yale Patt, you come up with a scheme to record individual modifications to blocks (such as the update to a single bit in a bitmap block) and their relationships to other individual changes (that change requires this other change to be written out first). Then, when you write out a block, you lock it and iterate through the records of individual changes to this block. For each individual change whose dependencies haven't yet been satisfied, you undo that change to the block, and then write out the resulting block. When the write is done, you re-apply the changes (roll forward), unlock, and continue on your way until the next write. The write you just completed may have satisfied the update dependencies of other blocks, so now you can go through the same process (lock, roll back, write, roll forward, unlock) for those blocks. Eventually, all the dependencies will be satisfied and everything will be written to disk, all without running into any circular dependencies. This, in a nutshell, is what makes soft updates unique. 

### Recording changes and dependencies

So what does a record of a metadata change, and its corresponding update dependencies, actually look like at the data structure level? First, there are twelve (as of the 1999 paper) distinct structures to record the different types of dependencies: 

> Structure| Dependency tracked  
> ---|---  
> bmsafemap| block/inode bitmaps  
> inodedep| inode  
> allocdirect| blocks referenced by direct block pointers  
> indirdep| indirect block  
> allocindir| blocks referenced by indirect block pointers  
> pagedep| directory entry add/remove  
> mkdir| new directory creation  
> dirrem| directory removal  
> freefrag| fragment to be freed  
> freeblks| block to be freed  
> freefile| inode to be freed  
  
Each kind of dependency-tracking structure includes pointers that allow it to be linked into lists attached to the buffers containing the relevant on-disk structures. These lists are what the soft updates code traverses during the roll-back and roll-forward operations on a block being written to disk. Each dependency structure has a set of state flags describing the current status of the dependency. The flags indicate whether the dependency is currently applied to the associated buffer, whether all of the writes it depends on have completed, and whether the update described by the dependency tracking structure itself has been written to disk. When all three of these flags are set (the update is applied to the in-memory buffer, all its dependent writes are completed, and the update is written to disk), the dependency structure can be thrown away. 

Page 7 of the [1999 soft updates paper [PDF]](<http://www.usenix.org/publications/library/proceedings/usenix99/full_papers/mckusick/mckusick.pdf>) begins the descriptions of specific kinds of update dependency structures and their relationships to each other. I've read this paper at least 15 times, and each time I when get to page 7, I'm feeling pretty good and thinking, "Yeah, okay, I must be smarter now than the last time I read this because I'm getting it this time," - and then I turn to page 8 and my head explodes. Here's the first figure on that page: 

> ![\[Figure 4\]](https://static.lwn.net/images/ns/kernel/mckusick-fig-4.png)

And that's only the figure from the left-hand column. An only slightly less complex spaghetti-gram occupies the right-hand column. This goes on for six pages, describing each specific kind of update dependency and its progression through various lists associated with buffers and file system structures and, most importantly, other update dependency structures. You find yourself struggling through paragraphs like: 

Figure 10 shows the structures involved in renaming a file. [Figure 10 looks much like the figure above.] The dependencies follow the same series of steps as those for adding a new file entry, with two variations. First, when a roll-back of an entry is needed because its inode has not yet been written to disk, the entry must be set back to the previous inode number rather than zero. The previous inode number is stored in a _dirrem_ structure. The DIRCHG flag is set in the _diradd_ structure so that the roll-back code knows to use the old inode number stored in the _dirrem_ structure. The second variation is that, after the modified directory entry is written to disk, the _dirrem_ structure is added to the work daemon's _tasklist_ list so that the link count of the old inode will be decremented as described in Section 3.9. 

Say that three times fast! 

The point is not that the details of soft updates are too complex for mere humans to understand (although I personally wouldn't bet against Greg Ganger being superhuman). The point is that this complexity reflects a lack of generality and abstraction in the design of soft updates as a whole. In soft updates, every file system operation must be individually analyzed for write dependencies, every on-disk structure must have a custom-designed dependency tracking structure, and every update operation must allocate one of these structures and hook itself into the web of other custom-designed dependency tracking structures. If you add a new file system feature, like extended attributes, or change the on-disk format, you have to start from scratch and reason out the relevant dependencies, design a new structure, and write the roll-forward/roll-back routines. It's fiddly, tedious work, and the difficulty of doing it correctly doesn't make it any better a use of programmer staff-hours. 

Contrast the highly operation-specific design of soft updates to the transaction-style interface used by most journaling and copy-on-write file systems. When you begin a logical operation (such as a file create), you create a transaction handle. Then, for each on-disk structure you have to modify, you add that buffer to the list of buffers modified by this transaction. When you are done, you close out the transaction and hand it off to the journaling (or COW) subsystem, which figures out how to merge it with other transactions and write them out to disk properly. The user of the transaction interface only has to know how to open, close, and add blocks to a transaction, while the transaction code only has to know which blocks are part of the same transaction. Adding a new write operation requires no special knowledge or analysis beyond remembering to add changed blocks to the transaction. 

The lack of generalization and abstraction shows up again when the combination of update dependency ordering and the underlying disk format poke out and cause strange user-visible behavior. The most prominent example shows up when removing a directory; following the rules governing update dependencies means that a directory's ".." entry can't be removed until the directory itself is recorded as unlinked on the disk. Chains of update dependencies sometimes resulted in up to two minutes of delay between the return of `rmdir()`, and the corresponding decrement of the parent directory's link count. This can break, among other things, a simple recursive "`rm -rf`". The fix was to fake up a second link count that is reported to userspace, but the real problem was a too-tight coupling between on-disk structures, the system to maintain on-disk consistency, and the user-visible structures. Long chains of update dependencies cause problems elsewhere, during unmount and `fsync()` in particular. 

### Fsck and the snapshot

But wait, there's more! The second stage of recovery for soft updates is to run `fsck` after the next boot, in the background using a snapshot of the file system metadata. File system snapshots in FFS are implemented by creating a sparse file the same size as the file system - the snapshot file. Whenever a block of metadata is altered, the original data is first copied to the corresponding block in the snapshot file. Reads of unaltered blocks in the snapshot redirect to the originals. Online `fsck` runs on the snapshot of the file system metadata, finding leaked blocks and inodes. Once it completes, `fsck` uses a special system call to mark these blocks and inodes as free again. 

Online fsck implemented in this manner has severe limitations. First, recovery from a crash still requires reading and processing the metadata for the entire file system - in the background, certainly, but that's still a lot of I/O. ([Freeblock scheduling](<http://www.pdl.cmu.edu/Freeblock/index.html>) piggybacks low-priority I/O, like that of a background `fsck`, on high-priority foreground I/O so that it interferes as little as possible with "real" work, but that's cold comfort.) Second, it's not actually a full file system check and repair, it's just a scan for leaked blocks and inodes - expected inconsistencies. The whole concept of running `fsck` on a snapshot file whose blocks are allocated from the same file system assumes that the file system is not corrupted in a way that leaves blocks marked as free when they are actually allocated. 

### Conclusion

Conceptually, soft updates can be explained concisely - order writes according to some simple rules, track updates to metadata blocks, roll-back updates with unsatisfied dependencies before writing the block to disk, then roll-forward the updates again. But when it come to implementation, only programmers with deep, encyclopedic knowledge of the file system's on-disk format can derive the correct ordering rules and construct the associated data structures and web of dependencies. The close coupling of on-disk data structures and their updates and the user-visible data structures and their updates results in weird, counter-intuitive behavior that must be covered up with additional code. 

Overall, soft updates is a sophisticated, insightful, clever idea - and an evolutionary dead end. Journaling and copy-on-write systems are easier to implement, require less special-purpose code, and demand far less of the programmer writing to the interface.
