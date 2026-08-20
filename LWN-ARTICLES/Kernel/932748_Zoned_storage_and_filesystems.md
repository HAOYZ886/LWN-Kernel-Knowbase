---
title: Zoned storage and filesystems
url: https://lwn.net/Articles/932748/
date: "May 25, 2023"
category: "Block layer-Zoned devices; Filesystems"
author: "By Jake Edge May 25, 2023 LSFMM+BPF"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
May 25, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

Issues around zoned storage for filesystems was the topic of a combined storage and filesystem session at [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>) led by Bart Van Assche, Viacheslav A. Dubeyko, and Naohiro Aota. Zoned storage began with the advent of [shingled magnetic recording](<https://en.wikipedia.org/wiki/Shingled_magnetic_recording>) (SMR) devices, but is now implemented by [NVMe zoned namespaces](<https://zonedstorage.io/docs/introduction/zns>) (ZNS) as well. SMR devices can have multiple zones with different characteristics, with some zones that can only be written in sequential order, while other, conventional zones can be written in any order. The talk was focused on filesystems using the sequential type of zones since the conventional zones are already well-supported in Linux and its filesystems. 

[ ![\[Bart Van Assche\]](https://static.lwn.net/images/2023/lsfmb-vanassche-sm.png) ](<https://lwn.net/Articles/932828/>)

Van Assche began by giving an overview of zoned storage and its advantages; he quickly went through some bullet points from [the talk slides](<https://docs.google.com/presentation/d/1u109nKNbDA0b7vzeeYUE_6hZ2PBy3XLAWutDl7XA464>). For NAND flash devices, having sequential zones means that they can have a smaller logical-to-physical (L2P) mapping table, which improves performance. In addition, these zones eliminate internal garbage collection and the consequent write amplification, which allows the host to have better control over the latency of writing to the device. Read performance can also be improved because filesystems can allocate a contiguous logical-block-address (LBA) range for files. 

He then turned to the zoned-storage interface. Zones are contiguous LBA ranges that do not overlap with other zones; multiple zones can be written simultaneously. There are four states for a zone: empty, open, closed, or full. Zones that are either open or closed are considered active; devices may have limits on the number of active zones. 

#### Powers of two

He stated that the NVMe standard specifies that zone sizes are always a power of two, but was corrected by several attendees. Linux imposes that restriction, not the standard. Multiple NAND flash vendors want to be able to have non-power-of-two (npo2) zone sizes. In particular, vendors of [Universal Flash Storage](<https://en.wikipedia.org/wiki/Universal_Flash_Storage>) (UFS) devices want more flexibility in the zone sizes. 

Pankaj Raghav of Samsung has posted [patches](<https://lwn.net/ml/linux-kernel/20220923173618.6899-1-p.raghav@samsung.com/>) for supporting zone sizes that are not a power of two. Android also needs this support, Van Assche said. He wondered if the patches were ready to go upstream at this point. He was hoping that block maintainer Jens Axboe would be present for the discussion, but that was not the case. 

Josef Bacik wondered if the Linux filesystems community really cared one way or the other. He asked Johannes Thumshirn if Btrfs cared, for example. Thumshirn said that he thought it would be messy to support npo2, but that the problems could perhaps be considered bugs and get fixed. Bacik asked how many of these devices actually exist today. Damien Le Moal said that effectively everything on the market today has zones that are sized as a power of two. 

[ ![\[Viacheslav A. Dubeyko\]](https://static.lwn.net/images/2023/lsfmb-dubeyko-sm.png) ](<https://lwn.net/Articles/932828/>)

Le Moal said that his view is that flash-based zones should look like the existing SMR sequential zones, all of which have sizes that are a power of two. As yet, there are few deployed flash-based zoned-storage systems, so avoiding confusing things between SMR and flash devices was desirable. The UFS vendors are trying to push npo2 to avoid having to add more functionality in their firmware, he said. "Do we want to take the burden of dealing with the non-power-of-two, instead of the drive vendors doing it?" 

Van Assche said it is more than just UFS vendors that would like to do this. Le Moal would still prefer that the drive vendors handle this and he does not see why there would be performance problems in doing so, as has sometimes come up. Others disagreed, or at least thought that there was enough push for npo2 from customers of various sorts that _something_ should be done. One attendee suggested a middle layer that would mediate between the filesystems and devices; extracting maximum performance is not really needed for these devices. "Let's just be done with it, please." From the frustration expressed, it is clear that the topic has come up a lot without getting resolved. 

Bacik said that he truly did not care, and thought that was generally true for filesystem people, but he would also like to see this problem resolved in some fashion. He looked briefly at the patches, which did not seem too invasive to him; "I'm not the block-layer guy, so I could be wrong, and Jens isn't here to yell at me". He does not understand "why we are fighting about this, if it's not that big of a deal to support". 

Someone pointed out that Christoph Hellwig was adamantly opposed to the npo2 support; "now I understand it", Bacik said with a laugh. Hannes Reinecke suggested that even the middle-layer approach that was suggested would get strong opposition from Hellwig (who was at the summit, but not at this discussion). Le Moal said that so far all of the reasons he has heard for supporting npo2 in the kernel were wrong and demonstrate a misunderstanding of zoned storage on the part of device makers. If that support goes into the kernel, it should only be done if there are sensible reasons to do so, he said. 

There was a fair amount of disagreement in the room, with people talking over each other and several simultaneous side conversations taking place. It was not particularly heated, but was somewhat chaotic and hard to follow. Van Assche said that there were not good arguments either for or against the npo2 support in his mind, but Android, at least, is being pushed hard by the storage vendors. The ultimate decision is Axboe's, Bacik said; more discussion of it in the room is not really going to change anything, so he suggested moving on. 

#### Zoned Btrfs

[ ![\[Naohiro Aota\]](https://static.lwn.net/images/2023/lsfmb-aota-sm.png) ](<https://lwn.net/Articles/932828/>)

At that point, Aota switched over to the status of [zoned-storage support in Btrfs](<https://lwn.net/Articles/853308/>), which he has been [working on](<https://lwn.net/Articles/788851/>) for a number of years now. Btrfs supports both SMR and ZNS devices, with the latter added for the 5.16 kernel. SMR works well, but there are some problems with the ZNS support, he said. 

Currently, Btrfs on ZNS can report `ENOSPC` even when there is still space on the device due to zones being activated at reservation time, rather than only while data is being written. That means there may be no zones available to be activated when data needs to be written. There can also be slow performance because metadata overcommit is disabled in Btrfs on zoned storage. He is reworking some of the code to address these problems, he said, which will allow the metadata overcommit to be re-enabled. 

#### Zone sizing

Dubeyko then shifted gears to another topic: what is the best zone size based on the differing needs of filesystems and SSD devices? Smaller zones (hypothetically 128KB) are more complicated for the device because they require a huge mapping table and a complex mapping scheme. But, for a filesystem, a small zone can have smaller extents, with faster reclaim, lower garbage-collection overhead, and faster read I/O, he said. Larger zones (2GB, for example) have a lot of negatives for filesystems, but are much easier for the devices. He wondered if it might make sense to allow filesystems to choose among a few different zone sizes for a device. 

Le Moal said that the zone size and overall capacity of the device have to work together. A 16TB drive with 128KB zones is "going to suffer"; the number of zones in the device makes a difference. He said that it is also not something that can be changed at the software level; it is up to the drive vendors to choose a zone size that makes the most sense for the most use cases of their hardware. 

One attendee said that they think the next generation of ZNS drives will generally have zones of around 50 or 100MB, and wondered if that was a reasonable size for filesystems. He believes that the 1-2GB zones used in current devices will likely be around 100MB in devices for high-volume deployments. Ted Ts'o said that he was confused why the zone size was even being discussed in the room, "because, ultimately, I don't think it's up to us". The market will dictate its needs to the vendors, so if a high-volume handset maker, such as Samsung, were to say that it wants UFS devices with zones of a certain size, that's what will be built, he said. Others generally agreed with that as time ran out on the session.
