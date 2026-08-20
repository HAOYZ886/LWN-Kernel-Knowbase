---
title: "XFS:  There and back ... and there again?"
url: https://lwn.net/Articles/638546/
date: "April 1, 2015"
category: "Filesystems-XFS"
author: "By Jake Edge April 1, 2015 Vault 2015"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
April 1, 2015

* * *

[Vault 2015](<https://lwn.net/Archives/ConferenceByYear/#2015-Vault>)

In a thought-provoking—and characteristically amusing—talk at the [Vault conference](<http://events.linuxfoundation.org/events/vault>), Dave Chinner looked at the history of XFS, its current status, and where the filesystem may be heading. In keeping with the title of the talk (shared by this article), he sees parallels in what drove the original development of XFS and what will be driving new filesystems. Chinner's vision of the future for today's filesystems, and not just of XFS, may be a bit surprising or controversial—possibly both. 

#### History

In the early 1990s, "before I knew who SGI was", Chinner said, storage was exceeding capacities that could be addressed with 32 bits. The existing filesystem for SGI's Unix, IRIX, was the [Extent File System](<http://en.wikipedia.org/wiki/Extent_File_System>) (EFS) that only supported 32-bit addresses. At that time, 64-bit CPUs with large-scale multiprocessing were coming on, but systems with hundreds of disks were not performing well. 

XFS came about to fix those problems. The "x" originally just meant "undefined". This new undefined filesystem, xFS, had to support a number of features: fast crash recovery, large filesystems with both sparse and contiguous large files, and filesystems and directories that could hold huge numbers of files. The filesystem would need to support terabyte and petabyte capacities in the foreseeable future at that time, he said. 

[ ![\[Dave Chinner\]](https://static.lwn.net/images/2015/vault-chinner-sm.jpg) ](<https://lwn.net/Articles/638626/>)

Chinner showed a graph of the lines of code changed in XFS over time that went all the way back to the beginning and up through commits from the previous week. Because the full commit history for XFS is available, graphs like that can be made, he said. Another way to use the history is to track bugs back to their introduction. The oldest bug anyone has found in recent times was 19 years old, he said. 

The first production release of XFS was in December 1994 in conjunction with the release of IRIX 5.3. In mid-1996, XFS became the default filesystem in IRIX 6.2. The on-disk format had already gone through four versions. Even those early versions had the "feature mask" that allows XFS filesystems to turn on or off individual filesystem features. Those can be set for a particular filesystem at `mkfs` time; the mask will be consulted for various types of operations. It is a feature of XFS that makes it particularly flexible. 

The "IRIX years" of the late 1990s saw various feature additions. Hardware RAID array support was added in 1997, which resulted in XFS gaining the ability to align its allocations to match the geometry of the underlying storage. By 1999, the original directory structure was showing its age, so version 2 directories were added, increasing directory scalability to tens of millions of files. Chinner has personally tried 350 million files in a directory, but he doesn't recommend it; running `ls` on such a directory takes twelve hours, he said with a laugh. 

#### Shift to Linux

Shortly thereafter, feature development in XFS on IRIX slowed down. SGI had shifted its focus to large NUMA machines and distributed IRIX; development moved from XFS to the new [CXFS](<http://en.wikipedia.org/wiki/CXFS>) cluster filesystem. But SGI also had a new product line that was based on Linux, which didn't have a filesystem with the features of XFS. So a team was formed in Melbourne, Australia to port XFS to Linux. 

In 2000, XFS was released under the GPL. The first stable release of XFS for Linux came in 2001. And, in 2002, XFS was merged into the mainline for Linux 2.5.36. 

One of the unfortunate side effects of XFS and other filesystems being merged into Linux was the proliferation of what Chinner called "[Bonnie++](<http://en.wikipedia.org/wiki/Bonnie%2B%2B>) speed racing". People were using the filesystem benchmarking tool with little knowledge, which resulted in them "twiddling knobs" in the filesystem that they did not understand. The problem with that is that Google still finds those posts today, so those looking for better XFS performance are finding these [old posts](<http://everything2.com/title/Filesystem+performance+tweaking+with+XFS+on+Linux>) and following the instructions therein, which leads to the never-ending "`noatime,nodiratime,logbufs=8` meme" for XFS mount options. 

In the early 2000s, the Linux version of XFS started to diverge from the IRIX version with the group quotas feature. The version 2 log format was added in 2002, which helped make metadata performance much better. Also in 2002 came inode cluster delete and configurable sector sizes for XFS. In 2004, the 2.4 and 2.6 development trees were unified and the feature mask was expanded. XFS had run out of bits in the feature mask, he said, so the last bit was used to indicate that there is an additional feature mask to be consulted. 

A "major achievement" was unlocked in 2004, when SUSE shipped full XFS support in SUSE Linux Enterprise Server (SLES) 9\. It was a validation of all the work that SGI had done on the filesystem, he said. Someone from the audience spoke up to say that SLES 8 had shipped with XFS support in 2001, which seemed to surprise Chinner a bit. 

The mid-2000s saw lots of misinformation spread about XFS (and other filesystems) during the "filesystem wars", Chinner said. His [slides [PDF]](<http://events.linuxfoundation.org/sites/events/files/slides/dchinner-xfs-there-and-back.pdf>) contain several quotes from that time about "magical" features, including large capacitors in SGI power supplies that caused XFS not to lose data on power failure, zeroing of data after an unlink operation so that undelete was not possible, and the zeroing of all open files when there is an unclean shutdown. None of those were true, but they have become part of the XFS lore. 

More features came about in 2005 and 2006. Extended attributes were added into the inode, for example, which resulted in an order-of-magnitude performance improvement for Samba atop XFS. That time frame was also the crossover point where Linux XFS started outperforming IRIX XFS. On Linux 2.6.16, XFS achieved a 10GB/second performance on a 24-processor Altix machine, he said. 

A few years further on, the "[`O_PONIES`](<https://lwn.net/Articles/351422/>) wars" began. Chinner pointed to a [Launchpad bug](<https://bugs.launchpad.net/ubuntu/+source/linux-source-2.6.15/+bug/37435>) about an XFS file data corruption problem under certain conditions. The workaround was to use `fdatasync()` after renaming the file, but that would require changes in user space. The bug was closed as a "Won't fix", but none of the XFS developers were ever even consulted about it. It turns out that it actually was a bug in XFS that was fixed a year later. 

#### Shift to the community

"Sad days" arrived in 2009, when the SGI XFS engineering team was disbanded. The company had been losing money since 1999, he said. The community stepped in to maintain the XFS tree while SGI reorganized. After that, SGI intermittently maintained XFS until the end of 2013; he didn't mention that most of the work during that time was being done by developers (like himself) working at other companies. Chinner said he went away to the racetrack for a weekend and came back to find out that he had been [nominated to take over as maintainer](<https://lwn.net/Articles/573658/>). 

Since the shutdown of the XFS team, development on the filesystem has largely shifted to the wider community, he said. Development did not slow down as a result, however; if anything, it accelerated. But all of that work is not just about adding code, a lot of code has been removed along the way, as well. 

He rhetorically asked if XFS is "still that big, bloated SGI thing"? He put up a graph showing the number of lines of code (LOC) in XFS for each kernel release. The size of the XFS codebase started dropping around 2.6.14, bottomed out in 3.6, rose again until 3.15 or so, and has leveled off since. The level (just under 70,000 LOC) is lower than where the graph started with 2.6.12 (roughly 75,000 LOC). The line for Btrfs crossed that of XFS around 3.5 and it is at 80,000+ and still climbing; it hasn't leveled off, Chinner said. 

He listed the top developers for XFS, with Christoph Hellwig at the top, followed closely by Chinner himself. He called out early XFS developers Adam Sweeney and Doug Doucette, noting that they had done "tons of work" in a fairly small number of years. He quoted Isaac Newton (""If I have seen further than others, it is by standing upon the shoulders of giants."") and said that XFS came about "not because of me", but from the work of all of those others on the list. 

Recent and ongoing developments in XFS include sparse inode chunk allocation to support GlusterFS and Ceph, unification of the quota API, and reverse mapping for the internal B-trees (more reasons for doing so cropped up at the recently completed [LSFMM Summit](<https://lwn.net/Articles/lsfmm2015/>), he said). In addition, [reflink](<https://lwn.net/Articles/331808/>) support for per-file snapshots has been added, defragmentation improvements have been made, and support for the [direct access block layer](<https://lwn.net/Articles/591779/>) (DAX) has been added. There is "lots going on at the moment", he said. 

#### The near future

In among working on all of that, Chinner has been trying to plot out the next five years or so of XFS development. He did a similar exercise back in 2008 and posted some [documents](<http://xfs.org/index.php/Ideas_for_XFS>) to [XFS.org](<http://xfs.org/>). Over the years since then, all of the features in those documents have been ticked off the list, though reverse B-trees, which are 95% complete, are "the last bit". So, now is the time to plan for the future. 

There are known storage technologies that are pulling filesystem developers in opposite directions, he said. [Shingled magnetic recording](<https://lwn.net/Articles/637035/>) (SMR) devices and [persistent memory](<https://lwn.net/Articles/636096/>) are going to change things radically. The key for developers of existing filesystems is to figure out "good enough" solutions to allow those filesystems to work on SMR and persistent memory devices. Eventually, someone will come up with a filesystem that specifically targets SMR devices or persistent memory that "blows everything else away". 

Over the next five or more years, XFS needs to have better integration with the block devices it sits on top of. Information needs to pass back and forth between XFS and the block device, he said. That will allow better support of thin provisioning. It will also help with offloading block clone, copy, and compress operations. There are more uses for that integration as well, including better snapshot awareness and control at the filesystem layer. 

Improving reliability is another area where XFS will need more work. Reconnecting orphaned filesystem objects will become possible with the B-tree reverse mapping abilities. That will also allow repairing the filesystem while it is online, rather than having to unmount it to do repairs. Proactive corruption detection is also on the list. When corruption is found, the affected part of the filesystem could be isolated for repair. These features would make XFS "effectively self-healing", Chinner said. 

There is a "fairly large amount of work" on the list, he said. While he is targeting five years, he can't do it alone in that time period. If it is just him, it will probably take six or seven years, he said with a grin. 

#### Farther out

But we also need to be thinking a little further ahead. Looking at the progression of capacities and access times for "spinning rust" shows 8GB, 7ms drives in the mid-1990s and 8TB, 15ms drives in the mid-2010s. That suggests that the mid-2030s will have 8PB (petabyte, 1000 terabytes) drives with 30ms access times. 

The progression in solid-state drives (SSDs) shows slow, unreliable, and "damn expensive" 30GB drives in 2005. Those drives were roughly $10/GB, but today's rack-mounted ([3U](<http://en.wikipedia.org/wiki/Rack_unit>)) 512TB SSDs are less than $1/GB and can achieve 7GB/second performance. That suggests to him that by 2025 we will have 3U SSDs with 8EB (exabyte, 1000 petabytes) capacity at $0.1/GB. 

SSDs have lots of compelling features (e.g. density, power consumption, performance) and are cost-competitive. But persistent memory will be even denser and faster, while still being competitive in terms of cost. It is at least five years before we see pervasive persistent memory deployment, however. Chinner did caution that [memristors](<http://en.wikipedia.org/wiki/Memristor>) are a wild card that could be "a game changer" for storage. 

Those projections indicate that focusing on spinning rust would be largely misplaced. XFS (and other filesystems) have to follow where the hardware is taking them. SSD storage will push capacity, scalability, and performance much faster than SMR technologies will. Though "most of the cat pictures" on the internet will likely be stored on SMR drives before too long, he said with a chuckle. 

8EB is about as large as XFS can go. In fact, the 64-bit address space will be exhausted for filesystems and CPUs in the next 10-15 years. Many people thought that the 128-bit addresses used by ZFS were "crazy", but it doesn't necessarily look that way now. 

By 2025-2030, XFS will be running up against capacity and addressability limits. It will also be hitting architectural performance limits in addition to complexity limits from continuing to support spinning rust disks. The architecture of XFS will be unsuited to the emerging storage technologies at that time. 

All of those projections place a hard limit on the development life of XFS. It doesn't make sense to do a large-scale rework of the filesystem to support SMR devices, for example. SMR will be supported, but the project is "not going to turn everything upside down for it". There is a limited amount of life left in the filesystem at this point and SMR technologies will be outrun by other storage types. 

From Btrfs, GlusterFS, Ceph, and others, we know that it takes 5-10 years for a new filesystem to mature. That implies we need to start developing any new filesystems soon. For XFS, there is the definite possibility that this 5-7 year development plan that he is working on will be the last. In 20 years, XFS will be a legacy filesystem. That's likely true for any current filesystem; he may not be right, he said, but if he is, we may well be seeing the last major development cycle for all of the existing filesystems in Linux. 

[I would like to thank the Linux Foundation for travel support to Boston for Vault.]
