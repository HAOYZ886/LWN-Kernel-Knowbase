---
title: Changing filesystem resize patterns
url: https://lwn.net/Articles/894629/
date: "May 11, 2022"
category: Resizing filesystems
author: "By Jake Edge May 11, 2022 LSFMM"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
May 11, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

In a filesystem session at the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM), Ted Ts'o brought up the subject of filesystems that get resized frequently and whether the default parameters for filesystem creation should change as a result. It stems from a conversation that he had with XFS developer Darrick Wong, who is experiencing some of the same challenges as ext4 in this area. He outlined the problem and how it comes about, then led the discussion on ways to perhaps address it. 

#### Problems

Linux filesystems were generally designed to support being resized, but the expectation is that they start as a fairly large filesystem and then big chunks are added periodically. Filesystem data structures are sized and created based on how big the filesystem being created will be. He gave the example of RAID array using md-raid that gets a new disk, then the filesystem is resized to take advantage of it. That new disk may be a substantial fraction of the size of the existing filesystem, but the filesystem is already rather large. 

[ ![\[Ted Ts'o\]](https://static.lwn.net/images/2022/lsfmm-tso-sm.png) ](<https://lwn.net/Articles/894713/>)

Another use case is with a few network-attached storage (NAS) projects that wanted to take a 100MB filesystem and install it using `dd`, then expand it to, say, 10TB. There were just a few projects like that, so the Linux filesystem developers were able to "pound on them until they did the right thing" and created larger filesystems to match the intended size, Ts'o said. But that same basic scenario is playing out in the cloud today. 

Cloud providers typically have some minimum virtual block-device size, say 10GB, where filesystems get created, then they are resized from there. For example, that 10GB filesystem might be expanded to tens of terabytes. The difference from the NAS projects is that there are many more of the cloud providers and many are fairly naive, he said. They use the [`mkfs`](<https://man7.org/linux/man-pages/man8/mkfs.8.html>) default parameters, which are geared toward a USB thumb-drive and may not work well on a giant filesystem. 

In addition, many cloud providers charge their customers based on the size of the virtual block device being used, which means that customers have good reason to wait until the filesystem is nearly full before expanding it. A common pattern is that once a filesystem gets, say, 99% full, another 1GB or 5GB are added; that pattern repeats over and over for the filesystem. "That tends to result in worst-case filesystem fragmentation." Most filesystems are not designed to work well when running nearly full, he said. 

#### Possible solutions

While the problems have been identified, solutions have not been; he was hoping that attendees had some interesting ideas. One thing that would be useful, Ts'o said, is to have a standard format for large filesystem images that could be inflated onto block devices into the full size of the filesystem. The ext4 developers have been experimenting with using the [qcow format](<https://en.wikipedia.org/wiki/Qcow>); there is a utility in [e2fsprogs](<https://github.com/tytso/e2fsprogs>) called [`e2image`](<https://man7.org/linux/man-pages/man8/e2image.8.html>) that can create these images. They only contain the blocks that are actually used by the filesystem, so they are substantially smaller than the filesystem they will create. The XFS developers have also been looking at [`xfsdump`](<https://man7.org/linux/man-pages/man8/xfsdump.8.html>), since it has some similar capabilities, but for XFS filesystems. 

When he and Wong were talking, they agreed that some single standard format would be useful. One possibility is qcow, but it is not well-specified and the QEMU developers, who created the qcow format, discourage its use as an interchange format, he said. Perhaps there are others that could be considered, but getting agreement between the various filesystems is important. That would help move away from the idea that `dd` is the state-of-the-art tool for transferring filesystems. 

Another possibility would be to change `mkfs` so that it "sometimes or always" created filesystems suitable for being expanded into huge sizes. That could be done in a simpleminded way by changing the defaults, so that it always creates a huge journal even on USB thumb-drives, for example. Amir Goldstein said that doing so would not work well, since some large percentage of these small devices would be consumed by a journal that is not useful for them. Ts'o said that it might not be the optimal solution, but it was something that could be done. 

Perhaps the block device could give some kind of hint to `mkfs` that would indicate it is one where the filesystem might grow in the future, Ts'o said. Those hints could say that a device is not resizable at all, such as the thumb-drive case, or that it is being installed in some virtual block device in the cloud, where expansion is fairly likely. That way, the USB thumb-drive defaults could be maintained, but the parameters could be switched to more suitable ones for the cloud case. There could be a set of heuristics based on the name of the device to try to figure out whether it is likely to grow. If the block device provides the hints, that pushes the problem down to the device drivers, but that code is in a position to know more about the underlying storage. None of these is a perfect solution, however. 

He also noted that providing hints does not solve the problem of customers who are trying to minimize their storage costs, by adding small amounts of storage whenever the filesystem fills. The performance of such filesystems is seriously degraded, he said, to the point where the customer is probably paying more for the extra computation needed than they are saving on storage. There is no solution there that he knows of, other than customer education, and the number of customers is vastly more than the number of filesystem developers, so that does not work well either. 

Goldstein asked if those creating filesystems that may expand would be using the `mkfs` option to put the resulting filesystem in a file; that could perhaps provide a hint to use a different set of parameters. Ts'o said that the NAS developers asked about this problem along the way, so the filesystem developers were able to give them a set of parameters that would work well for them. It does not work for the cloud case, however, since virtual block devices look like normal SCSI devices. Using heuristics based on the name of the device, or perhaps on some "magical SCSI attribute" that no one is setting right now, could be a way to recognize the cloud case. 

Josef Bacik said that [`blkid`](<https://man7.org/linux/man-pages/man8/blkid.8.html>) did not have any useful information but that the model name of the disk could perhaps be parsed to help determine if it is a cloud device. He would guess that cloud providers use a consistent set of those names within a given cloud. Ts'o agreed with that, and thought that the results should be encoded in `blkid` so that it does not have to be recreated each time. Bacik thought that made sense, so that filesystem developers could move to a single "source of truth" about the type of device. 

Chris Mason wondered if there were any statistics on how often people create small filesystems for the cloud and keep them small forever. He is concerned that making the decision solely on whether the device is a cloud type would punish users of small cloud filesystems. Ts'o said he did not have any statistics on that; he normally hears from customers who continually add 1GB chunks to their filesystems and then complain about the performance of them. 

Part of the problem is that the filesystems do not get information about the maximum size of the device; each cloud provider has its own minimum, maximum, and increment sizes, Ts'o said. Arguments could be added to `mkfs`, to allow the user to specify the maximum size, for example, but most users will not know about the options. Users generally use the distribution installer or the default cloud image, so there is no opportunity for them to provide that information. 

Goldstein asked: is it possible to resize the underlying parts of the filesystem, such as the journal, so that the problem is lessened? Ts'o said that it depends on the filesystem; ext4 can resize its journal because it allows the journal not to be contiguous, but he thinks that XFS needs a contiguous journal. Unfortunately, none of the XFS developers were able to attend LSFMM in person, but Eric Sandeen used the Zoom chat to point out that there is more than just the journal that needs to be adjusted. Mason also noted that there are a lot of data structures in a filesystem that would need to change as it gets larger. 

Bacik said that everyone in the room is well aware that "creating more options for users just creates more ways for things to go horribly wrong". The best path is to make the tools intelligent by default; they can gather information from `blkid` and elsewhere to make the best choice that they can. They will not always make the right decision, and perhaps options can be added for power users, The tradeoffs are likely to be filesystem-specific, Ts'o said, so giving the filesystems hints on what the likely use case is will allow them to do the right thing. 

There was some discussion of things that could be done to make filesystems more resilient to size increases. Ts'o listed a few things that ext4 could perhaps do, like moving to 4KB block sizes by default and auto-resizing the journal; Sandeen had mentioned that XFS should perhaps make filesystems with a large number of allocation groups more efficient as a partial workaround. Those things are filesystem-specific and Ts'o said that he was hoping to find ways to address the problems in a more general way. Time ran out on the session without any real solution of that nature, however.
