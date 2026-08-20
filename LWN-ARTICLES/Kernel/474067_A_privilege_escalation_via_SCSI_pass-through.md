---
title: "A privilege escalation via SCSI pass-through"
url: https://lwn.net/Articles/474067/
date: "January 4, 2012"
category: "SCSI-Command filtering"
author: "By Jake Edge January 4, 2012"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
January 4, 2012

One of the important attributes for virtualization is to provide complete isolation between the virtual machines, so that attackers (or bugs) in one VM cannot interfere with the other VMs. But, as a recent [bug report](<https://bugzilla.redhat.com/show_bug.cgi?id=752375>) shows, the kernel is vulnerable, in some configurations, to VMs that can read and write the disks of other VMs. That's clearly a serious security problem, but the discussion about patches to fix the bug makes it clear that it may take some time before the fix can be applied. 

The problem occurs when programs issue the SCSI pass-through `SG_IO` `ioctl()` to a particular disk partition (e.g. `/dev/sdb2`) or LVM volume, which causes the SCSI command to be sent to underlying block device (`/dev/sdb`). The actual commands that can be sent to the device via SG_IO are filtered for processes that don't have the `CAP_SYS_RAWIO` capability, but there are still dangerous things that can be done. In particular, if a process can write to the partition, it can write to the underlying device without being restricted to the boundaries of that partition. 

For virtualization configurations that mingle partitions or volumes used by different VMs on the same block device, that means that a VM can access--and change--the data on another VM's disk. Worse still, if the host OS stores its own data on that block device, a rogue VM could potentially compromise the host. Exploiting the vulnerability does not require a virtualization (or containerization) scenario, but those are the most likely ways that it could come about. Any process that can open the partition device node will be able to issue the `ioctl()`, but, on "standard" Linux systems, that ability is typically restricted to root. 

Based on the bug report, Paolo Bonzini found the problem back in November 2011, but security problems with `SG_IO` were [known](<https://lkml.org/lkml/2004/8/12/88>) as far back as August 2004. Bonzini posted [patches](<https://lwn.net/Articles/474068/>) to fix the problem at the end of December (though it would appear that the issue was under discussion on the closed kernel security mailing list in the interim). The proposed fix would disallow most SCSI commands on partition-like devices. So, doing any of the "dangerous" SCSI commands would fail unless the `ioctl()` is being called on the underlying block device. 

The patches sparked a few comments from Linus Torvalds, mostly regarding error return codes (partly because `ENOTTY` is badly named for its use as an indication of "no such ioctl"). But, beyond that, he started to wonder whether there might be situations where users _do_ issue SCSI commands to partitions and expect them to be passed down to the block device. It turns out that there is at least one place where it may be a common event: "ejecting" USB sticks and other removable media. Torvalds [notes](<https://lwn.net/Articles/474110/>): 

For example, I just traced it, and "eject /dev/sdb1" does a CDROMEJECT ioctl when used as the root user. I haven't tested the patch, but just reading it, I'd expect it to break that. 

And that's the *natural* way to eject a mounted device. Look at the USB memory sticks you have. They are almost all partitioned to have one partition, and that one partition doesn't cover the whole device. And it's that one partition you use to interact with it - it's what you mount, and what you eject. 

According to [Bonzini](<https://lwn.net/Articles/474112/>), the fact that the `CDROMEJECT` fails on a kernel with his proposed fix doesn't cause any problems in practice. But Torvalds's concern goes beyond that one particular example. The fix has been suggested for merging late in the 3.2 development cycle and his concern was the level of testing that it has been subjected to: ""I absolutely do not get the feeling that this has been tested so much and is so obvious that there is no risk of breakage."" Based on the discussion, the testing seems to have been focused on ensuring that the security hole was closed, without considering the other impacts that a--fairly sweeping--change might have. 

Torvalds would certainly like to see the vulnerability fixed, but not at the expense of a regression in what users have come to depend on. As he [pointed out](<https://lwn.net/Articles/474113/>): ""Suddenly totally changing things and saying 'you can't do that on a partition' when clearly people *have* been doing that on partitions isn't something we can do without serious testing."" His plan is to wait for the 3.3 merge window to bring in the fix, which should allow some testing time for distributions and others to ensure that the code doesn't have any unintended consequences. 

While it is important to fix security holes, it is equally important to keep everything else working, which is the bulk of Torvalds's concern. While the 3.3 development cycle may _still_ not be long enough to shake out all of the places where the SCSI pass-through is used on partial disks (partitions or logical volumes), it certainly will provide more of a chance to do so than would a merge in the final stages of 3.2 development. In the meantime, now that the bug and fix are out in the open, concerned administrators can apply the patch or take other steps to remedy the problem.
