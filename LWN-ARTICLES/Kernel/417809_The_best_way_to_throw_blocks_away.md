---
title: The best way to throw blocks away
url: https://lwn.net/Articles/417809/
date: "December 1, 2010"
category: "Block layer-Discard operations; Solid-state storage devices"
author: "By Jonathan Corbet December 1, 2010"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
December 1, 2010

An old-style rotating disk drive does not really care if any specific block contains useful data or not. Every block sits in its assigned spot (in a logical sense, at least), and the knowledge that the operating system does not care about the contents of any particular block is not something the drive can act upon in any way. More recent storage devices are different, though; they can - in theory, at least - optimize their behavior if they know which blocks are actually worth hanging onto. Linux has a couple of mechanisms for communicating that knowledge to the block layer - one added for 2.6.37 - but it's still not clear which of those is best. 

So when might a block device want to know about blocks that the host system no longer cares about? The answer is: just about any time that there is a significant mapping layer between the host's view of the device and the true underlying medium. One example is solid-state storage devices (SSDs). These devices must carefully shuffle data around to spread erase cycles across the media; otherwise, the device will almost certainly fail prematurely. If an SSD knows which blocks the system actually cares about, it can avoid copying the others and make the best use of each erase cycle. 

A related technology is "thin provisioning," where a storage array claims to be much larger than it really is. When the installed storage fills, the device can gently suggest that the operator install more drives, conveniently available from the array's vendor. In the absence of knowledge about disused blocks, the array must assume that every block that has ever been written to contains useful data. That approach may sell more drives in the short term, but vendors who want their customers to be happy in the long term might want to be a bit smarter about space management. 

Regardless of the type of any specific device, it cannot know about uninteresting blocks unless the operating system tells it. The ATA and SCSI standard committees have duly specified operations for communicating this formation; those operations are often called "trim" or "discard" at the operating system level. Linux has had support for trim operations for some time in the block layer; a few filesystems (and the swap code) have also been modified to send down trim commands when space is freed up. So Linux should be in good shape when it comes to trim support. 

The only problem is that on-the-fly trim (also called "online discard") doesn't work that well. On some devices, it slows operation considerably; there's also been some claims that excessive trimming can, itself, shorten drive life. The fact that the SATA version of trim is a non-queued operation (so all other I/O must be stopped before a trim can be sent to the drive) is also extremely unhelpful. The observed problems have been so widespread that SCSI maintainer James Bottomley was recently [heard](<https://lwn.net/Articles/417818/>) to say: 

However, I think it's time to question whether we actually still want to allow online discard at all. Most of the benchmarks show it to be a net lose to almost everything (either SSD or Thinly Provisioned arrays), so it's become an "enable this to degrade performance" option with no upside. 

The alternative is "batch discard," where a trim operation is used to mark large chunks of the device unused in a single operation. Batch discard operations could be run from the filesystem code; they could also run periodically from user space. Using batch discard to run trim on every free space extent would be a logical thing to do after an fsck run as well. Batching discard operations implies that the drive does not know immediately when space becomes unused, but it should be a more performance- and drive-friendly way to do things. 

The 2.6.37 includes a new `ioctl()` command called `FITRIM` which is intended for batch discard operations. The parameter to FITRIM is a structure describing the region to be marked: 

```
struct fstrim_range {
    	uint64_t start;
    	uint64_t len;
    	uint64_t minlen;
        };
```

An `ioctl(FITRIM)` call will instruct the filesystem that the free space between `start` and `start+len-1` (in bytes) should be marked as unused. Any extent less than `minlen` bytes will be ignored in this process. The operation can be run over the entire device by setting `start` to zero and `len` to `ULLONG_MAX`. It's worth repeating that this command is implemented by the filesystem, so only the space known by the filesystem to be free will actually be trimmed. In 2.6.37, it appears that only ext4 will have `FITRIM` support, but other filesystems will certainly get that support in time. 

Batch discard using `FITRIM` should address the problems seen with online discard - it can be applied to large chunks of space, at a time which is convenient for users of the system. So it may be tempting to just give up on online discard. But Chris Mason [cautions against](<https://lwn.net/Articles/417831/>) doing that: 

At any rate, I definitely think both the online trim and the FITRIM have their uses. One thing that has burnt us in the past is coding too much for the performance of the current crop of ssds when the next crop ends up making our optimizations useless. This is the main reason I think the online trim is going to be better and better. 

So the kernel developers will probably not trim online discard support at this time. No filesystem enables it by default, though, and that seems unlikely to change. But if, at some future time, implementations of the trim operation improve, Linux should be ready to use them.
