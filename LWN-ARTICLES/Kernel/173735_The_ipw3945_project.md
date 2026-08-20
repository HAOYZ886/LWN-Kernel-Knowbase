---
title: The ipw3945 project
url: https://lwn.net/Articles/173735/
date: "March 1, 2006"
category: "Device drivers-Wireless networking"
author: 
---

While there are a number of hopeful developments around the support of wireless network cards in Linux, that support remains one of the larger roadblocks for many users. It is thus always a welcome thing when a major manufacturer announces Linux support - and the beginnings of a working driver - for their products. So when Intel recently [announced](<http://lwn.net/Articles/173470/>) a project to support its 3945ABG wireless adapters, there was a certain amount of celebration. There was also come criticism, however, which highlights an ongoing issue with wireless support under Linux. 

The [ipw3945 project](<http://ipw3945.sourceforge.net>) currently has a developer release of the driver, with a stable version expected within a few weeks. This release supports all of the basic features one would expect, with some additional features (quality of service, for example) "not officially supported." It should, in other words, be enough to allow use of the device. 

It would seem that there is little to complain about here. But there is this little paragraph from the announcement: 

In order to meet the requirements of all geographies into which our adapters ship (over 100 countries) we have placed the regulatory enforcement logic into a user space daemon that we provide as a binary under the same license agreement as the microcode. We provide that binary pre-compiled as both a 32-bit and 64-bit application. The daemon utilizes a sysfs interface exposed by the driver in order to communicate with the hardware and configure the required regulatory parameters. 

The requirement for a binary-only blob brought out some concerns from developers who think that the regulatory-agency requirement has been overblown, and that it is not actually necessary to lock down the code in this way. Others disagree, noting that regulations in many parts of the world are quite strict with regard to allowing any user modification of hardware which can transmit. It is probably true that, in order to be able to offer this product in many parts of the world, Intel must lock down much of this logic in binary-only code. 

Given that, however, Intel has chosen an interesting way to go about it. The closed code is not part of the driver itself; it is a daemon which runs entirely in user space. The driver itself is fully free software. So there is no non-free code going into the kernel, which is surely a step in the right direction. 

The regulatory daemon controls the hardware by way of a special file exported through sysfs. The driver then interprets those commands - which enable or disable specific channels, set maximum power values, and so on - and programs the hardware accordingly. A quick look at the (15,000-line) driver source is sufficient to find the code which actually controls the transmitter's parameters. 

So, in other words, this arrangement has not actually locked down much of anything. The daemon comes with the usual "thou shalt not reverse engineer" provisions, but there are people in parts of the world who can safely ignore that requirement. It would seem that little work beyond running the daemon under `strace` would be required. It might also be possible to write a replacement just by studying the driver code, without looking at the Intel-supplied daemon at all. One way or another, it seems likely that a free replacement for the regulatory daemon will come along, sooner or (not much) later.
