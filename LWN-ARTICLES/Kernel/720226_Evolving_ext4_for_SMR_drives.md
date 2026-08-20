---
title: Evolving ext4 for SMR drives
url: https://lwn.net/Articles/720226/
date: "April 19, 2017"
category: "Filesystems-ext4"
author: "By Jake Edge April 19, 2017 Vault"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
April 19, 2017

* * *

[Vault](<https://lwn.net/Archives/ConferenceByYear/#2017-Vault>)

At the 2017 Vault conference for Linux storage, Ted Ts'o and Abutalib Aghayev gave a talk on some work they have done to make the ext4 filesystem work better on [shingled magnetic recording (SMR)](<https://en.wikipedia.org/wiki/Shingled_magnetic_recording>) drives. Those devices have some major differences from conventional drives, such that random writes perform quite poorly. Some fairly small changes that were made to ext4 had a dramatic effect on its performance on SMR drives—and provided a performance boost for metadata-heavy workloads on conventional magnetic recording (CMR) media as well. 

Ts'o said that he and Aghayev, who is a student at Carnegie Mellon University (CMU), had developed the ext4 changes; two professors, Garth Gibson at CMU and Peter Desnoyers at Northeastern, also provided useful input and advice. 

#### SMR basics

SMR drives pack more data into the same space as CMR drives by overlapping the tracks on the platter. Sequential writes will work well with SMR, while overwriting existing data will require copying data from adjacent tracks and rewriting it in a sequential fashion. SMR is targeted at backups, large media files, and similar use cases.

[ ![\[Ted Ts'o\]](https://static.lwn.net/images/2017/vault-tso-sm.jpg) ](<https://lwn.net/Articles/720414/>)

To the extent that rotational drives will stick around, SMR will be with us, Ts'o said. There are additional technologies "coming down the pike" that will be compatible with SMR. Millions of SMR drives have shipped and even consumers are using the technology for backups while using SSDs for data they need faster access to. 

There are two kinds of SMR drives, drive-managed and host-managed; the talk (and work) focused on drive-managed SMR, rather than host-managed, where the operating system has to actively manage the storage device. For drive-managed SMR, there is a shingle translation layer (STL) that is akin to the flash translation layer (FTL) for SSDs. The STL hides the various zones in an SMR device, which might be 256M in size and need to be written sequentially; it presents a block interface that masks the device's constraints. 

SMR disks typically have a persistent cache that is a lot faster than a CMR disk. The theory is that if there is idle time, and most disks in enterprise settings will have some, data can be moved from the persistent cache to the disk itself in a sequential manner at that time, Ts'o said. In addition, idle time allows for various cleaning and housekeeping tasks. As long as there is room in the persistent cache, writes to the device are extremely fast, but once it fills up, throughput drops off drastically. 

The persistent cache is invisible to the kernel unless the vendor provides some magic command to query its size and other characteristics. The exact behavior of the STL is vendor specific and subject to change, much like the situation with FTL implementations. But flash is so fast that it is hard to notice the difference when the translation layer chooses to write in different locations; for the STL, writing to the persistent cache is so much faster than to disk that it is quite noticeable. 

The STL will try to recognize sequential writes and bypass the persistent cache for those. In some ways the persistent cache is like the ext4 journal, Ts'o said. With a random write workload, once the persistent cache is full, each write becomes a large read-modify-write operation. The exact details of the persistent cache, how much there is and where it is located on the disk, for example, will vary; some drives they tested had 25GB of persistent cache, others were different. 

#### Small changes

The work that he and Aghayev did was to make fairly small changes to ext4 (40 lines of code modified, 600 lines added in new files) that made a dramatic difference. Those changes improved the performance of metadata-light workloads by 1.7-5.4x. For metadata-heavy workloads, the improvement was 2-13x. 

The way that ext4 uses the disk is particularly bad for SMR devices, he said, because the metadata is spread across the disk. Metadata writes are 4KB random writes, which is the worst possible thing for SMR. Those writes can dominate the work that the STL has to do even when you are storing large video streams that are SMR-friendly. If there is lots of idle time, the change is not all that dramatic but, if not, performance drops off substantially while the STL turns the random writes into sequential ones. 

Ext4 uses writeahead logging, which means that it writes metadata to a journal, which is sequential, but then does random writes to put the metadata in its final location once the journal fills or the dirty writeback timeout is reached. That means that every metadata block is written twice, once to the journal and once to its final destination; why not use the journal entry as the authoritative entry? The block in memory can be mapped to the journal location and be marked as clean in the page cache; if it gets evicted and is needed again, it can be looked up in the journal. 

When the journal gets full, something needs to be done, however. Many of the blocks in the journal are not actually important because they have been updated by an entry later in the journal. For those that are still valid, they could be copied to the final location on disk or simply to a new journal as a sequential write. "If you squint", he said, it kind of looks like a log-structured filesystem for metadata. 

In order to make this all work, they grew the journal from 128MB to 10GB; "on an 8TB drive, what's 10GB between friends?" They tried smaller journals, which worked, but the journal fills more quickly, requiring more copying. 

#### Results

Aghayev then took over to report on the performance of the changes. They tested ext4 versus the new filesystem, which they call "ext4-lazy", on an i7 processor system with 16GB of memory. He started by presenting the performance on CMR drives. 

[ ![\[Abutalib Aghayev\]](https://static.lwn.net/images/2017/vault-aghayev-sm.jpg) ](<https://lwn.net/Articles/720415/>)

The first benchmark used eight threads to create 800,000 directories with a directory-tree depth of ten. Ext4 took four minutes to complete, while ext4-lazy only took two minutes. When looking at the I/O distribution, ext4 wrote 4.5GB of metadata, with roughly the same amount of journal writes. Since ext4-lazy eliminates the metadata writes with only a small increase in journal writes, it makes sense that the benchmark only took half the time. 

The second test was for the "notoriously slow `rm -rf` on a massive directory" case. That is slow for all filesystems, Aghayev said, not just ext4. To delete the directory tree created in the first test took nearly 40 minutes for ext4, but less than ten for ext4-lazy. Looking at the I/O distribution, ext4-lazy skips the metadata writes that ext4 does, but that is a fairly small part of the I/O for the test; most of the I/O is in metadata reads and journal writes for both filesystems. But the metadata reads for ext4 require seeking all over the disk, while ext4-lazy reads them all from the journal. 

For a metadata-light workload, with less than 1% of the I/O involving metadata, ext4-lazy shows a much more modest performance increase. Running a benchmark that emulated a file server, showed a 5% performance increase for ext4-lazy. He recommended reading the [paper [PDF]](<https://www.usenix.org/system/files/conference/fast17/fast17-aghayev.pdf>) from the USENIX File and Storage Technologies (FAST) conference for more information. 

He then turned to the benchmarks on SMR devices. For those devices, ext4-lazy benefits from the fact that it does not require much cleaning time, while ext4 results in extra work that needs to be done after the benchmark is finished. The directory-creation benchmark shows a smaller gain for ext4-lazy (just under two minutes for ext4 versus just over one minute) on SMR, but that doesn't take into account the cleaning time, which is zero for ext4-lazy, but a whopping 14 minutes for ext4. 

Measuring cleaning time is not straightforward, however. They used a vendor tool in some cases, but also cut a hole into an SMR disk drive so they could observe what it was doing. Aghayev's advisors thought the hole idea would never work, but the drive is still working a year after doing so, he said. "You're lucky" said one audience member. It is difficult to get vendors to give out information about the inner workings of the drive, Aghayev said, so cutting a hole was what they were left with. 

On SMR, the directory-removal benchmark took 55 minutes for ext4 and around 4 minutes for ext4-lazy but, once again, cleaning time is significant as well. Ext4 required ten minutes of cleaning, while ext4-lazy needed only 20 seconds. The file-server benchmark showed similar results, though with a twist. Two different SMR devices showed different characteristics for ext4 and for ext4-lazy. Both devices showed roughly 2x performance for ext4-lazy, but the performance of ext4 on the two devices also showed a nearly 2x difference. The same is true for ext4-lazy between the two devices, but the order is reversed; the device that performed much better for ext4 performed nearly 2x worse for ext4-lazy when compared to the other device. That reflects the different ways that the STL handles cleaning; one does all or most of it when the cache gets full, while the other interleaves it with regular I/O. 

In conclusion, Aghayev said, ext4-lazy separates metadata from data and manages the former as a log, which is is not a new idea. Spreading metadata across the disk was invented some 30 years ago, maybe it is time to revisit that idea, he said. It is based on the explicit assumption that the cost of random reads is high, but also the implicit assumption that the cost of random reads is the same as the cost of random writes, which does not hold for SMR. 

Someone from the audience asked about ext4-lazy on SSDs. Aghayev said he thought there would be a performance increase, but has not done the experiments. Ts'o said he thought it would be better on the FTLs used by low-end devices like those on mobile handsets. But if the CPU pushes hard enough, high-end devices may also benefit, one attendee said. There were various suggestions for ways to make ext4-lazy better, but Ts'o noted that an explicit goal was to make minimal changes to an existing production filesystem so that users would have confidence in running on it. 

[I would like to thank the Linux Foundation for travel assistance to Cambridge, MA for Vault.]
