---
title: SCSI command filtering
url: https://lwn.net/Articles/193516/
date: "July 31, 2006"
category: "CD recording; SCSI-Command filtering; Security-SCSI command filtering"
author: 
---

Burning data to a CD or DVD is a complicated task, involving the use of a wide range of SCSI commands. So, any application which burns discs must have the ability to send special SCSI operations to the drive. Just before the 2.6.8 kernel came out, however, the kernel developers decided that applications should not be able to send just _any_ SCSI command. Some of those commands could lead the drive to rewrite its firmware, catch fire, or replace music tracks with recordings of Richard Stallman singing. In an attempt to keep such undesirable things from happening, Linus [added a late patch](<http://lwn.net/Articles/98379/>) which blocked unprivileged users from using any SCSI commands which do not appear in an in-kernel whitelist. 

It is almost certainly true that no user ever destroyed a CD drive with a 2.6.8 system. In fact, very few of them even wrote discs; the filtering at that stage was so severe that unprivileged users could not do anything useful at all. Subsequent updates made things better, however, and by about 2.6.10 burning worked again for most users. 

Not for all users, however. As Dave Jones recently [noted](<https://lwn.net/Articles/193518/>) on the linux-scsi list, the command filtering still trips up some Plextor drives. The cdrecord utility tries to send vendor-specific commands to those drives, but the kernel filters them out. Everything then comes to a halt, and the user must retry the operation as root to get the job done. Dave asked: might it be a good idea to add a per-vendor exceptions capability to the filtering code? 

The response which came back from a couple of block subsystem developers was that the command filtering should simply be taken out altogether. Evidently this topic had been discussed at the recent storage summit, and the participants had agreed that this feature should be removed. James Bottomley [put it this way](<https://lwn.net/Articles/193519/>): 

If we're going to allow users access to burn CDs, it's impossible to police them with certainty as this case indicates. If we allow vendor specific commands down, there are bound to be some that format the drive or destroy the firmware... 

So I think ripping the table out and acknowledging we have no security is better than giving the illusion of having it. 

There are a number of complaints about the filtering code. It is a way of encoding policy in the kernel, which is generally frowned upon - even though the policy, in this case, is really an attempt to enforce a difference between access to a disc within a drive and access to the drive itself. The command list will never be entirely correct; it seems that some drives must receive the appropriate, vendor-specific incantations or they will refuse to write discs. Some commands mean different things to different types of devices; what's safe for a CD burner might be a destructive operation on a different SCSI-like device. It also doesn't help that there are, in fact, two different SCSI command filters in the kernel (one in `drivers/scsi/sg.c`, the other in `block/scsi_ioctl.c`) which implement different policies. For all of these reasons, attendees at the storage summit apparently agreed to take the filtering out. 

There's just one little problem with this plan: Linus [feels differently about filtering](<https://lwn.net/Articles/193526/>): 

Put another way: you will remove that command filtering in block/scsi_ioctl.c only in a kernel that I don't maintain, or by disabling it in some way that is so hidden that I won't notice. Because I'm not so stupid as to think that it's ok for normal users to set driver passwords or rewrite the disk firmware just because they have write permissions to the device. That's pretty damn final. 

This statement would appear to be pretty damn final. That does not mean that the situation cannot be improved, however. The leading idea at the moment would appear to be to allow a privileged user to make changes to the command filter table. Distributions could then ship tools which detect problematic devices and modify the filtering tables accordingly; the whole thing could be transparently integrated with the hotplug functionality. Jens Axboe has [a patch](<http://git.kernel.dk/?p=linux-2.6-block.git;a=commitdiff;h=63bfd0060d0feb5f3ba161c2c1e6e8082fe2f68d>) (originally from Peter Jones) which turns the filter list into a per-device object, tweakable through sysfs, so each device could have its own set of exceptions. 

Just how this interface works may yet require some discussion to nail down. But the configurable, per-device filter looks like the way forward. It retains the filtering of dangerous commands while moving the policy decisions to user space. Once the policy can be changed, distributors can do the work to ensure that specific devices are well supported, or, if they prefer, simply mark all commands as "allowed" and, for all practical purposes, remove the filter altogether.
