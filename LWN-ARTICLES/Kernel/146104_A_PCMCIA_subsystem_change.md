---
title: A PCMCIA subsystem change
url: https://lwn.net/Articles/146104/
date: "August 3, 2005"
category: PCMCIA
author: 
---

Russell King recently sent out [a heads-up](<https://lwn.net/Articles/146105/>) regarding a PCMCIA subsystem change which will affect some users. In 2.6.13, if a PCMCIA driver is linked directly into the kernel, its devices will be recognized and bound at boot time. That means that no hotplug events will be generated for those devices. Since many systems use the hotplug subsystem to do things like configuring network interfaces, this change could lead to broken systems. 

There are also concerns about the naming of disk devices; the presence or absence of a PCMCIA device could cause the names of other disks on the system to change from one boot to the next. Dominik Brodowski has posted [a patch](<https://lwn.net/Articles/146106/>) which causes PCMCIA IDE devices to be initialized late in the boot process in an attempt to minimize this problem; he also notes that `udev` is the right way to deal with device naming issues. 

Meanwhile, most users will not be affected because most distributors build their PCMCIA drivers as modules. Devices managed by those drivers will be configured after the system is bootstrapped, and will generate hotplug events as usual.
