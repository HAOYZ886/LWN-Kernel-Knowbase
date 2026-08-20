---
title: "Supporting the UFS turbo-write mode"
url: https://lwn.net/Articles/788721/
date: "May 20, 2019"
category: "Filesystems-Flash"
author: "By Jake Edge May 20, 2019 LSFMM"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
May 20, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

In a combined filesystem and storage session at the 2019 Linux Storage, Filesystem, and Memory-Management Summit, Avri Altman wanted to discuss the "turbo-write" mode that is coming for [Universal Flash Storage](<https://en.wikipedia.org/wiki/Universal_Flash_Storage>) (UFS) devices. He wanted to introduce this new feature to assembled developers and to get some opinions on how to support this mode in the kernel. 

NAND flash devices can store three bits per cell (triple-level cell or TLC), but it is much slower than storing a single bit (single-level cell or SLC); TLC is generally two to three times slower than SLC. A new version of the UFS specification is being written and turbo-write is expected to be part of it. The idea behind turbo-write is to use an SLC buffer to provide faster writes, with the contents being shifted to the slower TLC as needed. So Altman wondered when turbo-write mode should be used. 

[ ![\[Avri Altman\]](https://static.lwn.net/images/2019/lsf-altman-sm.jpg) ](<https://lwn.net/Articles/788848/>)

Ted Ts'o asked what is managing the blocks; does Linux need to copy the data from SLC to TLC? Altman said that it was transparent to the operating system; the device is managing the physical addresses and copies. Ts'o wondered what would happen if all writes were set to turbo. That would lead to endurance problems for the device, Altman said; sending every write request through the SLC will kill the flash. 

Damien Le Moal said that the developers need to understand about the wear-leveling done by the device in order to make real use of turbo mode. At some point, the device will have to ignore the a request for turbo-write, because the SLC is full or due to wear-leveling constraints, but without more information, the system cannot make the right decisions; the driver for the device is best placed to make those decisions. 

But Ts'o said that the kernel developers have to make a bunch of assumptions because the devices (and their makers) do not give the developers anything to work with. The impact of copying the data to TLC is not known, for example; will that affect read and write performance while it is happening? There are lots of unknowns, presumably devices will have different ratios of SLC to TLC, which would have an effect on what those decisions should be. 

Altman said that the amount of SLC available can be queried, but wondered if there is a policy that would make sense even without that information. Le Moal reiterated that more is needed beyond just the SLC capacity; in particular, information about wear-leveling will be needed. But applications will just treat wear-leveling as somebody else's problem, James Bottomley said. No application is going to go slow if the only tradeoff is wear-leveling for all of the applications using the device. Ts'o said that the simplest thing would be to make all synchronous writes be turbo and all background writes done in the normal mode; it may mean that the device will only last three months, however. 

Le Moal argued that the driver is the right place to make the turbo-write decision; it sees all the traffic, from that it can determine the right course. But Ewan Milne said that the decision should be pushed even lower: into the drive itself. This SLC/TLC split is meant as a performance enhancement for high-capacity devices. The device itself has the most information about its state; the question in his mind is what the kernel developers could even do to help. But Ts'o pointed out that the drive does not know if something is waiting for the write to complete, while the kernel can (and does) differentiate synchronous writes. 

Bottomley asked what happens when the SLC portion of the drive fails; does the whole device fail or does it just degrade? Altman said that it does degrade, so Bottomley thought that the kernel could just set turbo mode for all writes and it would be a fast device for a while, then turn into a slower one. Ts'o said that these flash chips are targeting mobile devices, so if it goes slow after three months or something, the mobile-device makers will not care because the reviewers will never test them for that long. 

In the end, telling the drive that a write is a turbo-write is simply a hint, the drive needs to make the decision, Le Moal said; it is like I/O priority. But Martin Petersen said he wanted to get up on his soapbox to point out that hinting and I/O priority have failed; they are an "awful, awful way" to convey to the device what it is you want it to do, he said. Indicating metadata or transaction journal writes is something the device can actually use, but relative priority has always been broken. 

Chris Mason said that from a practical point of view, the real problem is that there is no success criteria. His suggestion in the short term is to wire up some of these ideas, define what success is, and then debate various approaches based on that. 

But Ts'o said that the problem is not as bad as for generic SCSI devices, since UFS is only going to be used for mobile devices. Christoph Hellwig cautioned that "I wish that were true", but there are other classes of hardware where UFS is being considered—though probably not for laptops, he conceded. The point is that UFS devices will not be hosting Oracle enterprise databases or the like, Ts'o said, so the device interaction can be tuned for mobile-style workloads. 

Ts'o said that kernel developers are nervous about wiring things up in a highly application-specific way, however. The handset vendors are going to be driven by the device benchmarks, which do not take into account things like device health and endurance. There are various hints that can be given to the driver; it is up to the driver or the device to make use of them, Bottomley said. So, Altman concluded, the UFS device driver is the central place to make the decisions. 

Bottomley suggested that the driver look at the synchronous bit and turn on turbo mode for those writes, then benchmark the results to see how well it works. Ts'o noted that ext4 journal writes are marked synchronous, which could be used. The bigger issue is how to benchmark these changes, there is a need for some kind of internal measure on how the SLC is being affected by various choices. Bottomley said that existing hints could be used for now and if there are others that work better, they could be added to the kernel, but only in a data-driven way. 

Altman also wanted to discuss policies on when the SLC buffer contents should be moved to TLC. Ts'o suggested maybe flushing more aggressively when the device is connected to a power source, when the drive is idle would be another criteria, but the flushing decision also depends on how full the SLC buffer is—those are all things that the driver or device should know. As with the turbo-write policy, the plan should be to prototype it and if it needs more kernel infrastructure to work, then request it at that point. 

To sum up, Altman said, both the turbo-write governance and the evacuation policy should be handled by the UFS driver. Ts'o agreed, noting that the mobile-storage community has traditionally been resistant to putting more smarts in the devices; if that were not the case, one could imagine other engineering solutions, such as well-defined flush policies that the kernel could choose from.
