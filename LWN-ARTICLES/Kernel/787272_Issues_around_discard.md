---
title: Issues around discard
url: https://lwn.net/Articles/787272/
date: "May 6, 2019"
category: "Block layer-Discard operations"
author: "By Jake Edge May 6, 2019 LSFMM"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
May 6, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

In a combined filesystem and storage session at the 2019 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM), Dennis Zhou wanted to talk about discard, which is the process of sending commands (e.g. `TRIM`) to block devices to indicate blocks that are no longer in use. Discard is a "serious black box", he said; it is a third way to interact with a drive, but Linux developers have no real insight into what its actual effects will be. That can lead to performance and other problems. 

Zhou works for Facebook, where discard is enabled, but the results are not great; there is a fair amount of latency observed as the flash translation layer (FTL) shuffles blocks around for wear leveling and/or garbage collection. Facebook runs a periodic [`fstrim`](<http://man7.org/linux/man-pages/man8/fstrim.8.html>) to discard unused blocks; that is something that the [Cassandra database](<http://cassandra.apache.org/>) recommends for it users. The company also has an internal delete scheduler that slowly deletes files, but the FTL can take an exorbitant amount of time when gigabytes of files are deleted; read and write performance can be affected. It is "kind of insane" that applications need to recognize that they can't issue a bunch of discards at one time; he wondered if there is a better way forward. 

[ ![\[Dennis Zhou\]](https://static.lwn.net/images/2019/lsf-zhou-sm.jpg) ](<https://lwn.net/Articles/787511/>)

An approach that Facebook is trying with Btrfs is to simply preferentially reuse logical-block addresses (LBAs), rather than discarding them. That requires cooperation between the filesystem and block layer, since the block layer knows when it can afford to do discard operations. What the FTL is doing is unknown, however, which could have implications on the lifetime of the device. He is looking for something that can be run in production. 

Erik Riedel asked what is special about a 1GB discard versus a 1GB read or write. You are asking the device to do a large operation, so it may take some time. But James Bottomley noted that discard had been sold as a "fast" way to clean up blocks that are no longer needed. Riedel suggested that vendors need to be held accountable for their devices; if `TRIM` has unacceptable characteristics, those should be fixed in the devices. 

Ric Wheeler said that there should be a tool that can show the problems with specific devices; naming and shaming vendors is one way to get them to change their behavior. Chris Mason noted that the kernel team at Facebook did not choose the hardware; it would likely choose differently. This is an attempt to do the best the team can with the hardware that it gets. 

Ted Ts'o said that he worries whenever the filesystem tries to second-guess the hardware. Different devices will have different properties; there is at least one SSD where the `TRIM` operation is handled in memory, so it is quite fast. Trying to encode heuristics for different devices at the filesystem layer would just be hyper-optimizing for today's devices; other devices will roll out and invalidate that work. 

The current predicament came about because the kernel developers gave the device makers an "out" with the `discard` mount flag, Martin Petersen said. If performance was bad for a device, the maker could recommend mounting without enabling discard; if the kernel developers had simply consistently enabled discard, vendors would have fixed their devices by now. Beyond that, some vendors have devices that advertise the availability of the `TRIM` command, but do nothing with it; using it simply burns a queue slot for no good reason. 

Riedel suggested that "name and shame" is the right way to handle this problem. If a device claims to have a feature that is not working, or not working well, that should be reported to provide the right incentives to vendors to fix things. 

Bottomley wondered if filesystems could just revert to their old behavior and not do discards at all. Wheeler said that could not work since discard is the only way to tell the block device that a block is not in use anymore. The bigger picture is that these drives exist and Linux needs to support them, Mason said. 

Part of the problem is that filesystems are "exceptionally inconsistent" in how they do discard, Mason continued. XFS does discard asynchronously, while ext4 and Btrfs do it synchronously. That means Linux does not have a consistent story to give to the vendors; name and shame requires that developers characterize what, exactly, filesystems need. 

The qualification that is done on the devices is often cursory, Wheeler said. The device is run with some load for 15 minutes or something like that before it is given a "thumbs up"; in addition, new, empty devices are typically tested. Petersen said that customers provide much better data on these devices; even though his employer, Oracle, has an extensive qualification cycle, field-deployed drives provide way more information. 

Zhou said that tens or hundreds of gigabytes can be queued up for discard, but there is no need to issue it all at once. Mason noted that filesystems have various types of rate limiting, but not for discard. Ts'o said it would be possible to do that, but it should be done in a single place; each filesystem should not have to implement its own. There was some discussion of whether these queued discards could be "called back" by the filesystem, but attendees thought the complexity of that was high for not much gain. However, if the queue is not going to empty frequently, removing entries might be required. 

In the end, Fred Knight said, the drive has to do the same amount of work if a block is discarded or just overwritten. Writing twice to the same LBA will not go to the same place in the flash. The FTL will erase the current location of the LBA's data, then write a new location for the block. All discard does is allow the erasure to happen earlier, thus saving time when the data is written. 

The problem is that kernel developers do not know what a given FTL will do, Ts'o said. For some vendors, writing the same LBA will be problematic, especially for the "cheap crappy" devices. In some cases, reusing an LBA is preferable to a discard, but the kernel would not know that is the case; it would simply be making assumptions. 

To a certain degree, Zhou said, "discard doesn't matter until it does"; until the device gets past a certain use level, writing new blocks without discarding the old ones doesn't impact performance. But then there is a wall at, say, 80% full, where the drive goes past the "point of forgiveness" and starts doing garbage collection on every write. There is a balance that needs to be struck for discard so that there are "enough" erasure blocks available to keep it out of that mode. 

That is hard for SSD vendors, however, because they cannot reproduce the problems that are seen, so they cannot fix them, an attendee said. The kernel developers need to work more closely with the device firmware developers and provide workloads, traces, and the like. Wheeler said that reporting workloads and traces is the "oldest problem in the book". We have to do better than we are doing now, he said, which is to provide nothing to the vendors, he said. 

Bottomley pushed back on the idea that preferentially reusing LBAs was the right path. If there are blocks available to be written, reusing the LBA is worse, as it will fragment extent-based filesystems. If there is an erase block available, no erase need be done on a write to a new LBA, and if there isn't, it is the same as reusing the LBA; so rewriting LBAs actually compounds the problem. 

The exact behavior is specific to the filesystem and workload, however. The sizes of erase blocks are not known to the kernel, or even by kernel developers, because the drive vendors have decided to keep them secret, Bottomley said. So every decision the kernel makes is based on an assumption of one kind or another. 

Riedel still believes this is a qualification problem at its core. The right solution is for the drives to do a good job with the expected workloads. But Petersen reiterated that by giving the vendors an out with the `discard` flag, nothing will ever change. Fixes will "never happen". 

The core of the problem is that reads and writes need to happen immediately, Zhou said, while discards can wait a bit without causing much of a problem. But there are different viewpoints on the problem itself, Ts'o said; desktop distributions will be different from servers or mobile devices. Mason noted that if you asked ten people in the room how discard should work, you would get something like 14 answers. 

The session wound down without much in the way of resolution. There was talk of some kind of tool that could be used to reproduce the problems and gather traces. There was also talk of rate-limiting discards, but no one wants to do a "massive roto-tilling" of the block layer given all of the unknowns, Ts'o said; he suggested any change be done in an optional module between the filesystems and the block layer, which could be revisited in a few years.
