---
title: Flexible data placement
url: https://lwn.net/Articles/1018642/
date: "May 2, 2025"
category: Block layer
author: "By Jake Edge May 2, 2025 LSFMM+BPF"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
May 2, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

At the 2025 Linux Storage, Filesystem, Memory Management, and BPF Summit (LSFMM+BPF) Kanchan Joshi and Keith Busch led a combined storage and filesystem session on data placement, which concerns how the data on a storage device is actually written. In a discussion that hearkened back to previous summits, the idea is to give hints to enterprise-class SSDs to help them make better choices on where the data should go; hinting was most recently [discussed at the summit in 2023](<https://lwn.net/Articles/932900/>). If SSDs can group data with similar lifetimes together, it can lead to longer life for the devices, but there is a need to work out the details. 

Joshi began by noting that the logical placement of data provided by the host system is not the same as the physical placement of it on the device. There is a question of where the placement decision is made; if there is a data creator and multiple layers between it and the device (e.g. filesystem, device mapper), it is the piece that is closest to the device that ultimately decides where the data goes, he said. Currently, data is generally written sequentially because there is a single append point in a single open erase block on the device. 

[ ![\[Kanchan Joshi\]](https://static.lwn.net/images/2025/lsfmb-joshi-sm.png) ](<https://lwn.net/Articles/1019534/>)

[Flexible data placement](<https://semiconductor.samsung.com/us/news-events/tech-blog/nvme-fdp-a-promising-new-ssd-data-placement-approach/>) (FDP) is an NVMe SSD feature that allows writes to be tagged to indicate whether they should be grouped together or not. SSDs with FDP can have multiple append points in separate erase blocks in order to group the data based on its tag. It is not an error to write untagged data or with an invalid tag, however. It is an open question whether the applications or the layers between, like filesystems, should be deciding which tags to apply; the device itself does not care, but if the data is tagged, it ""can get grouped as originally intended"", Joshi said. 

Busch said that SSDs generally have a lot of resources to do things in parallel, but that ""without any hints, it's not going to know what the separation should be"". Hints would allow multiple applications to be writing without sharing the resources. These hints will also help reduce write amplification because data with the same lifetime can be placed (and updated or erased) together. 

Josef Bacik said that he knew Busch had run some experiments using Btrfs that would group data writes separately from metadata writes, but that the performance improvements were not that large. Busch agreed, noting that simply separating data from metadata was not granular enough. It would be better if the B-tree writes could be separated from the journal writes, for example. 

Bacik suggested that he could tag based on the subvolume ID for a write, which might improve things. Providing a different tag for data based on grouping operations with similar write-time-to-discard (or -overwrite) characteristics will make a difference, Busch said. Bacik asked about the number of tag values available. Busch said the tag is 16 bits, but that today's hardware does not make use of all of them; the range is from eight tags up to low hundreds of tags. There is a likelihood of running out of tags depending on how they are allocated. 

Should filesystem developers just start using the feature and hope that it is helping out, Bacik asked. Previous efforts of this sort lacked for any kind of feedback mechanism, Busch said, but the current protocols do provide ways to get an SSD endurance measure, which allows running experiments to see if the tagging is helping. The numbers provided measure the write amplification of a workload, he said; the number of bytes written to the device is reported along with the amount of data that was actually written to get those bytes on the media. Testing should probably be done with identical workloads on two systems, one without tagging and one with a tagging scheme. 

All of the SSDs that Meta (where both Busch and Bacik work) is buying have FDP, Busch said, though it is not enabled by default. It takes a few minutes to configure it for a new drive. All of the major vendors are supporting the feature, but each does it a little differently, so the amount of improvement will vary between SSD types. 

[ ![\[Keith Busch\]](https://static.lwn.net/images/2025/lsfmb-busch-sm.png) ](<https://lwn.net/Articles/1019535/>)

Chris Mason asked if there had been testing done with other filesystems that have a journal, such as ext4 or XFS. ""There are filesystems with a really specific lifetime for a very heavily used part of the SSD and Btrfs is not one."" Busch said that the filesystem testing had not been particularly extensive; the focus shifted to the applications, but he agreed that tagging journal writes for those filesystems might have a big impact. 

Ted Ts'o said that there had been some testing done long ago, perhaps by Martin Petersen, that would separate database log writes or filesystem journal writes for a particular type of storage device. The results were encouraging, but the storage devices were expensive and hard to obtain, which meant that developers did not have much of a chance to experiment. 

Petersen said that the FDP model is fine for use cases where applications or filesystems have tagging added based on known workloads on a particular SSD model. ""But that's not really a good model for all the other use cases"". The reason that earlier hinting efforts failed also exists for FDP: there are, say, eight tags that need to somehow be split up between the various filesystems that are on the device. The problem is that the tag values are a scarce resource. If they were not, things could always be tagged based on how the data should be grouped, but FDP and the earlier mechanisms are not general enough; each drive, filesystem, and application combination has to be tested individually to see what works. 

Zach Brown said that he was happy to see the increased visibility that FDP devices are providing; ""we are so used to devices being shitty black boxes"". Busch agreed, noting that it is likely that the lack of feedback is part of what caused earlier hinting efforts to fail. 

The [current patches](<https://lwn.net/ml/all/20250203184129.1829324-1-kbusch%40meta.com/>) do not yet plumb the FDP feature through the rest of the system, Busch said. Instead, there is a passthrough that allows user space to write commands directly to the NVMe device, ""so you have full control there"". It exposes the number of tags available as `max_write_streams` for the disk in sysfs. ""The passthrough interface is not a particularly pleasant interface to use"", he said. NVMe commands have to be constructed in user space, which is not the right level of abstraction. So there is a new write-stream command for io_uring that provides a nicer way to access the feature. It is only available for direct I/O and he questions whether it even makes sense to hook it up for buffered I/O. 

Joshi said that connecting up FDP to filesystem I/O in the iomap interface is still an unsolved problem. There are plans of that sort which will need to be discussed, he said. Once a filesystem is mounted, it will own the block device and, thus, will see and can manage all of the write streams (tags). A filesystem can support application-managed write streams, with a mapping to the hardware tags based on its rules, or it can manage the streams itself directly. Those two may not be mutually exclusive, so a filesystem could choose to support a hybrid mode. 

It will require a new user interface, as well as per-filesystem enablement, even just for the application-managed case, he said. There is also the question of whether there should be generic application-management code for write streams, so that each filesystem does not need to implement their own. 

Stephen Bates asked whether the write streams were per-NVMe-namespace or global to the device. They are global, Busch said, but subsets of the tag values can be specifically assigned to a namespace. 

Busch said that he had hoped to use the existing write-hints interface, which is currently a no-op, for FDP. Christoph Hellwig said that the filesystems should still be involved in order to map the application-provided tag values properly. That requires lots of work in the filesystems, Busch said, while the write-hints option does not; it would be better for the filesystems to be involved, but it is an easier path, with some of the benefits, to leave it to the applications. 

Bacik would rather not put Btrfs, for example, in the middle as the arbiter of the tags, but filesystems may want to also use the tags. Busch said that filesystems could reserve some of the tag space for themselves if they are the arbiter, but even that may result in tag collisions with filesystems on other partitions. For that reason, Bacik would like the block layer to do the arbitration. 

Petersen said that there had been experiments done along the way where filesystems would tag writes based on attributes, such as journal writes versus random writes. A scheduler was added to do the mapping from the filesystem tags to the hardware streams. ""It worked. It wasn't pretty, but it had the benefit of being flexible, because you could change that scheduler to match your application workload"". Now, developers could do something similar with a BPF program as a scheduler that was tuned for the workload. 

One of his complaints about the application-driven hints is that the kernel already has the knowledge that it needs. The application should not have to tell the kernel that these writes are for an application and are bound for a particular file—""we know that, we're the kernel"". The user interface should not be tied to the hardware; ""we can, in the kernel, schedule resources, that's why we exist"". 

There are applications that could make use of multiple streams within a single file, though, Javier González said. Joshi agreed, saying that the filesystem and block layer cannot necessarily make the best decisions on the allocation of the write streams. Bacik said that he did not want the filesystems getting in the way of applications doing what they need to do. 

Ts'o said that the small number of streams available is what makes things difficult; if there were an infinite number, for the sake of argument, the inode number could be used as the tag ""and let the block layer sort it all out"". Since that is not the case, something needs to allocate the streams, which may mean denying them to some requesters; every application will claim that its files are the most important, of course. 

The session got a little chaotic toward the end, with multiple sub-conversations that made it hard to follow, perhaps because it had run well into the subsequent break. It turned to ways to measure how well (or poorly) the system is tagging its data. The measurements that can be used are the write-amplification information from the drive along with the 99th-percentile (p99) write latencies, both of which should reduce when data with similar lifetimes is being grouped correctly, Busch said. Joshi finished the session by briefly going over some of the results that came from a recent [paper on FDP](<https://arxiv.org/pdf/2503.11665>).
