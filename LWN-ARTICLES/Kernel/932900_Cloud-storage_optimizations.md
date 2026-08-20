---
title: "Cloud-storage optimizations"
url: https://lwn.net/Articles/932900/
date: "May 26, 2023"
category: Block layer
author: "By Jake Edge May 26, 2023 LSFMM+BPF"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
May 26, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

"I/O hints" for storage devices, which are meant to improve performance by giving the devices extra information about the nature of the I/O, have a long history with Linux. But the code for write hints was "ripped out last year", according to a [message](<https://lwn.net/ml/linux-fsdevel/Y%2F7L74P6jSWwOvWt@mit.edu/>) from Ted Ts'o proposing a discussion about new optimizations for cloud-storage devices. That discussion took place in a combined storage and filesystem session at the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>). In it, Ts'o proposed that the Linux community define its own set of hints rather than just following along with the hints in the standards—which have largely been ignored by the vendors in any case. 

#### Background

He began by pointing out that "we have been talking about this set of storage extensions for freaking ever"; the earliest LSFMM discussion that he found was [from the 2013 event](<https://lwn.net/Articles/548296/>), but he wondered if that was actually the earliest. There is mention of I/O hints in the [report](<https://lwn.net/Articles/490501/>) from day two of LSFMM 2012 that indicates the topic had already been around for a while, but perhaps not before that at LSFMM. "Here we are ten years later—possibly longer", he said, to knowing chuckles around the room; he wanted to reflect a bit on why that was. 

[ ![\[Ted Ts'o\]](https://static.lwn.net/images/2023/lsfmb-tso-sm.png) ](<https://lwn.net/Articles/933013/>)

Working with the standards committees is "slow and expensive"; he has done it and would not necessarily recommend it for others. It requires a lot of travel and there are several bodies involved, which multiplies the problem, especially in times where budgets are tight. But, then, even if a spec gets approved, hardware vendors rarely actually implement these features in easily available devices; it is often only available in high-end, extremely expensive drives. 

As a result of that, I/O hints were added to the kernel but were removed around 18 months ago because no one was using them, he said. "They're back" was heard from the audience to more chuckles. But Ts'o thinks things can be a bit different this time around because of the prevalence of cloud-based emulated block devices, which are essentially software-defined storage. Those devices can be updated with new features much more easily and quickly than waiting for hardware vendors to decide to implement something. In addition, in the past "hardware vendors would care about $OTHER_OS" and did not care what Linux people thought; but these days the dominant OS running on cloud virtual machines is Linux. 

Ts'o said that there is a weekly call among ext4 and other filesystem developers that coincidentally has attendees from Oracle, Amazon, and Google, who are, of course, cloud vendors. Many of the call attendees are thinking about doing similar things with their filesystems, which involve "making assumptions about how the emulated block device in the cloud works". It occurred to him that they could do more than that; "the somewhat radical idea" that he wanted to propose is that the Linux community could add its own vendor extensions that could be used by these devices. 

Instead of some storage vendor being responsible for the extension, it would come from the Linux community. A reference implementation could be created for QEMU and if one or more cloud vendors could be convinced to adopt it, "then it could be purposely built for us". Developers would not have to try to figure out how to map the SCSI I/O hints from a decade ago to Linux, he said. 

Storage-track organizer Martin Petersen pointed out that in his hints work from ten years ago, he had mapped [`posix_fadvise()`](<https://man7.org/linux/man-pages/man2/posix_fadvise.2.html>) flags to SCSI and NVMe hints; he shopped that around to various storage vendors as what would make sense for Linux "and it went nowhere". He is strongly in favor of reviving the effort and calling it a "Linux cloud" extension; "it makes a ton of sense, it fixes a ton of performance problems, and it is like 150 lines of code". 

#### Cloud optimizations

Given that attendees seemed to be in favor of the overall plan, Ts'o wanted to talk about specific optimizations that he and others are thinking about. The cloud vendors have observed that MySQL and PostgreSQL both use 16KB database pages and would like to be able to write those in all-or-nothing fashion. That guarantee could come from the kernel or the hardware, he said, but the requirement is for no "torn writes" (i.e. partial writes). 

NVMe already has a an atomic-write extension and one is being added to SCSI, but with slightly different semantics, Ts'o said. But, today, "as an accident of implementation", due to the flags that get passed in the BIO for a direct I/O write, the block layer will not tear an aligned 16KB write; it "will not split them apart in awkward places". 

Buffered I/O is not treated that way, he said, which can lead to torn writes. But for direct I/O, he and others have "desk-checked the code" as well as running torture tests to try to cause torn writes. There are some who are thinking of deploying this as it stands, but others are looking for a guarantee from the operating system rather than just rely on an accident of the implementation. 

An OS guarantee is a reasonable request, Ts'o said; in addition, getting some kind of atomic solution for buffered I/O would be great because PostgreSQL only does buffered I/O. This would allow database systems to eliminate their double-buffered writes. So far, it seems to work fine for the cloud-storage devices; "maybe there are some weird semantics between NVMe and SCSI, but we don't care". 

It would be nice if the block layer could find out whether the device guarantees that it will not tear for aligned writes of, say, 16, 32, or 64KB, so that the block layer can also split on those boundaries. Storage-track organizer Javier González pointed out that there is upcoming LSFMM session on support for large block sizes; there are already patches for some of that support available. 

Luis Chamberlain, who would be leading the large-block discussion the next day, wondered about the limit of the size of the atomic writes that users want and how that relates to the block size that the device specifies. Keith Busch said that for NVMe SSDs today, the sizes for atomic guarantees range from 4KB up to 64KB. But Fred Knight pointed out that there is a large storage vendor that guarantees atomic writes of "hundreds of megabytes", but the block size is 4KB. Since a large vendor has done that, he suspects that others will too. Chamberlain concluded that there would be value in supporting block sizes beyond 64KB. 

Ts'o said that providing information that a set of blocks is associated with a particular inode could be used by storage devices for, say, garbage collecting all of them together. He does not know how practical that actually is, but as a filesystem developer he has no problem adding the inode information if it will help. Petersen said that he and Christoph Hellwig had a proposal like that, using a hash of the inode number, around ten years ago that also did not go anywhere. But James Bottomley wondered if it even mattered; since there are mostly extent-based filesystems that write large extents, can't the storage devices just use the large write as a signal that the blocks go together? Ts'o said that was probably workload-dependent, but that this particular optimization was not really one of his priorities. 

A more interesting optimization in his mind is giving the device hints about whether a read is actually synchronous from an application or whether it is coming from the block layer doing a readahead of some kind. But Petersen and Josef Bacik said there is already a flag being used for that; Petersen said that it is needed because a failed readahead is not treated the same as a failed application read. 

Another optimization, which has probably seen the most work over the years, Ts'o said, is to provide a hint that a given write is for data, metadata, or a journal. That journal indication could be for a filesystem journal or a database journal. That could allow the storage devices to prioritize the writes that are truly important versus those from background activities like backups. 

#### Working group

He thinks that a working group including cloud-vendor representatives could define something along those lines, which could be implemented in QEMU. Using that to demonstrate the benefits could lead the cloud vendors to start implementing the feature. Bart Van Assche asked that Android be included in any such working group; the project is working on a proposal to standardize write hints to distinguish between data and metadata writes. González said that the NVMe device in QEMU is only used for compliance testing, not for performance, so there has been talk of creating another NVMe device for QEMU with a fast path that could go directly to a [VFIO](<https://docs.kernel.org/driver-api/vfio.html>) passthrough device. 

There was some fast-paced disagreement about whether the NVMe and SCSI standards bodies needed to see an open-source implementation before actually standardizing something. In the end, that may not matter, Ts'o said, if there is a "Linux cloud" vendor extension, things that fall under it do not need to work for the hardware vendors. He has observed that sometimes those vendors are more interested in throwing sand in the gears of the standardization process than they are in adding features—especially if they perceive it might give competitors an advantage. That statement was met with laughing denials from various parts of the room. 

In fact, the Linux community can move much more quickly without having to go to standards meetings in far-flung places multiple times per year, Ts'o said. "We can just simply make something that works"; people who can go to the standards meetings can take that work and standardize it if they want. He thinks it might be easier to align the cloud-storage people, which can result in a quicker turnaround on these kinds of features. 

González asked if Ts'o had some kind of governing or organizing body in mind for this work, but Ts'o said he had not gotten that far. He thought that something informal, which resulted in something that works in QEMU, would be sufficient, but if a more formal organization is needed, the Linux Foundation would be an obvious possibility. His suggestion would be to keep the process as lightweight as possible though, and liked Petersen's idea that the linux-fsdevel mailing list be the "organization".
