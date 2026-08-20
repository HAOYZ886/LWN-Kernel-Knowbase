---
title: "Kernel Summit 2006: The ioctl() interface"
url: https://lwn.net/Articles/191653/
date: "July 18, 2006"
category: ioctl
author: 
---

[2006 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2006/>) on LWN.net.   
---  
This year's Kernel Summit schedule dedicated a few slots to "dissenters" - those who wished to make a case for a change in the direction of kernel development. The first of those slots went to Randy Dunlap and Jesse Barnes, who set out to make the case that the much-maligned `ioctl()` interface is not as evil as it is sometimes made out to be. 

The complaints with `ioctl()` have been heard many times. It is an unregulated means by which new system calls can be added to the kernel - it is easy to add large numbers of them, and some developers do. The resulting API is typically not reviewed by anybody, and tends to be different for every device - even when the same functions are being performed. The `ioctl()` interface is hostile to scripts and to tools like `strace`, and it presents painful 32/64-bit compatibility issues. 

Linus added the complaint that there is no way to enumerate `ioctl()` calls - it is not possible for humans or programs to see which calls are supported by a given driver. It was suggested that the internal `ioctl()` interface could be changed so that each driver would have to register a table of the `ioctl()` calls it supports; the kernel could then list them, perform basic argument processing, etc. That approach would solve the enumeration problem and help move some error-prone processing into common code. This technique could also be implemented in conjunction with a new `kioctl()` system call if need be. 

There are some good things about `ioctl()`. It can both read from and write to user space, meaning that an operation can take instructions from an application, perform some task, and pass detailed information back. It works very well for device-specific behavior, and is easy for application programmers to use. 

There are a number of alternatives to `ioctl()`. `/proc` is well established, but, at this point, attempts to add more interfaces to `/proc` don't get very far. Sysfs is good for a number of things - Linus said that he loves it because it's easy to see just what is available. It is hard to use sysfs for complicated, transactional operations, however, while `ioctl()` works well. For some tasks, relayfs or debugfs can be used, but their purpose is exporting data from the kernel. There is always "foofs," the creation of a device-specific virtual filesystem, but that is a heavyweight approach. Kernel developers can also use netlink, add new system calls, implement a shared memory area for control operations, or create a separate device node for out-of-band control data. Alan Cox also mentioned the `setsockopt()` interface as a possible model. 

The session was not to come to any definite conclusions - it was never meant to. There was agreement, however, that there is a need for better guidelines on how to create appropriate interfaces in kernel code. All that's needed now is somebody to write those guidelines. 

  * [Next: The kernel ABI](<https://lwn.net/Articles/191654/>).
