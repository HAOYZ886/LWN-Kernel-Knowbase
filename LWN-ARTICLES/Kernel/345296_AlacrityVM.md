---
title: AlacrityVM
url: https://lwn.net/Articles/345296/
date: "August 5, 2009"
category: AlacrityVM; Virtualization
author: "By Jake Edge August 5, 2009"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
August 5, 2009

While virtualization has been a boon for many users and data centers, it tends to suffer from performance problems, particularly I/O performance. Addressing that problem is the goal of a newly [announced](<http://lwn.net/Articles/345016/>) project, [AlacrityVM](<http://developer.novell.com/wiki/index.php/AlacrityVM>), which has created a hypervisor based on KVM. By shortening the I/O path for guests, AlacrityVM seeks to provide I/O performance near that of "bare metal" hardware. 

The project is in a "pre-alpha" stage, according to the web page, but it is already reporting some fairly impressive results from a proof-of-concept network driver. Both for [throughput](<http://developer.novell.com/wiki/index.php/Image:Netperf-10ge-tcp-throughput.jpg>) and [latency](<http://developer.novell.com/wiki/index.php/Image:Netperf-10ge-udp-latency.jpg>), the AlacrityVM guest performance compared favorably to that of 2.6.28 and 2.6.29-rc8 hosts. It also clearly out-performed the virtio drivers in a KVM guest. 

The major change that allows AlacrityVM to achieve those gains come from a new kernel-based virtual I/O scheme known as [Virtual-Bus](<http://developer.novell.com/wiki/index.php/Virtual-bus>) (or vbus). Currently, KVM guests use emulated devices--[implemented](<http://developer.novell.com/wiki/index.php/Image:Kvm_interactions-emulation.jpg>) in user space by QEMU--in order to handle I/O requests. That leads to multiple kernel-to-user-space transitions for each I/O operation. The idea behind vbus is to allow guests to directly access the host kernel driver, thus reducing the overhead for I/O.

Using vbus, a host administrator can define a virtual bus that contains virtual devices--closely patterned on the Linux device model--which allow access to the underlying kernel driver. The guest accesses the bus through vbus guest drivers and will only be able to use those devices that the administrator explicitly instantiates on that vbus. The vbus interface supports only two "verbs": `call()` for synchronous requests, and `shm()` for asynchronous communication using shared memory. 

A [document [PDF]](<http://developer.novell.com/wiki/images/7/74/Vbus-introduction.pdf>) by AlacrityVM developer Gregory Haskins describes how to configure and use vbus. Vbus provides a sysfs interface that an administrator can use to create container-like objects that will constrain guests so that they can only access those devices specifically configured for their use. That helps alleviate one of the potential problems with guests accessing kernel drivers more-or-less directly: security. 

The vbus web page has a look at the security issues and how they are handled. The main concerns are ensuring that guests cannot use the vbus mechanism to escape their isolation from other guests and processes, as well as making sure that guests cannot cause a denial of service on the host. The bus can only be created and populated on the host side, and each lives in an isolated namespace, which reduces or eliminates the risk of a cross-bus exploit to violate the isolation. In addition, each task can only be associated with one vbus--enforced by putting a vbus reference in the task struct--so that a guest can only see the device ids specified for that bus. 

Care was taken in the vbus implementation to punish guests for any misbehavior, rather than the host. The two areas mentioned are for guests that, maliciously or otherwise, mangle data structures in the shared memory or fail to service their ring buffer. A naive implementation could allow these conditions to cause a denial of service by stalling host OS threads or by creating a condition that might normally be handled by a `BUG_ON()`. Vbus takes steps to ensure that the host to guest path is resistant to stalling, while also aborting guests that write garbage to the ring buffer data structures. 

Haskins has [posted](<http://lwn.net/Articles/345027/>) a series of patches to add the vbus infrastructure, along with a driver for accelerated ethernet. So far, the patches seem to be fairly well-received, though there are not, yet, very many comments. The web page makes it clear that the project's goal is ""to work towards upstream acceptance of the project on a timeline that suits the community"". The flexibility shown in that goal should serve the project well in getting mainline acceptance down the road. 

The project sums up its status and future plans on the web page as well: ""we have a working design which includes the basic hypervisor, linux-guest support, and accelerated networking. We will be expanding this to include other areas of importance, such as accelerated disk-io, IPC, real-time extensions, and accelerated MS Windows guest support."" As one might guess, the web page also has mailing lists for users and developers as well as kernel and user-space git trees available for interested folks. 

AlacrityVM and vbus both look to be interesting projects, that are probably worth investigating as potential virtualization solutions sometime in the future. The performance gains that come with vbus make it likely to be useful to other projects as well.
