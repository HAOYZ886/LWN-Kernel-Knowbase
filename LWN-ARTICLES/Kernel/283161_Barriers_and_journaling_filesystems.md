---
title: Barriers and journaling filesystems
url: https://lwn.net/Articles/283161/
date: "May 21, 2008"
category: "Block layer; Filesystems-ext3; Write barriers"
author: "By Jonathan Corbet May 21, 2008"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
May 21, 2008

Journaling filesystems come with a big promise: they free system administrators from the need to worry about disk corruption resulting from system crashes. It is, in fact, not even necessary to run a filesystem integrity checker in such situations. The real world, of course, is a little messier than that. As a recent discussion shows, it may be even messier than many of us thought, with the integrity promises of journaling filesystems being traded off against performance. 

A filesystem like ext3 works by maintaining a journal on a dedicated portion of the disk. Whenever a set of filesystem metadata changes are to be made, they are first written to the journal - without changing the rest of the filesystem. Once all of those changes have been journaled, a "commit record" is added to the journal to indicate that everything else there is valid. Only after the journal transaction has been committed in this fashion can the kernel do the real metadata writes at its leisure; should the system crash in the middle, the information needed to safely finish the job can be found in the journal. There will be no filesystem corruption caused by a partial metadata update. 

There is a hitch, though: the filesystem code must, before writing the commit record, be absolutely sure that all of the transaction's information has made it to the journal. Just doing the writes in the proper order is insufficient; contemporary drives maintain large internal caches and will reorder operations for better performance. So the filesystem must explicitly instruct the disk to get all of the journal data onto the media before writing the commit record; if the commit record gets written first, the journal may be corrupted. The kernel's block I/O subsystem makes this capability available through the use of barriers; in essence, a barrier forbids the writing of any blocks after the barrier until all blocks written before the barrier are committed to the media. By using barriers, filesystems can make sure that their on-disk structures remain consistent at all times. 

There is another hitch: the ext3 and ext4 filesystems, by default, do not use barriers. The option is there, but, unless the administrator has explicitly requested the use of barriers, these filesystems operate without them - though some distributions (notably SUSE) change that default. Eric Sandeen recently decided that this was not the best situation, so he [submitted a patch](<http://lwn.net/Articles/282958/>) changing the default for ext3 and ext4. That's when the discussion started. 

Andrew Morton's [response](<https://lwn.net/Articles/283164/>) tells a lot about why this default is set the way it is: 

Last time this came up lots of workloads slowed down by 30% so I dropped the patches in horror. I just don't think we can quietly go and slow everyone's machines down by this much... 

There are no happy solutions here, and I'm inclined to let this dog remain asleep and continue to leave it up to distributors to decide what their default should be. 

So barriers are disabled by default because they have a serious impact on performance. And, beyond that, the fact is that people get away with running their filesystems without using barriers. Reports of ext3 filesystem corruption are few and far between. 

It turns out that the "getting away with it" factor is not just luck. Ted Ts'o [explains](<https://lwn.net/Articles/283168/>) what's going on: the journal on ext3/ext4 filesystems is normally contiguous on the physical media. The filesystem code tries to create it that way, and, since the journal is normally created at the same time as the filesystem itself, contiguous space is easy to come by. Keeping the journal together will be good for performance, but it also helps to prevent reordering. In normal usage, the commit record will land on the block just after the rest of the journal data, so there is no reason for the drive to reorder things. The commit record will naturally be written just after all of the other journal log data has made it to the media. 

That said, nobody is foolish enough to claim that things will always happen that way. Disk drives have a certain well-documented tendency to stop cooperating at inopportune times. Beyond that, the journal is essentially a circular buffer; when a transaction wraps off the end, the commit record may be on an earlier block than some of the journal data. And so on. So the potential for corruption is always there; in fact, Chris Mason [has a torture-test program](<https://lwn.net/Articles/283169/>) which can make it happen fairly reliably. There can be no doubt that running without barriers is less safe than using them. 

Anybody can turn on barriers if they are willing to take the performance hit. Unless, of course, their filesystem is based on an LVM volume (as certain distributions do by default); it turns out that the device mapper code does not pass through or honor barriers. But, for everybody else, it would be nice if that performance cost could be reduced somewhat. And it seems that might be possible. 

The current ext3 code - when barriers are enabled - performs a sequence of operations like this for each transaction: 

  1. The log blocks are written to the journal. 
  2. A barrier operation is performed. 
  3. The commit record is written. 
  4. Another barrier is executed. 
  5. Metadata writes begin at some later point. 

On ext4, the first barrier (step 2) can be omitted because the ext4 filesystem supports checksums on the journal. If the journal log data and the commit record are reordered, and if the operation is interrupted by a crash, the journal's checksum will not match the one stored in the commit record and the transaction will be discarded. Chris Mason [suggests](<https://lwn.net/Articles/283171/>) that it would be "mostly safe" to omit that barrier with ext3 as well, with a possible exception when the journal wraps around. 

Another idea for making things faster is to defer barrier operations when possible. If there is no pressing need to flush things out, a few transactions can be built up in the journal and all shoved out with a single barrier. There is also some potential for improvement by carefully ordering operations so that barriers (which are normally implemented as "flush all outstanding operations to media" requests) do not force the writing of blocks which do not have specific ordering requirements. 

In summary: it looks like the time has come to figure out how to make the cost of barriers palatable. Ted Ts'o [seems to feel that way](<https://lwn.net/Articles/283288/>): 

I think we have to enable barriers for ext3/4, and then work to improve the overhead in ext4/jbd2. It's probably true that the vast majority of systems don't run under conditions similar to what Chris used to demonstrate the problem, but the default has to be filesystem safety. 

Your editor's sense is that this particular dog is now wide awake and is likely to bark for some time. That may disturb some of the neighbors, but it's better than letting somebody get bitten later on.
