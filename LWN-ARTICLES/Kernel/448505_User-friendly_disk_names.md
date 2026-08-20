---
title: "User-friendly disk names"
url: https://lwn.net/Articles/448505/
date: "June 22, 2011"
category: Device naming
author: "By Jake Edge June 22, 2011"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
June 22, 2011

Device names, particularly for disks, can be confusing to Linux administrators because they get assigned at boot time based on the order in which the disks are discovered. So the same physical disk can be assigned a different device name (in `/dev`) on each boot, which means that kernel log messages and the output of various utilities may not correspond with the administrator's view of the system. A recent [patch set](<https://lwn.net/Articles/447526/>) looks to change that situation, but it is meeting some resistance from kernel hackers who think it should be handled by user space. 

The patches posted by Nao Nishijima are pretty straightforward. They just add a `preferred_name` entry to `struct device`, which can be updated via sysfs. The patches then change some SCSI log messages and `/proc/partitions` output to use the preferred name if it has been set. Greg Kroah-Hartman expressed [concerns](<https://lwn.net/Articles/448652/>) about changing `/proc/partitions` as various tools parse that file so it is part of the kernel's user-space interface. Adding the preferred name as a new field on each line might very well confuse utilities that parse the file. 

More importantly, though, he notes that one could just change the tools so that they use the names as arguments or in their output. Any scheme that would map preferred names to specific disks requires some kind of mapping file, so tools that wanted to use these preferred names (things like `mount`, `smartd`, and other disk-related tools) could do so using that mapping without involving the kernel at all: 

Seriously, this could be done by now, it's been over a year since this was first discussed. All distros could have the updated packages by now and this would not be an issue. 

I still think this is the correct way to solve the problem as it is a userspace issue, not a kernel one. 

While the patches only use `preferred_name` for disk devices, the idea is to allow them to be added to any device (and then change log messages and utilities to use them). It is modeled after the `ifalias` entry that was added for network devices back in 2008, but some don't see that as something to emulate. Allowing only one alias for network devices is generally not enough ""because people want not a single but multiple names at the same time"", Kay Sievers [said](<https://lwn.net/Articles/448708/>), so `ifalias` is only used by some SNMP utilities. Currently, `udev` maintains a set of links in `/dev/disk/by-*` that relate disks to kernel devices by a variety of characteristics (ID, label, path, and UUID). James Bottomley would [like to see](<https://lwn.net/Articles/448672/>) that be extended for the preferred names: 

All userspace naming will be taken care of by the usual udev rules, so for disks, something like /dev/disk/by-preferred/<fred> which would be the usual symbolic link. 

This will ensure that kernel output and udev input are consistent. It will still require that user space utilities which derive a name for a device will need modifying to print out the preferred name. 

But there are problems inherent in that idea. In order for `udev` to know that the preferred name was set, a uevent would have to be generated. That could be done, but it leads to other problems, as Sievers [points out](<https://lwn.net/Articles/448673/>) (instead of `by-preferred`, he uses `by-pretty`): 

What would happen if we mount:  

```
/dev/disk/by-pretty/foo
```

and some tool later thinks the pretty name should better be 'bar', it writes the name to /sys, we get a uevent, the old link disappears, we get a new link, mount has no device node anymore for the mounted device ... 

Essentially, udev keeps track of the devices present in the system (and their attributes like, potentially, preferred name), but doesn't have any concept of tracking ""no longer valid names"" as Sievers [puts it](<https://lwn.net/Articles/448675/>). That means that `udev` can't just leave older entries around when user space changes the preferred name: ""We can not just add stuff to /dev without a udev database entry, it would never get removed on device unplug and leave a real mess behind."" 

One possible solution for the renaming problem is to only allow one write to `preferred_name`, so that, once established, those aliases couldn't be changed without a reboot. `udev` could set up the proper links, and various tools could use the aliases as needed. That would solve the renaming problem at the cost of some flexibility. In general, no one was really opposed to the idea of some kind of more-mnemonic name for disks, it is more of a question of how to get there. 

Sievers [proposed](<https://lwn.net/Articles/448684/>) adding a way for `udev` to list all of the symlinks that it creates during device discovery. Anyone (or any tool) that needed to associate an alias with a particular disk could use that output to determine the current device being used (based on the UUID for example), then make the substitution as appropriate. That would work, in general, but Bottomley [sees it](<https://lwn.net/Articles/448685/>) as overly complex for users: 

However, even if we assume they choose one of the current names, they still have to do the mapping manually; even if they have all the information, they can't just cut and paste from dmesg say, they have to cut, edit the buffer to put in the preferred name and then paste ... that's just one annoying step too far for most users. I agree that all the output tools within reason can be fixed to do this automatically, but fixing cat say, just so cat /proc/partitions works would never be acceptable upstream. 

The reason for storing this in the kernel is just that it's easier than trying to update all the tools, and it solves 90% of the problem, which makes the solution usable, even if we have to update tools to get to 100%. 

But Sievers and driver core maintainer Kroah-Hartman see it as papering over more substantial issues. Sievers, at least, would like to see text-file-style debug and error message output replaced (or supplemented) with something more structured: 

We need _smart_ userspace with a debug/error message channel from the kernel to userspace that pops out _structured_ data. Userspace needs to index the data, and merge a lot of userspace information into it. 

Adding just another single-name to the kernel just makes the much-too-dumb free-text printk() a bit more readable, but still sounds not like a solution. Pimping up syslog is not the solution to this problem, and it can't be solved in the kernel alone. 

But, from the user's perspective, disks may already have names (with labels on the enclosures themselves for example) and it would be quite convenient for the kernel's messages to reflect them. In the end, Sievers [isn't opposed](<https://lwn.net/Articles/448688/>) to a disk-specific (rather than for all devices) solution, though he thinks it isn't really the right direction to go. Kroah-Hartman [agrees](<https://lwn.net/Articles/448689/>) and is adamant that this change not go into the driver core. Based on that, Nishijima [plans](<https://lwn.net/Articles/448690/>) to redo the patches, moving the name to `struct gendisk`, renaming the field to `alias_name` (rather than "preferred") to better reflect its purpose, and generating a uevent when the name changes. 

But, following the lead of the network `ifalias` will add to the kernel's user-space interface, only this time for disks. While it may solve an immediate problem for administrators, it will also leave behind some legacy code when, or if, a better solution comes around. That's unfortunate, but, since it solves a real problem, and the change is restricted to subsystem whose maintainer (Bottomley) is in favor of it, it may well turn up in the mainline before long. Any change to system error and debug logging along the lines of what Sievers described is certainly quite a ways off, though there have long been calls for structured kernel output. Sometimes it is just easier to make a change like this in one place, rather than trying to identify and fix all of the places outside of the kernel that would need it.
