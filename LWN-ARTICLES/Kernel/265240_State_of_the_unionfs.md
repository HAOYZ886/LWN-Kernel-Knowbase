---
title: State of the unionfs
url: https://lwn.net/Articles/265240/
date: "January 15, 2008"
category: "Filesystems-Union; Unionfs"
author: "By Jonathan Corbet January 15, 2008"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
January 15, 2008

LWN last [looked at the unionfs filesystem](<http://lwn.net/Articles/217084/>) almost exactly one year ago. Things have been relatively quiet on the unionfs front during much of that time, but unionfs has not gone away. Now the unionfs developers are back with an improved version and a determined push to get the code into 2.6.25. So another look seems indicated. 

The core idea behind unionfs is to allow multiple, independent filesystems to be merged into a single, coherent whole. As an example, consider a user with a distribution install DVD full of packages, a small disk, and painfully slow bandwidth. It would be nice to keep the DVD-stored packages around for future installation. What is also nice, though, is to be able to keep a directory full of updates from the distributor and use those, when they exist, in favor of the read-only DVD version. Using unionfs, this user could mount the DVD read-only, then mount a writable filesystem (for the updates) on top of the DVD. Updated packages go into the writable filesystem, but all of the available packages are visible, together, in the unified view. To avoid confusion, the user could delete obsoleted packages, at which point they would no longer be visible in the unionfs filesystem, even though they cannot actually be deleted from the underlying DVD. Thus unionfs allows the creation of an apparently writable filesystem on a read-only base; many other applications are possible as well. 

If a user rewrites a file which is stored on a read-only "branch" of a union filesystem, the response is relatively straightforward: the newly-written file is stored on a higher-priority, writable branch. If no such branch exists, the operation fails. Dealing with the deletion of a file from a read-only branch is trickier, though. In this case, unionfs will create a "whiteout" in the form of a special file (starting with `.wh.`) on a writable branch. Some reviewers have disliked this approach since it will clutter the upper branch with those special files over time. But it is hard to come up with another way to handle deletion, especially if (as is the case here) your goal is to keep core VFS changes to an absolute minimum. 

That hasn't kept the unionfs developers from trying, though. Off to the side, they have a version of unionfs which maintains a small, special-purpose partition of its own (on writable storage). Metadata (whiteouts, in particular) is stored to this special unionfs partition and no longer clutters the component filesystems. There are other advantages to the dedicated partition scheme, including the ability to include one unionfs as a branch in a second union; see [the unionfs ODF document](<http://www.filesystems.org/unionfs-odf.txt>) for more information on this approach, which the developers hope to slowly migrate into the version they are currently proposing for the mainline. 

Another persistent problem with unionfs has been coping with modifications made directly to the component branches without going through the union. The January, 2007 version of the patch came packaged with some dire warnings: direct modification of unionfs branches could lead to system crashes and data loss. Given that filesystems which have been bundled into a union still exist independently, they will always present a tempting target for modification, even when there is not a specific reason (wanting to put files onto a specific component filesystem, for example). So a unionfs implementation which cannot handle such modifications sets a trap for every user who uses it. 

The developers claim to have solved this problem in [the current version](<http://lwn.net/Articles/265037/>) of the patch. Now, almost every entry into the unionfs code causes it to check the modification times for the relevant file in all layers of the union. If the file turns out to have been changed, unionfs will forget about the file and reload the information from scratch, causing the most current version of the file (or directory) to be visible to the user. This approach solves the problem in a relatively efficient manner, with one exception: unionfs cannot tell when a process modifies a file which it has mapped into its address space with `mmap()`. So, in that case, changes may not be visible to processes accessing the affected file through the unionfs. 

In both cases, the unionfs developers would really prefer to have better support from the VFS. Some operating systems have provided native support for whiteouts, but Linux lacks that support. There is also no way for a filesystem at the bottom of a stack of filesystems to notify the higher layers that something has been changed. Fixing either of these would require significant VFS modifications, though, and the changes might propagate down into the individual filesystem implementations as well. So nobody is expecting them to happen anytime soon. 

Another significant change in unionfs is the elimination of the `ioctl()` interface for the management of branches. All changes to an existing unionfs are now done using the `remount` option of the `mount` command. This change eliminates the need for a separate utility for unionfs configuration and makes it possible to do complicated changes in an atomic manner. 

The end result of all this is that the unionfs hackers think that the time has come to put the code into the mainline. There, it would become the second supported stacking filesystem (the first being eCryptfs), and would help toward the long-term goal of making the VFS layer work better with stacking. Some people speak as if the merging of unionfs into 2.6.25 is a done deal, but that is not yet guaranteed. Christoph Hellwig, whose opinion on such things carries a heavy weight, [is opposed](<https://lwn.net/Articles/265245/>) to the unionfs idea: 

I think we made it pretty clear that unionfs is not the way to go, and that we'll get the union mount patches clear once the per-mountpoint r/o and unprivileged mount patches series are in and stable. 

Unionfs hacker Erez Zadok [responds](<https://lwn.net/Articles/265248/>) that unionfs is working - and used - now, while getting union support into the VFS is a distant prospect. So he recommends: 

I think a better approach would be to start with Unionfs (a standalone file system that doesn't touch the rest of the kernel). And as Linux gradually starts supporting more and more features that help unioning/stacking in general, to change Unionfs to use those features (e.g., native whiteout support). Eventually there could be basic unioning support at the VFS level, and concurrently a file-system which offers the extra features (e.g., persistency). 

When one looks at a recent posting of [the union mount patch](<http://lwn.net/Articles/260989/>), it's hard to see them as a near-term solution. As described by its author (Bharata Rao), this work is in an early, exploratory state; there are a number of problems for which solutions are not really in sight. The union mount approach, which does the hard work in the VFS layer, may well be the right long-term approach, but it will not be in a state where it can be shipped to users anytime soon. 

In the end, the problem is a hard one, and unionfs has a considerable lead toward being a real solution. That, alone, is not enough to guarantee that unionfs will make it into the 2.6.25 kernel, but it does help that cause considerably. Anybody opposing the merger of unionfs will have to explain why the union filesystem capability should not be available to Linux users in 2008.
