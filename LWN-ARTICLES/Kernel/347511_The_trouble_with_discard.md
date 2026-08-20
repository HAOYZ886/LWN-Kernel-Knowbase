---
title: The trouble with discard
url: https://lwn.net/Articles/347511/
date: "August 18, 2009"
category: "Block layer-Discard operations; Solid-state storage devices"
author: "By Jonathan Corbet August 18, 2009"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
August 18, 2009

Traditionally, storage devices have managed the blocks of data given to them without being concerned about how the system used those blocks. Increasingly, though, there is value in making more information available to storage devices; in particular, there can be advantages to telling the device when specific blocks no longer contain data of interest to the host system. The "discard" concept was [added to the kernel](<http://lwn.net/Articles/293658/>) one year ago to communicate this information to storage devices. One year later, it seems that the original discard idea will not survive contact with real hardware - especially solid-state storage devices. 

There are a number of use cases for the discard functionality. Large, "enterprise-class" storage arrays can implement virtual devices with a much larger storage capacity than is actually installed in the cabinet; these arrays can use information about unneeded blocks to reuse the physical storage for other data. The [compcache](<http://lwn.net/Articles/334649/>) compressed in-memory swapping mechanism needs to know when specific swap slots are no longer needed to be able to free the memory used for those slots. Arguably, the strongest pressure driving the discard concept comes from solid-state storage devices (SSDs). These devices must move data around on the underlying flash storage to implement their wear-leveling algorithms. In the absence of discard-like functionality, an SSD will end up shuffling around data that the host system has long since stopped caring about; telling the device about unneeded blocks should result in better performance. 

The sad truth of the matter, though, is that this improved performance does not actually happen on SSDs. There are two reasons for this: 

  * At the ATA protocol level, a discard request is implemented by a "TRIM" command sent to the device. For reasons unknown to your editor, the protocol committee designed TRIM as a non-queued command. That means that, before sending a TRIM command to the device, the block layer must first wait for all outstanding I/O operations on that device to complete; no further operations can be started until the TRIM command completes. So every TRIM operation stalls the request queue. Even if TRIM were completely free, its non-queued nature would impose a significant I/O performance cost. (It's worth noting that the SCSI equivalent to TRIM is a tagged command which doesn't suffer from this problem). 

  * With current SSDs, TRIM appears to be anything but free. Mark Lord has [measured](<https://lwn.net/Articles/347514/>) regular delays of hundreds of milliseconds. Delays on that scale would be most unwelcome on a rotating storage device. On an SSD, hundred-millisecond latencies are simply intolerable. 

One would assume that the second problem will eventually go away as the firmware running in SSDs gets smarter. But the first problem can only be fixed by changing the protocol specification, so any possible fix would be years in the future. It's a fact of life that we will simply have to live with. 

There are a few proposals out there for how we might live with the performance problems associated with discard operations. Matthew Wilcox has [a plan](<https://lwn.net/Articles/347516/>) to reimplement the whole discard concept using a cache in the block layer. Rather than sending discard operations directly to the device, the block layer will remember them in its own cache. Any new write operations will then be compared against the discard cache; whenever an operation overwrites a sector marked for discard, the block layer will know that the discard operation is no longer necessary and can, itself, be discarded. That, by itself, would reduce the number of TRIM operations which must be sent to the device. But if the kernel can work to increase locality on block devices, performance should improve even more. One relatively easy-to-implement example would be actively reusing recently-emptied swap slots instead of scattering swapped pages across the swap device. As Matthew puts it: ""there's a better way for the drive to find out that the contents of a block no longer matter -- write some new data to it."" 

In Matthew's scheme, the block layer would occasionally flush the discard cache, sending the actual operations to the device. The caching should allow the coalescing of many operations, further improving performance. Greg Freemyer, instead, [suggests](<https://lwn.net/Articles/347522/>) that flushing the discard cache could be done by a user-space process. Greg says: 

Assuming we have a persistent bitmap in place, have a background scanner that kicks in when the cpu / disk is idle. It just continuously scans the bitmap looking for contiguous blocks of unused sectors. Each time it finds one, it sends the largest possible unmap down the block stack and eventually to the device. 

When normal cpu / disk activity kicks in, this process goes to sleep. 

A variant of this approach was posted by Christoph Hellwig, who has implemented [batched discard support in XFS](<http://lwn.net/Articles/347330/>). Christoph's patch adds a new `ioctl()` which wanders through the filesystem's free-space map and issues large discard operations on each of the free extents. The advantage of doing things at the filesystem level is that the filesystem already knows which blocks are uninteresting; there is no additional accounting required to obtain that information. This approach will also naturally generate large operations; larger discards tend to suit the needs of the hardware better. On the other hand, regularly discarding all of the free space in a filesystem makes it likely that some time will be spent telling the device to discard sectors which it already knows to be free. 

It is far too soon to hazard a guess as to which of these approaches - if any - will be merged into the mainline. There is a fair amount of coding and benchmarking work to be done still. But it is clear that the code which is in current mainline kernels is not up to the task of getting the best performance out of near-future hardware. 

Your editor feels the need to point out one possibly-overlooked aspect of this problem. An SSD is not just a dumb storage device; it is, instead, a reasonably powerful computer in its own right, running complex software, and connected via what is, essentially, a high-speed, point-to-point network. Some of the more enterprise-oriented devices are more explicitly organized this way; they are separate boxes which hook into an IP-based local net. Increasingly, the value in these devices is not in the relatively mundane pile of flash storage found inside; it's in the clever firmware which causes the device to look like a traditional disk and, one hopes, causes it to perform well. Competition in this area has brought about some improvements in this firmware, but we should see a modern SSD for what it is: a computer running proprietary software that we put at the core of our systems. 

It does not have to be that way; Linux does not need to talk to flash storage through a fancy translation layer. We have our own translation layer code (UBI), and a few filesystems which can work with bare flash. It would be most interesting to see what would happen if some manufacturer were to make competitive, bare-flash devices available as separate components. The kernel could then take over the flash management task, and our developers could turn their attention toward solving the problem correctly instead of working around problems in vendor solutions. Kernel developers made an explicit choice to avoid offloading much of the network stack onto interface hardware; it would be nice to have a similar choice regarding the offloading of low-level storage management. 

In the absence of that choice, we'll have no option but to deal with the translation layers offered by hardware vendors. The results look unlikely to be optimal in the near future, but they should still end up being better than what we have today.
