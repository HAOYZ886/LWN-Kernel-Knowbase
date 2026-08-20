---
title: Parallel IDE drivers
url: https://lwn.net/Articles/198344/
date: "September 5, 2006"
category: Parallel ATA; libata
author: 
---

Back in 2003, Jeff Garzik [announced](<http://lwn.net/Articles/33624/>) the availability of "a new SCSI driver." That driver was, in fact, the libata subsystem, which was to be the foundation for serial ATA support in Linux. In the process, however, Jeff had thought a bit about supporting the current parallel ATA (PATA) drives, but that was not really his goal: 

Note that PATA in my driver is only an afterthought. The main area of focus, now and in the future, is SATA. 

In the last three years, the parallel ATA drives that most of us use have continued to be driven by the old IDE driver subsystem. Some of this code dates back to the beginning of Linux; since then it has been maintained by a substantial list of people, a number of whom are widely held to have been driven insane by the experience. The current maintainer, Bartlomiej Zolnierkiewicz, has kept a rather low profile for some time now; he signed off no patches in either of the 2.6.17 or upcoming 2.6.18 kernels. Not much has been happening in the IDE area. 

That does not mean that things have been quiet in the parallel ATA area, however. Over the last year or so, Alan Cox has been working to bring full PATA support into the libata code. The resulting drivers have been sitting in the -mm tree for a while, but that period is about to end: the PATA driver set [has been queued for merging into 2.6.19](<http://lwn.net/Articles/198168/>). 

The stated advantages of the new PATA code are many. The code has been reworked from the beginning, and is up to current kernel standards. The use of libata means that these drivers are well integrated with their SATA cousins, bringing two divergent subsystems back together. The new drivers support a number of chipsets that the IDE layer doesn't handle. Error handling has been much improved. Also, according to [Alan's announcement](<http://lwn.net/Articles/194866/>) from August, the new drivers feature "active maintenance and updates" and "more interesting bugs to find and help fix." 

On the other hand, the new PATA drivers are not considered to be ready for production use yet, and distributors are not expected to enable them in the near future. The merging into 2.6.19 is intended mainly to broaden the test base. A completely new disk subsystem is the sort of thing that one likes to test very well before entrusting it with data that one wishes to actually keep; that process may go on for a little while yet. It is also worth noting that the new PATA code also drops support for some ancient IDE controllers. 

The issue that gets everybody's attention, however, is that, as with all drives handled through libata, PATA drives show up as if they were SCSI disks, and are named `/dev/sd*`. Anybody who just switches to the new drivers without updating `/etc/fstab` (or using the mount-by-label feature) is likely to have a rough bootstrap experience. That is an easy problem to work around, but the use of the SCSI drive namespace seems to bother some people. What appears to be happening in reality is that Linux is slowly moving toward having a generic disk subsystem, where everything can just be called `/dev/diskN`. All that's left is a few details and a new set of udev rules to rename the device nodes. 

Someday, most of us will be using the new PATA code. But this is not a process which is expected to go quickly, and there are no plans to remove or deprecate the existing IDE code: 

At this point in time it is premature to discuss or plan the point at which the old IDE layer would go away. That discussion can start at the point where everyone is happy that the new libata based layer is providing better quality and coverage than the old one. Even then there would be no need to hurry. 

So it appears that Linux will have parallel subsystems for parallel ATA support for some time.
