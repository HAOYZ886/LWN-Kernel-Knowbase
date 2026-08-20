---
title: Shooting at SCSI targets
url: https://lwn.net/Articles/420691/
date: "December 22, 2010"
category: SCSI
author: "By Jonathan Corbet December 22, 2010"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
December 22, 2010

The SCSI protocol normally specifies a two-sided conversation between an "initiator," which initiates requests, and a "target," which acts upon those requests. Normally, the initiator is the host computer, while the target is a storage device; it's thus not surprising that the bulk of the SCSI-related code in the kernel implements the initiator role. There are times, though, when it is useful to have a Linux system act as a SCSI target, usually when that system is the interface to some sort of complicated storage array. The target mode is currently implemented by the [STGT](<http://stgt.sourceforge.net/>) code, but that subsystem has been seen as being ripe for replacement for some time. 

There are two main contenders to replace STGT: [LIO](<http://www.linux-iscsi.org/index.php/Main_Page>) and [SCST](<http://scst.sourceforge.net/>). In the end, there's really only room in the kernel for one SCSI target implementation, so there naturally has been a fair amount of tension between these two projects. Whenever the discussion turned to choosing one, it tended toward the ugly side. SCSI maintainer James Bottomley has done his best to stay out of the flames, but, in the end, he must make a decision and merge one of them. 

A few months back, it began to become clear that LIO was going to be the winner. More recently, James [gave the green light](<https://lwn.net/Articles/420693/>) to begin merging this code for the 2.6.38 kernel. Suffice to say that SCST maintainer Vladislav Bolkhovitin [did not take the decision well](<https://lwn.net/Articles/420695/>) and did his best to restart the battle in a wider context. James has stuck with his decision, though, [saying](<https://lwn.net/Articles/420697/>) that there is not much to choose between technically, and that it came down to community: 

Or said a different way: as long as you choose the most community oriented of competing offerings, the community will fill any perceived gaps. Conversely, you can destroy a project simply by alienating the community. That's why community is more important than feature set. 

The previous discussions appear to have worn down most other participants, so few people chose to join in this time around. There doesn't seem to be anything to suggest that the decision will change at this point; unless something surprising happens, LIO will be the in-kernel SCSI target subsystem as of 2.6.38.
