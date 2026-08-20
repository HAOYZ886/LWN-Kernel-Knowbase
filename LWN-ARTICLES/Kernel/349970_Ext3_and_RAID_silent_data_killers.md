---
title: "Ext3 and RAID: silent data killers?"
url: https://lwn.net/Articles/349970/
date: "August 31, 2009"
category: "Data integrity; Filesystems-ext3; RAID"
author: "By Jonathan Corbet August 31, 2009"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
August 31, 2009

Technologies such as filesystem journaling (as used with ext3) or RAID are generally adopted with the purpose of improving overall reliability. Some system administrators may thus be a little disconcerted by a recent linux-kernel thread suggesting that, in some situations, those technologies can actually increase the risk of data loss. This article attempts to straighten out the arguments and reach a conclusion about how worried system administrators should be. 

The conversation actually began last March, when Pavel Machek posted [a proposed documentation patch](<https://lwn.net/Articles/349977/>) describing the assumptions that he saw as underlying the design of Linux filesystems. Things went quiet for a while, before springing back to life at the end of August. It would appear that Pavel had run into some data-loss problems when using a flash drive with a flaky connection to the computer; subsequent tests done by deliberately removing active drives confirmed that it is easy to lose data that way. He [hadn't expected that](<https://lwn.net/Articles/349978/>): 

Before I pulled that flash card, I assumed that doing so is safe, because flashcard is presented as block device and ext3 should cope with sudden disk disconnects. And I was wrong wrong wrong. (Noone told me at the university. I guess I should want my money back). 

In an attempt to prevent a surge in refund requests at universities worldwide, Pavel tried to get some warnings put into the kernel documentation. He has run into a surprising amount of opposition, which he (and some others) have taken as an attempt to sweep shortcomings in Linux filesystems under the rug. The real story, naturally, is a bit more complex. 

Journaling technology like that used in ext3 works by writing some data to the filesystem twice. Whenever the filesystem must make a metadata change, it will first gather together all of the block-level changes required and write them to a special area of the disk (the journal). Once it is known that the full description of the changes has made it to the media, a "commit record" is written, indicating that the filesystem code is committed to the change. Once the commit record is also safely on the media, the filesystem can start writing the metadata changes to the filesystem itself. Should the operation be interrupted (by a power failure, say, or a system crash or abrupt removal of the media), the filesystem can recover the plan for the changes from the journal and start the process over again. The end result is to make metadata changes transactional; they either happen completely or not at all. And that should prevent corruption of the filesystem structure. 

One thing worth noting here is that actual data is not normally written to the journal, so a certain amount of recently-written data can be lost in an abrupt failure. It is possible to configure ext3 (and ext4) to write data to the journal as well, but, since the performance cost is significant, this option is not heavily used. So one should keep in mind that most filesystem journaling is there to protect metadata, not the data itself. Journaling does provide some data protection anyway - if the metadata is lost, the associated data can no longer be found - but that's not its primary reason for existing. 

It is not the lack of journaling for data which has created grief for Pavel and others, though. The nature of flash-based storage makes another "interesting" failure mode possible. Filesystems work with fixed-size blocks, normally 4096 bytes on Linux. Storage devices also use fixed-size blocks; on traditional rotating media, those blocks are traditionally 512 bytes in length, though [larger block sizes are on the horizon](<http://lwn.net/Articles/322777/>). The key point is that, on a normal rotating disk, the filesystem can write a block without disturbing any unrelated blocks on the drive. 

Flash storage also uses fixed-size blocks, but they tend to be large - typically tens to hundreds of kilobytes. Flash blocks can only be rewritten as a unit, so writing a 4096-byte "block" at the operating system level will require a larger read-modify-write cycle within the flash drive. It is certainly possible for a careful programmer to write flash-drive firmware which does this operation in a safe, transactional manner. It is also possible that the flash drive manufacturer was rather more interested in getting a cheap device to market quickly than careful programming. In the commodity PC hardware market, that possibility becomes something much closer to a certainty. 

What this all means is that, on a low-quality flash drive, an interrupted write operation could result in the corruption of blocks unrelated to that operation. If the interrupted write was for metadata, a journaling filesystem will redo the operation on the next mount, ensuring that the metadata ends up in its intended destination. But the filesystem cannot know about any unrelated blocks which might have been trashed at the same time. So journaling will not protect against this kind of failure - even if it causes the sort of metadata corruption that journaling is intended to prevent. 

This is the "bug" in ext3 that Pavel wished to document. He further asserted that journaling filesystems can actually make things worse in this situation. Since a full fsck is not normally required on journaling filesystems, even after an improper dismount, any "collateral" metadata damage will go undetected. At best, the user may remain unaware for some time that random data has been lost. At worst, corrupt metadata could cause the code to corrupt other parts of the filesystem over the course of subsequent operation. The skipped fsck may have enabled the system to come back up quickly, but it has done so at the risk of letting corruption persist and, possibly, spread. 

One could easily argue that the real problem here is the use of hidden translation layers to make a flash device look like a normal drive. David Woodhouse [did exactly that](<https://lwn.net/Articles/349985/>): 

This just goes to show why having this "translation layer" done in firmware on the device itself is a _bad_ idea. We're much better off when we have full access to the underlying flash and the OS can actually see what's going on. That way, we can actually debug, fix and recover from such problems. 

The manufacturers of flash drives have, thus far, proved impervious to this line of reasoning, though. 

There is a similar failure mode with RAID devices which was also discussed. Drives can be grouped into a RAID5 or RAID6 array, with the result that the array as a whole can survive the total failure of any drive within it. As long as only one drive fails at a time, users of RAID arrays can rest assured that the smoke coming out of their array is not taking their data with it. 

But what if more than one drive fails? RAID works by combining blocks into larger stripes and associating checksums with those stripes. Updating a block requires rewriting the stripe containing it and the associated checksum block. So, if writing a block can cause the array to lose the entire stripe, we could see data loss much like that which can happen with a flash drive. As a normal rule, this kind of loss will not occur with a RAID array. But it _can_ happen if (1) one drive has already failed, causing the array to run in "degraded" mode, and (2) a second failure occurs (Pavel pulls the power cord, say) while the write is happening. 

Pavel concluded from this scenario that RAID devices may actually be more dangerous than storing data on a single disk; he started a whole separate subthread (under the subject "[raid is dangerous but that's secret](<https://lwn.net/Articles/349987/>)") to that effect. This claim caused a fair amount of concern on the list; many felt that it would push users to forgo technologies like RAID in favor of single, non-redundant drive configurations. Users who do that will avoid the possibility of data loss resulting from a specific, unlikely double failure, but at the cost of rendering themselves entirely vulnerable to a much more likely single failure. The end result would be a lot more data lost. 

The real lessons from this discussion are fairly straightforward: 

  * Treat flash drives with care, do not expect them to be more reliable than they are, and do not remove them from the system until all writes are complete. 

  * RAID arrays can increase data reliability, but an array which is not running with its full complement of working, populated drives has lost the redundancy which provides that reliability. If the consequences of a second failure would be too severe, one should avoid writing to arrays running in degraded mode. 

  * As Ric Wheeler [pointed out](<https://lwn.net/Articles/349989/>), the easiest way to lose data on a Linux system is to run the disks with their write cache enabled. This is especially true on RAID5/6 systems, where write barriers are still not properly supported. There has been some talk of disabling drive write caches and enabling barriers by default, but no patches have been posted yet. 

  * There is no substitute for good backups. Your editor would add that any backups which have not been checked recently have a strong chance of not being good backups. 

How this information will be reflected in the kernel documentation remains to be seen. Some of it seems like the sort of system administration information which is not normally considered appropriate for inclusion in the documentation of the kernel itself. But there is value in knowing what assumptions one's filesystems are built on and what the possible failure modes are. A better understanding of how we can lose data can only help us to keep that from actually happening.
