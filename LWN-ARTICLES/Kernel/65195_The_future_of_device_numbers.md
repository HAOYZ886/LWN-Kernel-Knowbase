---
title: The future of device numbers
url: https://lwn.net/Articles/65195/
date: "January 6, 2004"
category: Device numbers; udev
author: 
---

Greg Kroah-Hartman has, it seems, received a fair amount of email from devfs users, many of whom are not pleased with the fact that devfs has been marked "deprecated" in 2.6. Never mind that Greg didn't do that... But Greg _is_ the primary author of udev, which is intended to replace devfs in the future. With the intent of cutting down on hate mail, Greg has posted [a lengthy diatribe](<https://lwn.net/Articles/65197/>) on why, he thinks, the udev approach is better. It's not at all clear that his posting will have succeeded in that goal, but it does make the current thinking (accepted by most kernel developers, it seems) clearer. 

The posting also inspired a lengthy thread on the meaning of Linux device numbers and how they will be handled in the future. For starters, we now have [Linus's explanation](<https://lwn.net/Articles/65200/>) of why he chose to expand the device number type to 32 bits, rather than the expected 64: 

Note that one reason I didn't much like the 64-bit versions is that not only are they bigger, they also encourage insanity. Ie you'd find SCSI people who want to try to encode device/controller/bus/target/lun info into the device number. 

We should resist any effort that makes the numbers "mean" something. They are random cookies. Not "unique identifiers", and not "addresses". 

Linus's talk of "random cookies" set off some alarms from developers who foresee a world where devices could have different numbers every time the system boots. Linus's response was unrepentant; he claims that (1) that world already exists, and (2) attempts to create relatively stable device numbers just encourage applications to depend on those numbers not changing, and thus create bugs. 

Anybody who has plugged two similar USB devices into the same system has already experienced one kind of device number instability. The kernel will assign numbers based on the order in which it discovers the devices; that order depends on a number of things, including, simply, which device was plugged in first. There is no way in the general case to provide stable numbers for this sort of hot-pluggable device. Other devices, such as iSCSI disks, are even worse. Discovering all of the available devices can be a challenge by itself; there is no way that this discovery will happen in a predictable order. 

So, for many kinds of devices, variable device numbers is simply a fact of life. So, [says Linus](<https://lwn.net/Articles/65203/>), it is better not to even try to keep numbers stable. 

Basically, if you cannot 100% guarantee reproducibility (and nobody can, not your hashes, not anything else), then the _appearance_ of reproducibility is literally a mistake. Because it ends up being a bug waiting to happen - and one that is very very hard to reproduce on a developer machine. 

To bring that point home, Linus has raised an idea that Greg has presented a few times in the past: making all device numbers random. This change would quickly flush out any code which made assumptions about device numbers, whether it be in the kernel or in user space. Of course, random device number assignment is a feature for a development kernel; Linus [acknowledges](<https://lwn.net/Articles/65209/>) that, "for simple politeness reasons," device numbers should be kept as stable as possible in stable kernel releases. 

In any case, the point of all this is not to confuse users about the organization of their system. But, in a world where device numbers can offer no real clues about the hardware on a computer, something else needs to create stable names by which devices can be identified. That, of course, is the purpose of tools like udev. As a way of showing how flexible udev can be, Greg posted [a brief script](<https://lwn.net/Articles/65213/>) which makes CD drives available by the name of the disk (as obtained from CDDB) currently inside. This scheme is unlikely to become part of any major distribution in the near future, but it does show how elaborate device naming can be. For some sorts of devices, a conversation with a remote server may well be part of the naming process. As naming gets more complex, it becomes increasingly clear that it simply cannot be done in the kernel. 

That, of course, is one of the main objections to devfs - the naming policy is implemented entirely in kernel space. The udev approach moves that policy back out to user space, where it can be easily changed and extended. The remaining devfs users will want to look at switching over, but there is no particular hurry; Andrew Morton has [made it clear](<https://lwn.net/Articles/65216/>) that devfs will continue to be supported through the lifetime of 2.6 and, possibly, beyond.
