---
title: "4K-sector drives and Linux"
url: https://lwn.net/Articles/377895/
date: "March 9, 2010"
category: "Block layer-Large physical sectors"
author: "By Jonathan Corbet March 9, 2010"
---

By **Jonathan Corbet**  
March 9, 2010

Almost exactly one year ago, LWN [examined the problem of 4K-sector drives](<http://lwn.net/Articles/322777/>) and the reasons for their existence. In short, going to 4KB physical sectors allows drive manufacturers to increase storage density, always welcome in that competitive market. Recently, there have been a number of reports that Linux is not ready to work with these drives; kernel developer Tejun Heo even [posted an extensive, worth-reading summary](<https://lwn.net/Articles/377897/>) stating that ""4 KiB logical sector support is broken in both the kernel and partitioners."" As the subsequent discussion revealed, though, the truth of the matter is that we're not quite that badly prepared. 

Linux is fully prepared for a change in the size of physical sectors on a storage device, and has been for a long time. The block layer was written with an avoidance of hardwired sector sizes in mind. Sector counts and offsets are indeed managed as 512-byte units at that level of the kernel, but the block layer is careful to perform all I/O in units of the correct size. So, one would hope, everything would Just Work. 

But, as Tejun's document notes, ""unfortunately, there are complications."" These complications result from the fact that the rest of the world is not prepared to deal with anything other than 512-byte sectors, starting with the BIOS found on almost all systems. In fact, a BIOS which can boot from a 4K-sector drive is an exceedingly rare item - if, indeed, it exists at all. Fixing the BIOS is evidently harder than one might think, and, evidently, there is little motivation to do so. Martin Petersen, who has done much of the work around supporting these drives in Linux, [noted](<https://lwn.net/Articles/377900/>): 

Part of the hesitation to work on booting off of 4 KB lbs drives is motivated by a general trend in the industry to move boot functionality to SSD. There are 4 KB LBS SSDs out there but in general the industry is sticking to ATA for local boot. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The problem does not just exist at the BIOS level: bootloaders (whether they are Linux-oriented or not) are not set up to handle larger sectors; neither are partitioning tools, not to mention a wide variety of other operating systems. Something must be done to enable 4K-sector drives to work with all of this software. 

That something, of course, is to interpose a mapping layer in the middle. So most 4K-sector drives will implement separate logical and physical sector sizes, with the logical size - the one presented to the host computer - remaining 512 bytes. The system can then pretend that it's dealing with the same kind of hardware it has always dealt with, and everything just works as desired. 

Except that, naturally enough, there are complications. A 512-byte sector written to a 4K-sector drive will now force the drive to perform a read-modify-write cycle to avoid losing the data in the rest of the sector. That slows things down, of course, and also increases the risk of data loss should something go wrong in the middle. To avoid this kind of problem, the operating system should do transfers that are a multiple of the physical sector size whenever possible. But, to do that, it must know the physical sector size. As it happens, that information has been made available; the kernel makes use of this information internally and exports it via sysfs. 

It is not quite that simple, though. The Linux kernel can go out of its way to use the physical sector size, and to align all transfers on 4KB boundaries from the beginning of the partition. But that goes badly wrong if the partition itself is not properly aligned; in this case, _every_ carefully-arranged 4KB block will overlap two physical sectors - hardly an optimal outcome. 

As it happens, badly-aligned partitions are not just common; they are the norm. Consider an example: your editor was a lucky recipient of an Intel solid-state drive at the Kernel Summit which was quickly plugged into his system and partitioned for use. It has been a great move: git repositories on an SSD are much nicer to work with. A quick look at the partition table, though, shows this: 

> 

```
>     Disk /dev/sda: 80.0 GB, 80026361856 bytes
>     255 heads, 63 sectors/track, 9729 cylinders, total 156301488 sectors
>     Units = sectors of 1 * 512 = 512 bytes
>     Sector size (logical/physical): 512 bytes / 512 bytes
>     I/O size (minimum/optimal): 512 bytes / 512 bytes
>     Disk identifier: 0x5361058c
>     
>        Device Boot      Start         End      Blocks   Id  System
>     /dev/sda1              63    52452224    26226081   83  Linux
>
```

Note that `fdisk`, despite having been taken out of the "DOS compatibility" mode, is displaying the drive dimensions in units of heads and cylinders. Needless to say, this device has neither; even on rotating media, those numbers are entirely fictional; they are a legacy from a dark time before Linux even existed. But that legacy is still making life difficult now. 

Once upon a time, it was determined that 63 (512-byte) sectors was far more than anybody would be able to fit into a single disk track. Since track-aligned I/O is faster on a rotating drive, it made sense to align partitions so that the data began at the beginning of a track. So, traditionally, the first partition on a drive begins at (logical) sector 63, the last sector of the first track. That sector holds the boot block; any filesystem stored on the partition will follow at the beginning of the next track. That placement, of course, misaligns the filesystem with regard to any physical sector size larger than 512 bytes; logical sector 64 (the first data sector in the partition) will be placed at the end of a 4K physical sector. Any subsequent partitions on the device will almost certainly be misaligned in the same way. 

One might argue that the right thing to do is to simply ditch this particular practice and align partitions properly; it should not be all that hard to teach partitioning tools about physical sector sizes. This can certainly be done. The tools have been slow to catch on, but a suitably motivated system administrator can usually convince them to place partitions sensibly even now. So weird alignments should not be an insurmountable problem. 

Unfortunately, there are complications. It would appear that Windows XP not only expects misaligned partitions; it actually will not function properly without them. One simply cannot run XP on a device which has been properly partitioned for 4K physical sector sizes. To cope with that, drive manufacturers have introduced an even worse hack: shifting all 512-byte logical sectors forward by one, so that logical sector 64 lands at the beginning of a physical sector. So any partitioning tool which wants to lay things out properly must know where the origin of the device actually is - and not all devices are entirely forthcoming with that information. 

With luck, the off-by-one problem will go away before it becomes a big issue. As [James Bottomley put it](<https://lwn.net/Articles/377909/>): ""...fortunately very few of these have been seen in the wild and we're hopeful they can be shot before they breed."" But that doesn't fix the problem with the alignment of partitions for use by XP. Later versions of Windows need not concern themselves with this problem, since they rarely coexist with XP (and Windows has never been greatly concerned about coexistence with other systems in general). Linux, though, may well be installed on the same drive as XP; that leads to differing alignment requirements for different partitions. Making _that_ just work is not going to be fun. 

Martin [suggests](<https://lwn.net/Articles/377910/>) that it might be best to just ignore the XP issue: 

With regards to XP compatibility I don't think we should go too much out of our way to accommodate it. XP has been disowned by its master and I think virtualization will take care of the rest. 

It may well be that there will not be a significant number of XP installations on new-generation storage devices, but failure to support XP may still create some misery in some quarters. 

A related issue pointed out by Tejun is that the DOS partition format, which is still widely used, tops out at 2TB, which just does not seem all that large anymore. Using 4K logical sectors in the partition table can extend that limit as far as 16TB, but, again, that requires cooperation from the BIOS - and it still does not seem all that large. The long-term solution would appear to be moving to a partition format like [GPT](<http://en.wikipedia.org/wiki/GUID_Partition_Table>), but that is not likely to be an easy migration. 

In summary: Linux is not all that badly placed to support 4K-sector drives, especially when there is no need to share a drive with older operating systems. There is still work required at the tools level to make that support work optimally without the need for low-level intervention by system administrators, but that is, as they say, just a matter of a bit of programming. As these drives become more widely available, we will be able to make good use of them.
