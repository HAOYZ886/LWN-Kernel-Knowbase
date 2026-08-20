---
title: Responding to ext4 journal corruption
url: https://lwn.net/Articles/284037/
date: "May 27, 2008"
category: "Filesystems-ext4"
author: "By Jonathan Corbet May 27, 2008"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
May 27, 2008

[Last week's article](<http://lwn.net/Articles/283161/>) on barriers described one way in which things could go wrong with journaling filesystems. Therein, it was noted that the journal checksum feature added to the ext4 filesystem would mitigate some of those problems by preventing the replay of the journal if it had not been completely written before a crash. As a discussion this week shows, though, the situation is not quite that simple. 

Ted Ts'o was doing some ext4 testing when he [noticed a problem](<https://lwn.net/Articles/284038/>) with how the journal checksum is handled. The journal will normally contain several transactions which have not yet been fully played into the filesystem. Each one of those transactions includes a commit record which contains, among other things, a checksum for the transaction. If the checksum matches the actual transaction data in the journal, the system knows that the transaction was written completely and without errors; it should thus be safe to replay the transaction into the filesystem. 

The problem that Ted noticed was this: if a transaction in the middle of the series failed to match its checksum, the playback of the journal would stop - but only after writing the corrupted transaction into the filesystem. This is a sort of worst-of-all-worlds scenario: the kernel will dump data which is known to be corrupt into the filesystem, then silently throw away the (presumably good) transactions after the bad one. The ext4 developers quickly arrived at a consensus that this behavior is a bug which should be fixed. 

But what should really done is not as clear as one might think. Ted's suggestion was this: 

So I think the right thing to do is to replay the *entire* journal, including the commits with the failed checksums (except in the case where journal_async_commit is enabled and the last commit has a bad checksum, in which case we skip the last transaction). By replaying the entire journal, we don't lose any of the revoke blocks, which is critical in making sure we don't overwrite any data blocks, and replaying subsequent metadata blocks will probably leave us in a much better position for e2fsck to be able to recover the filesystem. 

A bit of background might help in understanding the problem that Ted is trying to solve here. In the default `data=ordered` mode, ext3 and ext4 do not write all data to the journal before it goes to the filesystem itself. Instead, only filesystem metadata goes to the journal; data blocks are written directly to the filesystem. The "ordered" part means that all of the data blocks will be written before the filesystem code will start writing the metadata; in this way, the metadata will always describe a complete and correct filesystem. 

Now imagine a journal which contains a set of transactions similar to these (in this order): 

  1. A file is created, with its associated metadata. 

  2. That file is then deleted, and its metadata blocks are released. 

  3. Some other file is extended, with the newly-freed metadata blocks being reused as data blocks. 

Imagine further that the system crashes with those transactions in the journal, but transaction 2 is corrupt. Simply skipping the bad transaction and replaying transaction 3 would lead to the filesystem being most confused about the status of the reused blocks. But just stopping at the corrupt transaction also has a problem: the data blocks created in transaction 3 may have already been written, but, as of transaction 1, the filesystem thinks those are metadata blocks. That, too, leads to a corrupt filesystem. By replaying the entire journal, Ted hopes to catch situations like that and leave the filesystem in an overall better shape. 

It is, perhaps, not surprising that there was some disagreement with this approach. Andreas Dilger [argued](<https://lwn.net/Articles/284045/>): 

The whole point of this patch was to avoid the case where random garbage had been written into the journal and then splattering it all over the filesystem. Considering that the journal has the highest density of important metadata in the filesystem, it is virtually impossible to get more serious corruption than in the journal. 

The [next proposal](<https://lwn.net/Articles/284047/>) was to make a change to the on-disk journal format ("one more time") turning the per-transaction checksum into a per-block checksum. Then it would be possible to get a handle on just how bad any corruption is, and even corrupt transactions could be mostly replayed. As of this writing, that looks like the approach which will be taken. 

Arguably, the real conclusion to take from this discussion was best [expressed](<http://lwn.net/Articles/283781/>) by Arjan van de Ven in an entirely different context: ""having a journal is soooo 1999"". The Btrfs filesystem, which has a good chance of replacing ext3 and ext4 a few years from now, does not have a journal; instead, it uses its fast snapshot mechanism to keep transactions consistent. Btrfs may, thus, avoid some of the problems that come with journaling - though, perhaps, at the cost of introducing a set of interesting new problems.
