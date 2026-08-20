---
title: "LFCS: Preparing Linux for nonvolatile memory devices"
url: https://lwn.net/Articles/547903/
date: "April 19, 2013"
category: "Memory management-Nonvolatile memory; Solid-state storage devices"
author: "By Jonathan Corbet April 19, 2013"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
April 19, 2013

Since the demise of core memory, there has been a fundamental dichotomy in data storage technology: memory is either fast and ephemeral, or slow and persistent. The situation is changing, though, and that leads to some interesting challenges for the Linux kernel. How will we adapt to the coming world where nonvolatile memory (NVM) devices are commonplace? Ric Wheeler led a session at the 2013 Linux Foundation Collaboration Summit to discuss this issue. 

In a theme that was to recur over the course of the hour, Ric noted that we have been hearing about NVM for some years. NVM devices have a number of characteristics that distinguish them from other technologies. They are byte addressable like ordinary RAM, but unlike storage devices which have always been block-oriented. They are persistent: they do not lose state when the power goes away. They are comparable to ordinary memory in speed, and also in price, so they will not be as large as hard drives anytime soon. They also are not yet available for most of us to play with at any reasonable price. 

Early solid-state devices looked a lot like disks; they used normal protocols and were not so fast that the system could not keep up with them. That situation changed, though, with the next wave of devices, which were usually connected via PCI Express (PCIe). There is a lot of code in the I/O stack that [![\[Ric Wheeler\]](https://static.lwn.net/images/conf/2013/lf-collab/RicWheeler1-sm.jpg)](<https://lwn.net/Articles/547927/>) sits between the system and the storage; as storage devices get faster, the overhead of all that code is increasingly painful. Much of that code is not useful in this situation, since it was designed for high-latency devices. As a result, Linux still can't get full performance out of bus-connected solid-state devices. 

As an aside, Ric had a few suggestions to offer to anybody working to tune a Linux system to work with existing fast block devices. The relevant parameters are found under `/sys/block/_dev_ /queue`, where `_dev_` is the name of the relevant block device (`sda`, for example). The `rotational` parameter is the most important; it should be set to zero for solid-state devices. The CFQ I/O scheduler (selected with the `scheduler` attribute) is not the best for solid-state devices; the deadline scheduler is a better choice. It is also important to pay attention to the block sizes of the underlying device and align filesystems accordingly; see [this paper by Martin Petersen [PDF]](<http://mkp.net/pubs/storage-topology.pdf>) for details. 

Back to the topic at hand, Ric noted that, along with all the technical challenges, there are some organizational difficulties. Kernel developers tend to be quite specialized: at the storage layer, SCSI and SATA drives are handled by different groups. The block layer itself is maintained by a separate, very small group. There is yet another group for each filesystem, and we have a lot of filesystems. All of these groups will have to work together to make NVM devices work optimally on Linux systems. 

#### Crawling first

Making the best use of NVM devices will require new programming models and new APIs. That kind of change takes time, but the hardware could be arriving soon. So, Ric said, we need to make them work as well as we can within the existing APIs; this is, he said, the "crawl phase." In this phase, NVM devices will be accessed through the same old block API, much like solid-state devices are now. The key will be to make those APIs work as quickly as possible. It is a shame, he said, but we need a block driver that will turn this cool technology into something boring. There is also a need for a lot of work to squeeze overhead out of the block I/O path. 

Ted Ts'o suggested that, while it is hard to get applications to move to new APIs, it is easier to make libraries like sqlite use them. That should bring improved performance to applications with no code changes at all. It was pointed out, though, that users are often reluctant to even recompile applications, so it could still take quite a while for performance improvements to be seen by end users. 

The current "crawl" status is that block drivers for NVM devices are being developed now. We're also seeing caching technologies that can use NVM devices to provide faster access to traditional storage devices. The dm-cache device mapper target was merged for 3.9, and the [bcache](<https://lwn.net/Articles/497024/>) mechanism is queued for 3.10. Ric said that various vendor-specific solutions are under development as well. 

Getting to the "walk" phase involves making modifications to existing filesystems. One obvious optimization is to move filesystem journals to faster devices; frequently-used metadata can also be moved. Getting the best performance will require reworking the transaction logic to get rid of a lot of the currently-existing barriers and flush operations, though. At the moment, Btrfs has a bit of "dynamic steering" capability that is a start in that direction, but there is still a lot that needs to be done. 

It is also time to start thinking about the creation of byte-level I/O APIs for new applications to use; the developers are currently looking for ideas about how applications would actually like to use NVM devices, though. Ric mentioned that the venerable `mmap()` interface will need to be looked at carefully and "might not be salvageable." Application developers will need to be educated on the capabilities of NVM devices, and hardware needs to be put into their hands. 

That last part may prove difficult. Over the course of the session, a number of participants complained that these devices have been "just around the corner" for the last decade, but they never actually materialize. There is a bit of a credibility problem at this point. As Tejun Heo said, nothing is concrete; there is no way to know what the performance characteristics of these devices will be or how to optimize for them. The word is that this situation will change, with developers initially getting hardware under non-disclosure agreements. But, for the moment, it's hard to know what is the best way to support this class of hardware. 

Eventually, Ric said, we'll arrive at the "run phase," where there will be new APIs at the device level that can be used by filesystems and storage. There will be new Linux filesystems designed just for NVM devices (in a later session, we were told that Fusion-IO had such a filesystem that would be released at some unspecified time in the future). The Storage Network Industry Association has [a working group](<http://snia.org/forums/sssi/nvmp>) dedicated to these issues. All told, the transition will take a while and will be painful, Ric said, much like the move to 64-bit systems. 

#### Concerns

The subsequent discussion covered a number of topics, starting with a simple question: why not just use NVM devices as RAM that doesn't forget its contents when the power goes out? One problem with doing things that way is that, while NVM may perform like RAM, other aspects — such as lifespan — may be different. Excessive writes to an NVM device may reduce its useful lifetime considerably. 

There was some talk about the difficulty of getting support for new types of devices into Linux in general. The development community goes way beyond the kernel; there are many layers of projects involved in the creation of a full system. This community seems mysterious to a lot of vendors. It can take many years to get features to the point that users can actually take advantage of them. An example that was raised was parallel NFS, which has been in development for at least ten years, but we're only now getting our first enterprise support — and that is client support only. 

Another point of discussion was replication of data. With ordinary block devices, replication of data across multiple devices is relatively easy. With NVM devices that are directly accessed by user space, instead, the "interception point" is gone, so there is no way for the kernel to transparently replicate data on its way to persistent storage. It was pointed out that, since applications are going to have to be changed to take advantage of NVM devices anyway, it makes sense to add replication features to the new APIs at the same time. 

The issue of how trustworthy these devices are came up briefly. Applications are not accustomed to dealing with memory errors; that may have to change in the future. So the new APIs will need to include features for checksumming and error checking as well. Boaz Harrosh pointed out that, until we know what the failure characteristics of these new devices are, we will not be able to defend against them. Martin Petersen responded that the hardware interfaces to these devices are intended to be independent of the underlying technology. There are, it seems, several technologies competing for a place in the "post-flash" world; the interfaces, hopefully, will hide the differences between those technologies. 

In summary, we seem to be headed toward an interesting new world, but it's still not clear what that world will look like or when it will arrive. Chances are that we will have good kernel support for NVM devices by the time they are generally available, but higher-level software may take a while to catch up and take full advantage of this new class of hardware. It should be an interesting transition. 

[Your editor would like to thank the Linux Foundation for assistance with travel to the event.]
