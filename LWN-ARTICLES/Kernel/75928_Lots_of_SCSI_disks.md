---
title: Lots of SCSI disks
url: https://lwn.net/Articles/75928/
date: "March 16, 2004"
category: Device naming; SCSI
author: 
---

One of the motivations for increasing the size of the `dev_t` device number type in 2.6 was to allow the use of huge numbers of SCSI disks. In the 2.6.4 kernel, however, that promise remains unfulfilled; the SCSI subsystem makes no use of the expanded device number range. That will change in 2.6.5, however; a patch has been merged which allows the enumeration of up to 1 million SCSI disks. 

The authors of this patch had an interesting problem to solve: they wanted to be able to enumerate all of those disks without breaking existing systems. In other words, all of the existing SCSI device numbers have to work as they do in 2.4 and prior kernels. The solution is expressed in the following macro, which turns a device index (the "nth disk") and a partition number into its associated device number: 

```
static unsigned int make_sd_dev(unsigned int sd_nr, unsigned int part)
    {
    	return  (part & 0xf) | ((sd_nr & 0xf) << 4) |
    		(sd_major((sd_nr & 0xf0) >> 4) << 20) | (sd_nr & 0xfff00);
    }
```

LWN readers will, no doubt, immediately understand what is going on here. Your editor, however, had to stare at it for a little while. Then, as a way of avoiding doing real work, he made the following diagram to show how a device index and partition number are transmogrified into a device number. 

![\[SCSI numbering diagram\]](https://static.lwn.net/images/ns/scsi-numbers.png)

The "remap" operation takes four bits from the device index and uses them to index into an array of the 16 major numbers which have been assigned for some time to SCSI disks: 8, 65-71, and 128-135. The lowest four bits of the device index move directly down into the minor number. The result is that the first 256 SCSI disks will get exactly the same major and minor numbers that they have in 2.4 kernels. 

Once that space has been exhausted, however, the four red bits in the diagram will return to zero, the major number will go back to 8, the highest-order bits in the device index are routed back into the minor number, and, as a result, the 257th disk will be given device number `8:256`. The 273rd disk will advance again to the next major number; it will be given number `65:256`. Additional disks will be distributed across the available major numbers indefinitely until their combined power load flips a breaker somewhere. 

The result is a scheme which might be a little hard for humans to follow, but, when you are dealing with thousands of disks, that will be the case anyway. Meanwhile, most of the main design goals - support lots of disks without breaking existing systems - have been met. There is one remaining issue, however: some SCSI users have been asking for the ability to have more than 15 partitions on one drive. Supporting a larger partition space and simultaneously preserving compatibility is not currently possible because the block layer expects partitions to be assigned contiguous minor numbers. Fixing that will require tweaks to the gendisk code.
