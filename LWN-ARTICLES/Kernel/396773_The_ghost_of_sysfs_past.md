---
title: The ghost of sysfs past
url: https://lwn.net/Articles/396773/
date: "July 21, 2010"
category: "Development model-User-space ABI"
author: "By Jonathan Corbet July 21, 2010"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
July 21, 2010

A few years back, it seemed that incompatible sysfs changes created broken systems on a regular basis. Since then, though, things have gotten better, with no reports of broken systems or forced udev upgrades for a while. That improvement is the result of a deliberate effort on the part of the sysfs hackers to stabilize things and to establish best practices for the use of sysfs-exported information. As some linux-next testers are currently finding out, though, the legacy of older sysfs problems has not entirely faded away yet. 

The `CONFIG_SYSFS_DEPRECATED` configuration option exists as one way of mitigating the effects of a major sysfs change. In the early days of sysfs, devices tended to pop up in strange places, including, especially, under `/sys/class`. In order to bring more consistency to the filesystem, the layout was reorganized to move more device information into `/sys/devices`, create the `/sys/block` directory, and more. Needless to say, any such change would be fatal for systems which expected the old layout, so the configuration option was added to restore that old layout when needed. 

In 2010, nobody has shipped a distribution which relies on the old layout for some time. So Greg Kroah-Hartman has posted [a patch to remove the configuration option](<https://lwn.net/Articles/396782/>) and the significant amount of code needed to support it; that patch has also gone into linux-next. Greg notes: ""This is no longer needed by any userspace tools, so it's safe to remove."" 

Except that maybe it's not safe to remove. Andrew Morton quickly [reported](<https://lwn.net/Articles/396783/>) that his Fedora Core 6 box would not boot without this option. Andrew is well known for running archaic distributions just for the purpose of finding this kind of compatibility issue; one might argue that there probably are not that many other FC6 boxes in use, and even fewer which will be wanting to run 2.6.35 kernels. But, as Dave Airlie [noted](<https://lwn.net/Articles/396784/>), RHEL5 boxes will also fail to boot, and there are rather more of those in operation. 

Dave's advice was blunt: ""Live with your mistakes guys, don't try and bury them."" He knows as well as anybody what the cost of living with mistakes is: the graphics ABIs include a few of their own. Mistakes will happen, but, when they become part of the user-space ABI, they can be difficult to get away from. That is why ABI additions tend to come under high levels of scrutiny: once somebody depends on them, they must be supported indefinitely.
