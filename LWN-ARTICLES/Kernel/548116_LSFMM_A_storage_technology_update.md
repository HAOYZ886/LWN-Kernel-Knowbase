---
title: "LSFMM: A storage technology update"
url: https://lwn.net/Articles/548116/
date: "April 23, 2013"
category: SCSI; Shingled magnetic recording
author: "By Jonathan Corbet April 23, 2013 LSFMM Summit 2013"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
April 23, 2013

* * *

[LSFMM Summit 2013](<https://lwn.net/Articles/LSFMM2013/>)

Rob Elliott stood up before the assembled group to discuss developments in the T10 (SCSI) standard community and beyond. There is a lot going on in the storage industry that will have to be answered in the Linux kernel. 

Rob started by stating that storage devices continue to get faster, to the point where they can saturate any interface they are connected to. Keeping up with these devices is increasingly hard for any operating system. He said we would see the introduction of "non-uniform I/O access" systems where I/O controllers are local to each CPU. Christoph Hellwig interjected that some developers have been working with such systems for the last fifteen years. It was acknowledged that the idea is not entirely new, but with the proviso that the parts of the kernel that can work effectively on such systems are limited — the "knowledge" of how to handle non-uniform I/O access has not been spread through the kernel as a whole. 

The "SBC-4" proposal envisions a new, highly reduced SCSI command set. SCSI drivers currently have a lot of special cases to handle the specific subset of commands supported by each device. SBC-4 devices would be guaranteed to support the full (reduced) command set and nothing more, eliminating a lot of special-case code. Of course, to SCSI developers, SBC-4 can also look like yet another special case to handle. Perhaps in an answer to that, Rob suggested the creation of a new, optimized SCSI disk driver that would only work with such devices. 

The other concern with SBC-4 was that, inevitably, USB drives would claim to support it as well. Since those drives are famous for not supporting any standard all that well, the likely result is a new set of heuristics and workarounds. That led to a brief discussion on the idea of certifying devices for use with Linux. The sense seemed to be that certification would be nice, but the manufacturers of dirt-cheap USB drives have no incentive to get their hardware certified to any standard, so the overall situation would not change much. 

There are a couple of interesting additions to the SCSI command set in the works. One is "WRITE ATOMIC" — an extended write operation that would either succeed fully or not at all. A filesystem could bundle a bunch of data into an atomic write with the understanding that there was no risk of the operation completing only partially. Atomic writes combine well with scatter/gather operations, which are the other addition. Scatter/gather operations need not specify a single contiguous region of the disk; instead, they provide a vector of offsets and lengths, scattering the data across the drive. A scattered write could combine a change to a file with the changes to the associated metadata in a single operation. That has the potential to eliminate the need for a lot of defensive measures currently in place to ensure metadata consistency. 

Once again, though, there is a problem: how well would these features be supported? Both are optional, and, for scatter/gather operations, there is no minimum number of segments that would have to be supported. Dave Chinner said that, for XFS to fully use this feature, it would have to support as many as 500,000 segments, holding 1-2GB of data, in a single operation. It seems unlikely that many devices will support operations that large. Ted Ts'o said that ext4, instead, could get away with a half dozen segments — but that would just be to make journaling more efficient; ext4 would still need its journal when operating in that mode. Josef Bacik also agreed that scatter/gather operations could make Btrfs logging more efficient, but it couldn't eliminate the need altogether. Btrfs, he said, can build up several gigabytes of dirty metadata; it is hard to imagine that it could all be handed to the drive in a single operation. 

There are various proposals out to improve the effectiveness of the "DIF" data-integrity mechanism. They mostly consist of using different CRC algorithms or adding the logical block number to the CRC to help guard against block aliasing. Other proposals include a "product type" field for multiple-card readers, "conglomerate LUNS" to handle vast numbers of logical unit numbers in virtualized environments, deadline scheduling for I/O priority ordering, adding a "force" bit to the UNMAP command to guarantee that old data would be unretrievable, and an option to sanitize old data when it is overwritten. 

Rob then moved to a set of "bogged-down proposals," the most important of which is shingled drives. A "shingled magnetic recording" (SMR) drive is a rotating drive that packs its tracks so closely that one track cannot be overwritten without destroying the neighboring tracks as well. The result is that overwriting data requires rewriting the entire set of closely-spaced tracks; that is an expensive tradeoff, but the benefit — much higher storage density — is deemed to be worth the cost in some situations. 

At least, the storage industry seems to think it's worth the cost. Most of the rest of the industry doesn't seem to be overly interested; to them, SMR drives just look like more complexity to deal with. But these drives are here anyway; they are the most cost-effective path toward larger capacities. The question is simply: how much of the resulting ugliness should be exposed to the system software? 

One option is to treat an SMR drive as if it were a traditional tape drive with the added benefit of random read access. Christoph liked this idea, saying it matched how the drives will be typically used: for backups and such. It also seems to match the biggest use case for these drives, which is to store the data which is driving much of the storage industry: photographs and cat videos. 

Martin Petersen asked if it was worthwhile to do "crazy stuff" in the I/O stack to support a technology which may be transitional at best. The answer was that the constraints that are driving the push to SMR drives will be around for a while. Once you start packing your bits closely enough, it becomes impossible to change them individually. It was suggested that solid-state storage could eventually run into similar problems. So the problem of SMR drives may be around for longer than anybody in the room might have liked. 

In any case, as James Bottomley pointed out, Linux has always been friendly toward experimental technology, even though not all of that technology makes it out to end users. Ric Wheeler added that supporting this technology is good for Linux in the long run. But, he added, we are likely to see more use of tiered storage systems in the future where SMR drives will be hidden behind a layer that provides a more reasonable interface to the host system. 

#### SCSI Express

There followed a plenary session ostensibly about SCSI express and the challenges of handling high-speed devices. The session was somewhat scattered, though, and hard to take good notes from. So, unfortunately, not much will be reported here. 

One topic that came up during the discussion was the multi-queue block layer patches that Jens Axboe has been working on for some time. It is fair to say that the developers in the room think that, maybe, those patches have had long enough to mature within Fusion-io and should perhaps be posted to the wider world. 

Jens agreed that the time had come. The patches, he said, are in his linux-block tree, but have not yet made it to any mailing list. They are in the form of a 72-patch series that does some "weird and crazy things"; the work is not complete, but it does appear to be stable. The main push is to get drivers away from using the `make_request()` mechanism when they need to try for higher performance. Using `make_request()` allows a driver to replace much of the block layer, at the cost of having to replicate much of the logic already found in the block layer. Driver conversion, he said, is relatively easy. 

The main barrier to merging the code, he said, was just the need to clean it up and add a few more features. James suggested that it should be posted for merging in the 3.11 cycle; any problems could be fixed up thereafter. Jens agreed; he has since [made his repository available](<https://plus.google.com/111643045511375507360/posts/iohyBeoz9Hj>) for those who would like to take an early look at the code.
