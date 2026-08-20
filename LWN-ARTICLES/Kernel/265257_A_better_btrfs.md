---
title: A better btrfs
url: https://lwn.net/Articles/265257/
date: "January 15, 2008"
category: "Btrfs; Filesystems-Btrfs"
author: "By Jonathan Corbet January 15, 2008"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
January 15, 2008

Chris Mason has recently released [Btrfs v0.10](<http://lwn.net/Articles/265199/>), which contains a number of interesting new features. In general, Btrfs has come a long way since LWN first [wrote about it](<http://lwn.net/Articles/238923/>) last June. Btrfs may, in some years, be the filesystem most of us are using - at least, for those of us who will still be using rotating storage then. So it bears watching. 

Btrfs, remember, is an entire new filesystem being developed by Chris Mason. It is a copy-on-write system which is capable of quickly creating snapshots of the state of the filesystem at any time. The snapshotting is so fast, in fact, that it is used as the Btrfs transactional mechanism, eliminating the need for a separate journal. It supports subvolumes - essentially the existence of multiple, independent filesystems on the same device. Btrfs is designed for speed, and also provides checksumming for all stored data. 

Some kernel patches show up and quickly find their way into production use. For example, one year ago, nobody (outside of the -ck list, perhaps) was talking about fair scheduling; but, as of this writing, the CFS scheduler has been shipping for a few months. KVM also went from initial posting to merged over the course of about two kernel release cycles. Filesystems do not work that way, though. Filesystem developers tend to be a cautious, conservative bunch; those who aren't that way tend not to survive their first few encounters with users who have lost data. This is all a way of saying that, even though Btrfs is advancing quickly, one should not plan on using it in any sort of production role for a while yet. As if to drive that point home, Btrfs still crashes the system when the filesystem runs out of space. The v0.10 patch, like its predecessors, also changes the on-disk format. 

The on-disk format change is one of the key features in this version of the Btrfs patch. The format now includes back references on almost all objects in the filesystem. As a result, it is now easy to answer questions like "to which file does this block belong?" Back references have a few uses, not the least of which is the addition of some redundant information which can be used to check the integrity of the filesystem. If a file claims to own a set of blocks which, in turn, claim to belong to a different file, then something is clearly wrong. Back references can also be used to quickly determine which files are affected when disk blocks turn bad. 

Most users, however, will be more interested in another new feature which has been enabled by the existence of back references: online resizing. It is now possible to change the size of a Btrfs filesystem while it is mounted and busy - this includes shrinking the filesystem. If the Btrfs code has to give up some space, it can now quickly find the affected files and move the necessary blocks out of the way. So Btrfs should work nicely with the device mapper code, growing or shrinking filesystems as conditions require. 

Another interesting feature in v0.10 is the associated in-place ext3 converter. It is now possible to non-destructively convert an existing ext3 filesystem to Btrfs - and to go back if need be. The converter works by stashing a copy of the ext3 metadata found at the beginning of the disk, then creating a parallel directory tree in the free space on the filesystem. So the entire ext3 filesystem remains on the disk, taking up some space but preserving a fallback should Btrfs not work out. The actual file data is shared between the two filesystems; since Btrfs does copy-on-write, the original ext3 filesystem remains even after the Btrfs filesystem has been changed. Switching to Btrfs forevermore is a simple matter of deleting the ext3 subvolume, recovering the extra disk space in the process. 

Finally, the copy-on-write mechanism can be turned off now with a mount option. For certain types of workloads, copy-on-write just slows things down without providing any real advantages. Since (1) one of those workloads is relational database management, and (2) Chris works for Oracle, the only surprise here is that this option took as long as it did to arrive. If multiple snapshots reference a given file, though, copy-on-write is still performed; otherwise it would not be possible to keep the snapshots independent of each other. 

For those who are curious about where Btrfs will go from here, Chris has posted [a timeline](<http://oss.oracle.com/projects/btrfs/dist/documentation/todo.html>) describing what he plans to accomplish over the coming year. Next on the list would appear to be "storage pools," allowing a Btrfs filesystem to span multiple devices. Once that's in place, striping and mirroring will be implemented within the filesystem. Longer-term projects include per-directory snapshots, fine-grained locking (the filesystem currently uses a single, global lock), built-in incremental backup support, and online filesystem checking. Fixing that pesky out-of-space problem isn't on the list, but one assumes Chris has it in the back of his mind somewhere.
