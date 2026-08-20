---
title: "Journal-guided RAID resync"
url: https://lwn.net/Articles/363490/
date: "November 24, 2009"
category: "Block layer-RAID; RAID"
author: "By Jonathan Corbet November 24, 2009"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
November 24, 2009

The RAID4, 5, and 6 storage technologies are designed to protect against the failure of a single drive. Blocks of data are spread out across the array and, for each stripe, there is a parity block stored on one of the drives. Should one drive fail, the lost data can be recovered through the use of the remaining drives and the parity information. This mechanism copes less well with system crashes and power failures, though, forcing software RAID administrators to choose between speed and reliability. A new mechanism called journal-guided resyncronization may make life easier, but only if it actually gets into the kernel. 

The problem is that data and parity blocks must be updated in an atomic manner; if the two go out of sync, then the RAID array is no longer in a position to recover lost data. Indeed, it could return corrupted data. Expensive hardware RAID solutions use battery backup to ensure that updates are not interrupted partway through, but software RAID solutions often do not have that option. So if the system crashes - or the power fails - in the middle of an update to a RAID volume, that volume could end up being corrupted. Computer users, being a short-sighted kind of people in general, tend to regard this as a Bad Thing. 

There are a couple of possible ways of mitigating this risk. One is to perform a full rescan of the RAID volume after a crash, fixing up any partially-updated stripes. The problem here is that (1) the correct fix for an inconsistent stripe may not always be clear, and (2) this process can take a _long_ time. Long enough to cause users to think nostalgically about the days of fast, reliable floppy-disk storage. 

An alternative approach is to introduce a type of journaling to the RAID layer. The RAID implementation can set aside some storage where it writes stripes (perhaps not the data, but, perhaps, just the numbers of the affected stripes) prior to changing the real array. This approach works, and it can recover a crashed RAID array without a full rescan, but there is a cost here too: that journaling can slow down the operation of the array significantly. Writes to the journal must be synchronous or it cannot be counted on to do its job, so write operations become far slower than they were before. Given that, it's not surprising that a lot of RAID administrators turn off RAID-level journaling and spend a lot of time hoping that nothing goes wrong. 

A few years ago, Timothy E. Denehy, Andrea C. Arpaci-Dusseau, and Remzi H. Arpaci-Dusseau published [a paper](<http://www.usenix.org/events/fast05/tech/full_papers/denehy/denehy_html/main.html>) describing a better way, which they called "journal-guided resynchronization." Contemporary filesystems tend to do journaling of their own; why not use the filesystem journal to track changes to the RAID array as well? Running one journal can only be cheaper than running two - especially when one considers that the RAID journal must track, among other things, changes to the filesystem journal. The only problem is that the RAID and filesystem layers communicate through the relatively narrow block-layer API; using filesystem journaling to track RAID-level information has the potential to mix the layers considerably. 

Jody McIntyre's [journal-guided resync implementation](<http://lwn.net/Articles/363114/>) adds a new "declared" mode to the ext3 filesystem. As the journal is being written, a new "declare block" is added describing exactly which blocks are to be written to the storage device. Those blocks are then written with a new BIO flag stating that the filesystem has taken responsibility for resynchronizing the stripe should something go wrong; that lets the storage layer forget about that particular problem. Should the system crash, the filesystem will find those declare blocks in the journal; it can then issue a (new) `BIO_SYNCRAID` operation asking the storage subsystem to resynchronize the specific stripes containing the listed blocks. 

The result should be the best of both worlds. The cost of adding one more block to the filesystem journal is far less than doing that journaling at the RAID layer; Jody claims a 3-5% performance hit, as compared to 30% with the MD write-intent bitmap mechanism. But resynchronization after a crash should be quite fast, since it need only look at the parts of the array which were under active modification at the time. The only problem is that it requires the addition of specific support at the filesystem layer, so each filesystem must be modified separately. How this technique could be used in a filesystem which works without journaling (Btrfs comes to mind) would also have to be worked out. 

There's one other little problem as well. This work was done at Sun as a way of improving performance with the Lustre filesystem. But Jody notes: 

Unfortunately, we have determined that these patches are NOT useful to Lustre. Therefore I will not be doing any more work on them. I am sending them now in case they are useful as a starting point for someone else's work. 

So this patch series has been abandoned for now. It seems like this functionality should be useful to software RAID users, so, hopefully, somebody will pick them up and carry them forward. In the absence of a new developer, software RAID administrators will continue to face an unhappy choice well into the future.
