---
title: "Scaling the kernel's MAINTAINERS file"
url: https://lwn.net/Articles/730509/
date: "August 10, 2017"
category: "Development model-Maintainers"
author: "By Jonathan Corbet August 10, 2017"
---

By **Jonathan Corbet**  
August 10, 2017

The kernel's development community is large, to the point that it is often far from obvious who a given patch should be sent to. As the community has grown, it has developed mechanisms for tracking that information centered on a text file called `MAINTAINERS`. But now it would appear that this scalability mechanism has scalability problems of its own. 

The [`MAINTAINERS`](<http://elixir.free-electrons.com/linux/v4.12.5/source/MAINTAINERS>) file sits at the top of the kernel's source tree; it is a plain-text file with a number of fields describing how various kernel subsystems are managed. For example, the following entry was consulted for [a recent article](<https://lwn.net/Articles/730217/>) on device-tree issues: 

```
OPEN FIRMWARE AND FLATTENED DEVICE TREE
        M:	Rob Herring <robh+dt@———>
        M:	Frank Rowand <frowand.list@———>
        L:	devicetree@vger.kernel.org
        W:	http://www.devicetree.org/
        T:	git git://git.kernel.org/pub/scm/linux/kernel/git/robh/linux.git
        S:	Maintained
        F:	drivers/of/
        F:	include/linux/of*.h
        F:	scripts/dtc/
        F:	Documentation/ABI/testing/sysfs-firmware-ofw
```

This entry says that the flattened device-tree subsystem has two maintainers (the "`M:`" fields), Rob Herring and Frank Rowand. The relevant mailing list (`L:`) and web site (`W:`) are given, as is the subsystem's Git tree (`T:`). The `F:` lines describe the files and directories that are considered to be a part of this subsystem. The file can be read in a text editor, but it is often consulted by way of the `get_maintainer.pl` script, which reads a patch and outputs a list of addresses to which the patch should be sent. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

This file is over 14,000 lines long in current kernels and contains entries for over 1,700 subsystems in the kernel. Those subsystems are ostensibly listed in alphabetical order but, as Randy Dunlap noted in [a patch](<https://lwn.net/Articles/730511/>) posted in July, many of the entries are not in the right place. His patch sought to improve the situation, but that is where the real trouble started. 

In a community as large as the kernel, changes to subsystems happen on a frequent basis. If the relevant maintainers are paying attention, those changes result in patches to the `MAINTAINERS` file — on the order of 100 patches in each development cycle. Needless to say, a certain number of those patches will conflict with each other, and Linus Torvalds gets to clean up the mess. When Dunlap's patch inevitably resulted in a merge conflict, Torvalds [proclaimed](<https://lwn.net/Articles/730512/>) his frustration with the `MAINTAINERS` file. Perhaps, he asked, there would be some way to split it up? 

His original suggestion was to spread the maintainer information through the kernel source tree, like the kernel configuration file was split up starting in 1.3.36. He later [suggested](<https://lwn.net/Articles/730513/>) that the maintainer information could, in fact, live in the `Kconfig` files with the configuration options, though that idea did not get far. Regardless of the exact scheme used, he [explained](<https://lwn.net/Articles/730514/>), splitting up the maintainer information would make resolving conflicts into a less frustrating task. 

The advantages of a less frustrated Linus Torvalds are evident to most kernel developers, so interest in improving the `MAINTAINERS` file is relatively high. Joe Perches [suggested](<https://lwn.net/Articles/730515/>) splitting the file into a top-level `MAINTAINERS` directory; that would keep the information easy to find and easy for scripts to work with. And that would indeed appear to be way things are headed. 

That leaves the question of how the file should be split. Yet another suggestion from Torvalds was to [split the file by mailing list](<https://lwn.net/Articles/730519/>), so that all entries pointing to the same list would end up in the same file. But Perches came back with [a script](<https://lwn.net/Articles/730520/>) that splits the file by directory hierarchy instead. The result is 54 files in the `MAINTAINERS` directory with names like `arch_arm64`, `drivers_gpu_drm`, or `filesystems`. Any subsystem without enough entries to justify its own file ends up in `MAINTAINERS/MAINTAINERS`. 

Torvalds [tried it out](<https://lwn.net/Articles/730521/>) but chose not to go with the result. He had a few quibbles with how the split is done, but concluded that ""*some* of it looks really nice"". But perhaps the concern that really stopped things was a familiar one: making this change will create merge conflicts with any subsystem tree containing changes to the `MAINTAINERS` file. The most likely solution to that problem is to make the change immediately after the 4.14-rc1 release. Torvalds has not committed to doing so, but it does seem likely that the `MAINTAINERS` split is coming in the relatively near future.
