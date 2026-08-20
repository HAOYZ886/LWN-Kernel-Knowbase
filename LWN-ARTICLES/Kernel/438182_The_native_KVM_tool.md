---
title: The native KVM tool
url: https://lwn.net/Articles/438182/
date: "April 12, 2011"
category: "KVM; QEMU; Virtualization-KVM"
author: "By Jonathan Corbet April 12, 2011"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
April 12, 2011

The KVM subsystem provides native virtualization support in the Linux kernel. To that end, it provides a virtualized CPU and access to memory, but not a whole lot more; some other software component is needed to provide virtual versions of all the hardware (console, disk drives, network adapters, etc) that a kernel normally expects to find when it boots. With KVM, a version of the [QEMU](<http://wiki.qemu.org/Main_Page>) emulator is normally used to provide that hardware. While QEMU is stable and capable, it is not universally loved; a competitor has just come along that may not displace QEMU, but it may claim some of its limelight. 

Just over one year ago, LWN covered [an extended discussion](<https://lwn.net/Articles/379869/>) about KVM, and, in particular, about the version of QEMU used by KVM. At that time, there were some suggestions that QEMU should be forked and brought into the kernel source tree; the idea was that faster and more responsive development would result. That fork never happened, and the idea seemed to fade away. 

That idea is now back, in a rather different form, with Pekka Enberg's [announcement](<https://lwn.net/Articles/436781/>) of the "native KVM tool." In short, this tool provides a command (called `kvm`) which can substitute for QEMU - as long as nobody cares about most of the features provided by QEMU. The native tool is able to boot a kernel which can talk over a serial console. It lacks graphics support, networking, SMP support, and much more, but it can get to a login prompt when run inside a terminal emulator. 

Why is such a tool interesting? There seem to be a few, not entirely compatible reasons. Replacing QEMU is a nice idea because, as Avi Kivity [noted](<https://lwn.net/Articles/438187/>), ""It's an ugly gooball"". The `kvm` code - being new and with few features - is compact, clean, and easy to work with. Some developers have said that `kvm` makes debugging (especially for early-boot problems) easier, but others doubt that it can ever replace QEMU, with its extensive hardware emulation, in that role. There's also talk of moving `kvm` toward the paravirtualization model in the interest of getting top performance, but there is also resistance to doing anything which would make it unable to run native kernels. 

Developers seem to like the idea of this project, and chances are that it will go somewhere even if it never threatens to push QEMU aside. There are a few complaints about the `kvm` name - QEMU already has a `kvm` command and the name is hard to search for anyway - but no alternative names seem to be in the running as of this writing. Regardless of its name, this project may be worth watching; it's clearly the sort of tool that people want to hack on.
