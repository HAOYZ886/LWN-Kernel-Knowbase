---
title: Securely deleting files from ext4 filesystems
url: https://lwn.net/Articles/462437/
date: "October 11, 2011"
category: "Filesystems-ext4; Security-Security technologies"
author: "By Jonathan Corbet October 11, 2011"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
October 11, 2011

Deleting a file is sufficient to make it go away as far as the directory structure is concerned; that important, not-backed-up document is gone before the careless user has even removed a repentant finger from the "enter" key. On the underlying storage device, though, parts or all of the doomed file can live on indefinitely. A suitably determined person can often recover data from a file that was thought to be deleted and gone. In some situations, this persistence of data can be most unwelcome. Paper shredders exist for situations where recovery of a "deleted" paper document is undesirable; there is clear value in a similar functionality for filesystem-based files. 

A look at the `chattr` man page indicates that "secure delete" functionality can be enabled for a file by setting the right attribute flag. The only problem is that most filesystems do not actually honor that flag; it's a sort of empty security theater. Even the TSA does a better job. The [ext4 secure delete patch set](<https://lwn.net/Articles/462398/>) from Allison Henderson may fill that particular gap for the ext4 filesystem, but a few things will need to be fixed up first. 

The core part of the patch is relatively straightforward: if part or all of a file is being deleted, the relevant blocks will be overwritten synchronously as part of the operation. By default, the blocks are overwritten with zeroes, but there is an option to use random bytes from the kernel's entropy pool instead. A bit of care must be taken when the file involved contains holes - it wouldn't do to overwrite blocks that don't exist. There is also some logic to take advantage of the "secure discard" feature supported by some devices (solid-state disks, primarily) if that feature is available. Secure discard handles the deletion internally to the device - perhaps just by marking the relevant blocks unreadable until something else overwrites them - eliminating the need to perform extra I/O from the kernel. 

The job does not stop there, though. The very existence of the file - or information contained in its name - could be sensitive as well. So the secure delete code must also clear out the directory entry associated with the file (if it is being deleted, as opposed to just truncated, obviously). Associated metadata - extended attributes, access control lists, etc - must also be cleaned out in this way. 

Then there is the little issue of the journal. At any given time, the journal could contain any number of blocks from the deleted file, possibly in several versions. Clearly, sanitizing the journal is also required, but it must be done carefully: clearing journal blocks before the associated transaction has been committed could result in a corrupted filesystem and/or the inability to recover properly from a crash. So the first thing that must happen is a synchronous journal flush. 

Once any outstanding transactions have been cleared, the (now old and unneeded) data in the journal can be cleaned up. The only problem is that the journal does not maintain any association between its internal blocks and the files they belong to in the filesystem. The patch addresses that problem by adding a new data structure mapping between journal blocks and file blocks; the secure deletion code then traverses that structure in search of blocks in need of overwriting. 

The journal cleanup code drew some complaints from developers; the first problem is that directory and metadata blocks are not cleared from the journal. The deeper complaint, though, was that it represented an excessive mixing of the two layers; the filesystem really should not have an overly deep understanding of how the journal works. The end result is that some of this cleanup work is likely to move into the jbd2 journaling layer; as an added benefit, other filesystems should then be able to take advantage of it as well. 

Darrick Wong also [pointed out](<https://lwn.net/Articles/462440/>) that this block mapping is not, itself, written to the journal; if the system were to crash before the journal could be cleaned, that cleanup would not happen after a reboot. He suggested that filesystems should mark blocks in need of secure deletion when they are first added to a journal transaction; the journal code could take care of things from then on. It seems like a cleaner solution, but there is, naturally, a catch. 

That catch is that there is no way to create a file with the "secure delete" attribute set; that attribute must be added after the fact. One would assume that users would ask for secure delete before actually writing any data to the file, but associated information (the file name, for example) must exist before the file can be so marked. So the journal may well contain blocks for "secure delete" files that will not be marked for overwriting. There is no way to fix that within the existing POSIX API. 

Darrick suggested a few ways to deal with this problem. One is to just give up and tell developers that, if even the name of the file is important, they should create that file under a temporary name, set the "secure delete" attribute, then rename the file to its real name. An alternative would be to mark the blocks for _all_ newly-created files as needing overwriting; that would eliminate the race to get the attribute set at the cost of slowing down all file creation operations. Or, he said, the journal could track the mapping between blocks and files, much as Allison's patch does. One other option would be to mark the entire filesystem as needing secure delete; that feature, evidently, is already in the works, but enabling it will have an obvious performance cost. 

The end result is that there are a few details to be worked out yet. The feature seems useful, though, for users who are concerned about data from their files hanging around after the files themselves have been deleted. Perhaps, around the 3.3 kernel or thereafter, "`chattr +s`" on ext4 will be more than bad security theater.
