---
title: The 2006 Linux File Systems Workshop
url: https://lwn.net/Articles/190222/
date: "July 5, 2006"
category: "File Systems Workshop-2006"
author: "July 5, 2006 This article was contributed by Valerie Aurora"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

July 5, 2006

This article was contributed by Valerie Aurora

The Linux file systems community met in Portland in June 2006 to discuss the next 5 years of file system development in Linux. Organized by [Val Henson](<http://www.nmt.edu/~val>), Zach Brown, and Arjan van de Ven, and sponsored by [Intel](<http://www.intel.com/software/opensource>), [Google](<http://code.google.com/>), [Oracle](<http://oss.oracle.com>), the [Linux File Systems Workshop](<http://www.nmt.edu/~val/fs_workshop/>) brought together thirteen Linux file systems developers and experts to share data and brainstorm for three days. Our goal was to discuss the direction of Linux file systems development during the next 5 years, with a focus on disruptive technologies rather than incremental improvements. Our goal was not to design one new file system to rule them all, but to come up with several useful new file system architecture ideas (which may or may not reuse existing file system code). To stay focused, we explicitly ruled out discussion of the design of distributed or clustered file systems, with the exception of how they impact local file system design. We came out of the workshop with broad agreement on the problems facing Linux file systems, several exciting file system architecture ideas, and a commitment to working together on the next generation of Linux file systems. 

### The Problem

Why do we need a Linux file systems workshop, when all seems well in Linux file systems land? Disks purr gently along, larger and fatter than ever before, but still essentially the same. I/O errors are an endangered species, more rumor than fact, and easily corrected with a simple fsck. The "df" command returns a comforting 50% free on most of your file systems. You chuckle gently as you read old file system man pages with directions for tuning inode/block ratios. Sure, that 32-bit file system size limit is looming somewhere over the horizon, but a quick patch to change the size of your block pointers is all you need and you'll be back in business again. After all, file systems are a solved problem, right? Right? 

If computer hardware never changed, we kernel developers would have nothing better to do than argue about the optimal scheduling algorithm and flame each others' coding style. Unfortunately, hardware has this terrible habit of changing frequently, drastically, and worst of all, exponentially. File systems are especially vulnerable to changes in hardware because of their long-lived nature. Much of operating systems software can be changed at will given a simple system reboot. But file systems - and their on-disk data layouts - live on and on. 

What has changed in hardware that affects file systems? Let's start with some simple, unavoidable facts about the way disks are evolving. Everyone knows that disk capacity is growing exponentially, doubling every 9-18 months. But what about disk bandwidth and seek time? At the last [Storage Networking World conference](<http://www.snwusa.com/>), Seagate presented some details of their hard disk road map for the next 7 years (see page 16 of [the slides [PDF]](<http://www.snwusa.com/documents/presentations-s06/MarkKryder.pdf>)). Their predictions for 3.5 inch hard disks are summarized in the following table. 

> Parameter| 2006| 2009| 2013| Improvement  
> ---|---|---|---|---  
> Capacity (GB)| 500| 2000| 8000| 16x  
> Bandwidth (Mb/s)| 1000| 2000| 5000| 5x  
> Read seek time (ms)| 8| 7.2| 6.5| 1.2x  
  
In summary, over the next 7 years, disk capacity will increase by 16 times, while disk bandwidth will increase only 5 times, and seek time will barely budge! Today it takes a theoretical minimum 4,000 seconds, or about 1 hour to read an entire disk sequentially (in reality, it's longer due to a variety of factors). In 2013, it will take a minimum of 12,800 seconds, or about 3.5 hours, to read an entire disk - an increase of 3 times. Random I/O workloads are even worse, since seek times are nearly flat. A workload that reads, e.g., 10% of the disk non-sequentially will take much longer on our 8TB 2013-era disk than it did on our 500GB 2006-era disk. 

Another interesting change in hardware is the rate of increase in capacity versus the rate of reduction in I/O errors per bit. In order for a disk to have the same overall number of I/O errors, every time capacity doubles, the per-bit I/O error rate must halve. Needless to say, this isn't happening, so I/O errors are actually more common even though the per-bit error rate has dropped. 

These are only a few of the changes in disk hardware that will occur over the next decade. What do these changes mean for file systems? First, fsck will take a lot longer in absolute terms, because disk capacity is larger, but disk bandwidth is relatively smaller, and seek time is relatively much larger. Fsck on multi-terabyte file systems today can easily take 2 days, and in the future it will take even longer! Second, the increasing number of I/O errors means that fsck is going to happen a lot more often - and journaling won't help. Existing file systems simply weren't designed with this kind of I/O error frequency in mind. 

These problems aren't theoretical - they are already affecting systems that you care about. Recently, the main server for Linux kernel source, kernel.org, suffered file system corruption from a failure at the RAID level. It took over a week for fsck to repair the (ext3) file system, when it would have taken far less time to restore from backup. 

### The workshop

Now that the stage is set, we'll move on to what happened at the 2006 Workshop. The coverage has been split into the following pages: 

  * [Day 1](<https://lwn.net/Articles/190223/>), devoted mostly to understand the current state of the art: file system repair, disk errors, lessons learned from existing file systems, and major filesystem architectures. 

  * [Days 2 and 3](<https://lwn.net/Articles/190224/>), concerned with the way forward: interesting ideas, near-term needs, and development plans.
