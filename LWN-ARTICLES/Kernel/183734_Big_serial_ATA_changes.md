---
title: Big serial ATA changes
url: https://lwn.net/Articles/183734/
date: "May 16, 2006"
category: Serial ATA
author: 
---

Jeff Garzik has recently [let it be known](<http://lwn.net/Articles/183685/>) that he has merged a large set of patches to the serial ATA (SATA) subsystem. Says Jeff: ""If all goes well, this update should improve error handling, solve several outstanding, difficult-to-solve bugs, and provide a good foundation for adding some nifty features in the future."" His plans are to get the new code merged into the 2.6.18 kernel, once that cycle begins. The result could be a significantly different experience for Linux SATA users, some of whom have been fighting problems for some time. 

The patches themselves have been posted to the linux-ide list. It makes for some imposing reading: they are 122 patches, divided into eleven sets. This flood of code is primarily the work of Tejun Heo, though Jens Axboe and Albert Lee have also played a significant part. In brief, what is coming is: 

  * A completely reworked libata error handler. This code makes up about a third of the total set of patches, and cleans up a lot of things. It creates a modularized error handling mechanism which allows low-level drivers to intervene or change the response at various points in the process. Memory needed for error handling is now allocated ahead of time, minimizing the possibility for complications just when things are already going wrong. There is a special circular buffer set aside for recording errors; this information is used, for example, within the recovery code to determine that the error rate is too high and that transmission speed should be lowered. 

The result of all this work should be a much more robust SATA subsystem which can recover from a much wider range of errors. 

  * A new programmed I/O loop which uses interrupts, rather than older method of polling the controller from a kernel thread. In cases where programmed I/O is needed, the new code should be more efficient. 

  * Native Command Queuing (NCQ). NCQ is the SATA version of tagged command queuing - the ability to have several I/O requests to the same drive outstanding at the same time. NCQ eliminates the idle time between when one command completes and the next is issued, but the real advantage is with the ordering of operations. The Linux block I/O subsystem attempts to issue block I/O requests in an efficient order, but it must use a certain amount of guessing, since there is no way to know how the blocks are really organized on the disk. But the drive itself knows very well where each block lives, so it is well placed to optimize the ordering of requests. The result can be a significant improvement in performance. 

The Linux NCQ implementation can have up to 32 operations outstanding at any given time - though both the drive and the host controller can reduce that number. Your editor is not aware of any relative performance benchmarks which have been posted. 

  * Hotplug support is another large piece of the patch set. With these patches in place, the SATA layer can deal with drives which come and go - as long as the underlying hardware was designed with hotplugging in mind. There is also a "warmplug" capability for more limited hardware, where a system user can request the addition or removal of drives on a running system. 

  * A new layer (called "ata_link") has been added to libata; ata_link handles the physical-layer connection to the drives. The main motivation for ata_link appears to make it possible to support SATA [port multipliers](<http://www.sata-io.org/portmultiplier.asp>), which expand the number of drives which can be plugged into a system. The current port multiplier code supports the "frame information structure" switching mode, whereby all connected drives can be active simultaneously. For now, it only works with the sil24 driver, but support for others will certainly come. 

Most of this code has been under development and discussion for some time. The sense (among its developers) is that the bulk of it is ready to go into 2.6.18, though the hotplug, ata_link, and port multiplier code [may have to wait](<https://lwn.net/Articles/183756/>) for another cycle. Andrew Morton has [expressed some concerns](<https://lwn.net/Articles/183758/>) about merging all of this code when a rather long list of SATA-related bugs remains outstanding; Jeff [responded](<https://lwn.net/Articles/183759/>) that this code will fix many of the bugs and make tracking down many of the rest easier. So, chances are, 2.6.18 will include a much-improved SATA layer.
